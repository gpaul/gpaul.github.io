---
layout: post
title: "Clean shutdown example"
date: 2014-12-24 10:02:09 +0000
comments: true
categories: 
---

In my last post I said you should shutdown your goroutines.

Let's build a little app that calculates checksums of all the files in a directory tree.

{% highlight go %}
package main

import (
	"crypto/md5"
	"encoding/base64"
	"flag"
	"fmt"
	"io"
	"os"
	"path/filepath"
)

func main() {
	var rootdir string
	flag.StringVar(&rootdir, "dir", ".", "directory to calculate checksums of")
	flag.Parse()

	// expand paths like "." and "./foo" to "/home" and "/home/foo"
	rootdir, err := filepath.Abs(rootdir)
	if err != nil {
		panic(fmt.Errorf("cannot expand '%s' to absolute path: %v", rootdir, err))
	}

	// check that the path exists
	stat, err := os.Stat(rootdir)
	if err != nil {
		panic(fmt.Errorf("cannot stat '%s': %v", rootdir, err))
	}
	if !stat.IsDir() {
		panic(fmt.Errorf("%s is not a directory", rootdir))
	}

	checksums, err := WalkPath(rootdir)
	if err != nil {
		panic(fmt.Errorf("could not calculate checksums: %v", err))
	}
	for _, checksum := range checksums {
		fmt.Println(checksum.String())
	}
}

type checksum struct {
	filepath string
	sum      []byte
}

func (c *checksum) String() string {
	return base64.StdEncoding.EncodeToString(c.sum) + " " + c.filepath
}

func WalkPath(path string) ([]checksum, error) {
	var sums []checksum
	// fn is called for every file
	fn := func(path string, info os.FileInfo, err error) error {
		if info.IsDir() {
			// we don't checksum directories, only files
			return nil
		}

		// open the file
		file, err := os.Open(path)
		if err != nil {
			return err
		}
		defer file.Close()

		// checksum its contents
		hash := md5.New()
		if _, err := io.Copy(hash, file); err != nil {
			return err
		}
		sum := hash.Sum(nil)

		// save the result
		sums = append(sums, checksum{path, sum})
		return nil
	}
	err := filepath.Walk(path, fn)
	if err != nil {
		return nil, err
	}
	return sums, nil
}
{% endhighlight %}

Here's some sample output:

{% highlight bash %}
[gustav@gpaul-dell test]$ go run walk.go 
wVenkDHhxA+FkxgpvF/FUg== /dev/shm/test/bar
KnJaKj3+S4Dgehm7s3BsWw== /dev/shm/test/baz/more
07BzhNET7exJ6qYjitX/AA== /dev/shm/test/foo
KbumPnhXCnP/gJ5IMLsSyw== /dev/shm/test/walk.go
{% endhighlight %}

You'll notice that all the entries are sorted lexicographically. This property is inherited from the os.Walk function which walks paths in that order.

If these files are large and span several disks its is worth calculating the checksums concurrently. I'm going to modify the WalkPath() function only. For every file I now spawn a goroutine to calculate its checksum. The first error to occur 
will be returned to the WalkFunc and the Walk will terminate.

Results are accumulated into a shared struct that is protected by a Mutex. The 'pure CSP' way would be to
have an additional goroutine that accumulates results, but this gets really messy when you try to wait
for all goroutines to terminate, given that some could return errors and others results. Sometimes a good old mutex
is really the simplest solution. UPDATE: see my next post for what is probably a more go-like architecture.

After the walk terminates, whether by error or by natural completion, we wait for all the spawned goroutines to exit by calling c.wg.Wait().

I've introduced the idea of a control structure 'type ctrl struct' to wrap some of the boiler plate around managing
a concurrent job.

{% highlight go %}

// ctrl consolidates the output, the error channel and the termination WaitGroup
// into a single structure for convenience
type ctrl struct {
	// used to accumulate our results
	acc *checksums
	// used to report errors
	errs chan error
	// used to wait for goroutines to exit
	wg *sync.WaitGroup
}

// WalkPath walks a directory tree and returns the
// checksum and filepath of every file in the tree.
func WalkPath(path string) ([]checksum, error) {
	// setup the control structure
	c := ctrl{
		&checksums{},
		make(chan error, 1),
		&sync.WaitGroup{},
	}

	// fn is our os.WalkFunc, it will be called for every file and directory.
	// It starts a goroutine for every file that calculates the file's checksum.
	fn := func(path string, info os.FileInfo, err error) error {
		if info.IsDir() {
			// we don't checksum directories, only files
			return nil
		}
		if err != nil {
			return err
		}
		// have any workers returned errors?
		select {
		case err = <-c.errs:
			// yes, return that error and terminate the walk
			return err
		default:
			// nope, still going strong
		}
		c.wg.Add(1)
		go checksumFile(path, c)
		return nil
	}
	err := filepath.Walk(path, fn)
	c.wg.Wait()
	if err != nil {
		return nil, err
	}
	// check if any of the last few calculations failed
	select {
	case err := <-c.errs:
		// yep, we failed before calculating all checksums
		return nil, err
	default:
		// no errors were reported and c.wg.Wait() ensures that
		// all goroutines have stopped running. This means
		// the entire run was successful!
	}
	return c.acc.checksums(), nil
}

type checksum struct {
	filepath string
	sum      []byte
}

func (c *checksum) String() string {
	return base64.StdEncoding.EncodeToString(c.sum) + " " + c.filepath
}

type checksums struct {
	lk   sync.Mutex
	sums []checksum
}

func (cs *checksums) add(sum checksum) {
	cs.lk.Lock()
	cs.sums = append(cs.sums, sum)
	cs.lk.Unlock()
}

func (cs *checksums) checksums() []checksum {
	return cs.sums
}

func checksumFile(path string, c ctrl) {
	defer c.wg.Done()
	// open the file
	file, err := os.Open(path)
	if err != nil {
		notifyErr(c, err)
		return
	}
	defer file.Close()

	// checksum its contents
	hash := md5.New()
	if _, err := io.Copy(hash, file); err != nil {
		notifyErr(c, err)
		return
	}
	c.acc.add(checksum{path, hash.Sum(nil)})
}

func notifyErr(c ctrl, err error) {
	// Notify the producer of the error.
	// If there's already an error on the channel,
	// don't bother adding another one, just exit.
	select {
	case c.errs <- err:
	default:
	}
}
{% endhighlight %}

I'm not a big fan of unbounded processes and this program will spawn a new goroutine for every file
as quickly as it can walk the directory tree. Throttling the process to a maximum number of concurrent goroutines
is luckily very easy.

{% highlight go %}
// throttle implements a semaphore used to limit the number
// of concurrent calculateChecksum workers.
type throttle chan struct{}

func newThrottle(n int) throttle {
	t := make(throttle, n)
	for ii := 0; ii < n; ii++ {
		t.ready()
	}
	return t
}

func (t throttle) wait()  { <-t }
func (t throttle) ready() { t <- struct{}{} }
{% endhighlight %}

We use the throttle in our code as follows

{% highlight go %}
type ctrl struct {
	...
	// semaphore to throttle the number of concurrent reads
	throttle throttle
}

func WalkPath(...) {
	...
	const numWorkers = 10
	// setup the control structure
	c := ctrl{
		...
		newThrottle(numWorkers),
	}
	...

	fn := func(...) error {
		...
		c.throttle.wait()
		c.wg.Add(1)
		go checksumFile(path, c)
		return nil
	}
}

func checksumFile(path string, c ctrl) {
	defer c.wg.Done()
	defer c.throttle.ready()
	...
}

{% endhighlight %}

Very simple, and I feel much happier with bounded resource usage.

Let's run it again.

{% highlight bash %}
[gustav@gpaul-dell test]$ go run walk.go 
07BzhNET7exJ6qYjitX/AA== /dev/shm/test/foo
7HhE4vOBPKVjHlsV2Alqxg== /dev/shm/test/walk.go
KnJaKj3+S4Dgehm7s3BsWw== /dev/shm/test/baz/more
wVenkDHhxA+FkxgpvF/FUg== /dev/shm/test/bar
{% endhighlight %}

What happended to the sort order? os.Walk is cool enough to walk the directory tree in order and start our goroutines in order, too. However, this doesn't ensure that they finish in order! We need to sort
the results when we finish accumulating them concurrently.

{% highlight go %}
package main

import (
	...
	"sort"
)

...

type checksums struct {
	lk   sync.Mutex
	sums []checksum
}

func (cs *checksums) add(sum checksum) {
	...
}

func (cs *checksums) checksums() []checksum {
	sort.Sort(cs)
	return cs.sums
}

func (cs *checksums) Len() int { return len(cs.sums) }
func (cs *checksums) Less(i, j int) bool {
	return cs.sums[i].filepath < cs.sums[j].filepath
}
func (cs *checksums) Swap(i, j int) {
	cs.sums[i], cs.sums[j] = cs.sums[j], cs.sums[i]
}

{% endhighlight %}

This was a pretty involved example, but hopefully you learnt as much following along as I did writing it. If this example was wrapped in a library then you could happily use it with the knowledge that there are no zombie goroutines hanging around when the call to WalkPath() returns. I think that is crucial for any library code that
wants to spawn goroutines. 

On that note, don't spawn goroutines in library code if it isn't necessary, it clutters
the stack trace when the client application crashes and adds unnecessary scheduling overhead to the client application.
