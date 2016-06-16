---
layout: post
title: "Another clean shutdown example"
date: 2015-01-01 19:47:09 +0000
comments: true
categories: 
---

In my last post I showed you one way to implement a pipeline with clean shutdown.

I didn't like all the selects and channels much so here's another attempt. For context you should read the previous post.

We still terminate when we get the first error, but I now spawn the producer and accumulator functions as goroutines, letting me drop a bunch of selects that was otherwise necessary to synchronize production and accumulation. I think this is probably a better solution as its cleaner (although a bit more refactoring would be done were this production code.)

I also get the feeling that ranging over a channel until it closes is a go idiom that should make you feel like you're in good company.

Coming up with better names would also be on my todo list for pushing this code to production. The 'checksum' struct is much more than that now, maybe checksumResult but that feels a bit long. How I long for a simple t_checksum! :)

There's some logic like stopping on the first error which were implicit and maybe doesn't make sense. Maybe printing 'error' instead of the checksum but continuing would be more robust behaviour. It depends on operational circumstances.

Anyway, enjoy.

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
	"sort"
	"sync"
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

	checksums, err := walkPath(rootdir)
	if err != nil {
		panic(fmt.Errorf("could not calculate checksums: %v", err))
	}
	for _, checksum := range checksums {
		fmt.Println(checksum.String())
	}
}

func walkPath(path string) ([]checksum, error) {
	const numWorkers = 10

	var wg, wg2 sync.WaitGroup
	workCh := make(chan string)
	errCh := make(chan error, 1)
	checksumCh := make(chan checksum)

	// start the producer
	wg.Add(1)
	go func() {
		// doing the waitgroup accounting here keeps the produceFiles
		// function signature clean
		defer wg.Done()
		produceFiles(path, workCh, errCh)
	}()

	// start consumers/workers
	for ii := 0; ii < numWorkers; ii++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			checksummer(workCh, checksumCh, errCh)
		}()
	}

	// start an accumulator to get results or the first error if there is one
	acc := new(checksums)
	var err error

	wg2.Add(1)
	go func() {
		defer wg2.Done()
		for checksum := range checksumCh {
			if checksum.err == nil {
				acc.add(checksum)
				continue
			}
			if err == nil {
				err = checksum.err
			}
		}
	}()

	// Here we'll wait for the producer to finish walking the tree or to hit an error.
	// Either way, it will close the workCh and return. This will cause the workers to exit
	// and finally when only the accumulator is still running wg.Wait() will unblock.
	wg.Wait()

	// all production has stopped, close the checksumCh to unblock the the
	// accumulator and wait for it to return
	close(checksumCh)
	wg2.Wait()

	if err != nil {
		return nil, err
	}
	return acc.checksums(), nil
}

func produceFiles(root string, workCh chan<- string, errCh <-chan error) {
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
		case err = <-errCh:
			// yes, return that error and terminate the walk
			return err
		default:
			// nope, still going strong
		}
		workCh <- path
		return nil
	}
	filepath.Walk(root, fn)
	close(workCh)
}

func checksummer(workCh <-chan string, checksumCh chan<- checksum, errCh chan<- error) {
	for path := range workCh {
		c := checksum{path, nil, nil}

		// open the file
		file, err := os.Open(path)
		if err != nil {
			c.err = err
			checksumError(errCh, checksumCh, c)
			continue
		}

		// checksum its contents
		hash := md5.New()
		if _, err := io.Copy(hash, file); err != nil {
			file.Close()
			c.err = err
			checksumError(errCh, checksumCh, c)
			continue
		}
		file.Close()
		c.sum = hash.Sum(nil)
		checksumCh <- c
	}
}

func checksumError(errCh chan<- error, checksumCh chan<- checksum, sum checksum) {
	// Notify the producer of the error.
	// If there's already an error on the channel,
	// don't bother adding another one, just exit.
	select {
	case errCh <- sum.err:
	default:
	}
	checksumCh <- sum
}

type checksum struct {
	filepath string
	sum      []byte
	err      error
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
