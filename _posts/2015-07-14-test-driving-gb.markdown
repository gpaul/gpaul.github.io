---
layout: post
title: "test-driving gb"
date: 2015-07-14 10:49:00 +0000
comments: true
categories: 
---

I'm going to give gb a try. Dave Cheney is a cool guy and I've been hearing a lot about it.

I've learned a shitload by watching colleagues work, seeing how they think, learning 
about new tools and generally seeing my local maxima for what they were. (perl -ne is awesome.) 

As such I'm going to experiment with a 'watch over my shoulder' style for this post.

## Installation

The [installation docs](http://getgb.io/docs/install/) are pretty simple. Let's hit it.

{% highlight bash %}
$ mkdir ~/test-gb
$ cd !$
cd ~/test-gb
$ export GOPATH=`pwd`
$ go get github.com/constabulary/gb/...
$
{% endhighlight %}

I'm a fan of verbose output. Let's do that properly

{% highlight bash %}
$ rm -rf bin pkg src
$ go get -v github.com/constabulary/gb/...
github.com/constabulary/gb (download)
github.com/constabulary/gb
github.com/constabulary/gb/cmd
github.com/constabulary/gb/cmd/gb-vendor/vendor
github.com/constabulary/gb/cmd/gb
github.com/constabulary/gb/cmd/gb-vendor
{% endhighlight %}

I find that output much more satisfying.

I imagine that command dropped 'gb' in the bin directory, let's check:

{% highlight bash %}
$ ls bin
gb  gb-vendor
{% endhighlight %}

I was expecting 'gb', but 'gb-vendor' ... that's suspicious. Maybe the help text
proves insightful.

{% highlight bash %}
$ ./bin/gb-vendor -help
FATAL could not locate project root: project root is blank
{% endhighlight %}

The fact that we see a FATAL log instead of useful help text let's me think gb-vendor doesn't take any command line flags. It also means this isn't a case of simply missing a config file, as you'd expect something like config file location to be a flag. This leaves one option: environment variables. This is a fairly obscure way to pass configuration to a binary when said binary doesn't have any usage text - unless this binary is not intended to be run manually.

All this from running a Go binary with -help. I think [we could do with better usage text](https://github.com/constabulary/gb/pull/265). Be right back...

...back. It turns out that the usage text for this binary is actually pretty solid - gb-vendor just isn't intended to be run by hand. Instead, it is a plugin intended to be run by the gb command when you run 'gb vendor'. 

The specific problem is that it checks an environment variable 'GB_PROJECT_DIR' first thing. When that's blank, we're met with the FATAL log. I patched the main function to print help text first. 

Now we're met with the following:

{% highlight bash %}
$ ./bin/gb-vendor
gb-vendor, a gb plugin to manage your vendored dependencies.

Usage:

        gb vendor command [arguments]

The commands are:

        fetch       fetch a remote dependency
        update      update a local dependency
        list        lists dependencies, one per line
        delete      deletes a local dependency
        purge       purges all unreferenced dependencies

Use "gb vendor help [command]" for more information about a command.

Additional help topics:


Use "gb vendor help [topic]" for more information about that topic.
{% endhighlight %}

That's way better. For one, it's now clear that this binary really ought to
be run as 'gb vendor' - gb will run the gb-vendor binary for us. This is why the
assumption that the environment variable is present isn't that strange - it's actually totally sensible. 

## My first gb command

The [getgb.io](http://getgb.io) website says 'gb' is 
"a project based build tool for the Go programming language." It's a replacement
for the standard 'go' tool. I work on a project of several hundred thousand lines of go code, with tens of external repositories pulled into our repo as git subtrees and it works very well. As such I'm in no hurry to drop the go tool, but the point of this post is to have a look at gb, so let's go.

Let's try to build our GOPATH.

{% highlight bash %}
$ gb
gb, a project based build tool for the Go programming language.

Usage:

        gb command [arguments]

The commands are:

        build       build a package
        depset      
        doc         show documentation for a package or symbol
        env         print project environment variables
        generate    generate Go files by processing source
        list        list the packages named by the importpaths
        plugin      run a plugin
        test        test packages

Use "gb help [command]" for more information about a command.

Additional help topics:

        project     gb project layout

Use "gb help [topic]" for more information about that topic.
{% endhighlight %}

Of those, _build_ and _env_ look pretty interesting.

{% highlight bash %}
$ ./bin/gb env
GB_PROJECT_DIR="/home/gustav/test-gb"
{% endhighlight %}

Ha! That's exactly the environment variable the gb-vendor was missing. Makes sense.

Let's try _build_.

{% highlight bash %}
$ ./bin/gb build
github.com/constabulary/gb
github.com/constabulary/gb/cmd
github.com/constabulary/gb/cmd/gb-vendor/vendor
github.com/constabulary/gb/cmd/gb
github.com/constabulary/gb/cmd/gb-vendor
{% endhighlight %}

Pretty sweet. The go tool's build is idempotent, let's see about gb's.

{% highlight bash %}
$ ./bin/gb build
github.com/constabulary/gb/cmd/gb
github.com/constabulary/gb/cmd/gb-vendor
$ ./bin/gb build
github.com/constabulary/gb/cmd/gb
github.com/constabulary/gb/cmd/gb-vendor
{% endhighlight %}

From the output I'm guessing it rebuilds the binaries. Let's check that

{% highlight bash %}
$ strace -f -q -e open ./bin/gb build
...
[pid  8425] open("/home/gustav/test-gb/src/github.com/constabulary/gb/cmd/resolve_test.go", O_RDONLY|O_CLOEXEC) = 3
[pid  8425] open("/home/gustav/test-gb/src/github.com/constabulary/gb/cmd/test.go", O_RDONLY|O_CLOEXEC) = 3
[pid  8425] open("/home/gustav/test-gb/src/github.com/constabulary/gb/cmd/test_test.go", O_RDONLY|O_CLOEXEC) = 3
[pid  8425] open("/home/gustav/test-gb/src/github.com/constabulary/gb/cmd/testflag.go", O_RDONLY|O_CLOEXEC) = 3
...
github.com/constabulary/gb/cmd/gb
github.com/constabulary/gb/cmd/gb-vendor
...
[pid  8432] open("/home/gustav/test-gb/bin/gb-vendor", O_WRONLY|O_CREAT|O_TRUNC, 0775) = 3
[pid  8432] open("/dev/shm/gb185198689/github.com/constabulary/gb/cmd/gb-vendor/main.a", O_RDONLY) = 4
[pid  8432] open("/home/gustav/go/pkg/linux_amd64/runtime.a", O_RDONLY) = 4
...
[pid  8433] open("/home/gustav/test-gb/bin/gb", O_WRONLY|O_CREAT|O_TRUNC, 0775) = 3
[pid  8433] open("/dev/shm/gb185198689/github.com/constabulary/gb/cmd/gb/main.a", O_RDONLY) = 4
[pid  8433] open("/home/gustav/go/pkg/linux_amd64/runtime.a", O_RDONLY) = 4
[pid  8432] open("/home/gustav/go/pkg/linux_amd64/flag.a", O_RDONLY) = 4
...
{% endhighlight %}

It seems like it reads the source files in your project root and decides what to rebuild. In our case all the archive files were up to date. It then relinks the binaries (O_CREAT|O_TRUNC) means it truncates the existing binaries if they exist and creates them if they don't.

You should really consider using strace for everything. I use it very regularly and I love it. It shows you everything a program does except memory access (it shows mallocs!) and computation - which is often more than enough to see where problems lie. It's also really good for reading consolidated log output from processes that have the annoying habit of logging to multiple files - or when that process executes in a separate mount namespace and dumps its logs in its data directory (I'm looking at you leveldb.)

## A new project

I want to start a new project. I'll leave gb where it is and add the bin directory to my PATH.

{% highlight bash %}
$ echo PATH=$PATH:/home/gustav/test-gb/bin >> ~/.bash_profile
$ source !$
source ~/.bash_profile
$ which gb
~/test-gb/bin/gb
{% endhighlight %}

Excellent.

{% highlight bash %}
$ mkdir ~/gb-first-project
$ cd !$
cd ~/gb-first-project
$ mkdir -p src/myproject/cmd/project
$ cat <<EOF > src/myproject/cmd/project/main.go
package main

import "fmt"

func main() {
	fmt.Println("first gb project!")
}
$ gb build
myproject/cmd/project
$ ./bin/project
first gb project!
{% endhighlight %}

That was very straight-forward. Exactly the same flow as the go tool thus far. If you were expecting something spectacular you're missing the point. I don't want my build tools to be spectacular, I want them to do as little
as possible without making me do a bunch of stuff myself. I expect gb to come into its own only once
you start to add third-party dependencies. Before then it shouldn't surprise you.

## Adding a third-party dependency with gb

Let's add [gogoprotobuf](https://github.com/gogo/protobuf) as a dependency. It's
always just a matter of time before I add it to any new project anyway, let's
add it while we're still fresh. I use it for everything. I smile every time I merge it into a new project. 
The author is also a good friend of mine and if you knew how seriously he takes software testing and code quality you would import his code, too.

The gogoprotobuf README says to install it by running

{% highlight bash %}
go get github.com/gogo/protobuf/proto
go get github.com/gogo/protobuf/protoc-gen-gogo
go get github.com/gogo/protobuf/gogoproto
{% endhighlight %}

I'm going to use the 'gb vendor' command I patched earlier. We'll use 'gb fetch' instead of 'go get'.

{% highlight bash %}
$ gb vendor fetch github.com/gogo/protobuf/proto
fetching recursive dependency github.com/gogo/protobuf/proto/testdata
$ gb vendor fetch github.com/gogo/protobuf/protoc-gen-gogo
fetching recursive dependency github.com/gogo/protobuf/gogoproto
fetching recursive dependency github.com/gogo/protobuf/plugin/defaultcheck
fetching recursive dependency github.com/gogo/protobuf/plugin/description
fetching recursive dependency github.com/gogo/protobuf/plugin/embedcheck
fetching recursive dependency github.com/gogo/protobuf/plugin/enumstringer
fetching recursive dependency github.com/gogo/protobuf/plugin/equal
fetching recursive dependency github.com/gogo/protobuf/plugin/face
fetching recursive dependency github.com/gogo/protobuf/plugin/gostring
fetching recursive dependency github.com/gogo/protobuf/plugin/marshalto
fetching recursive dependency github.com/gogo/protobuf/plugin/populate
fetching recursive dependency github.com/gogo/protobuf/plugin/size
fetching recursive dependency github.com/gogo/protobuf/plugin/stringer
fetching recursive dependency github.com/gogo/protobuf/plugin/testgen
fetching recursive dependency github.com/gogo/protobuf/plugin/union
fetching recursive dependency github.com/gogo/protobuf/plugin/unmarshal
$ gb vendor fetch github.com/gogo/protobuf/gogoproto
FATAL command "fetch" failed: github.com/gogo/protobuf/gogoproto is already vendored
FATAL command "vendor" failed: exit status 1
{% endhighlight %}

So far so good. Its interesting to note that 'gb vendor' sees fetching of an already vendored package as a fatal error. There's a separate cmd 'gb vendor update' to update it.

Let's check our source tree.

{% highlight bash %}
$ find . -maxdepth 1 -type d 
.
./src
./vendor
./pkg
./bin
$ find src/ -maxdepth 3 -type d
src/
src/myproject
src/myproject/cmd
src/myproject/cmd/project
$ find vendor -maxdepth 4 -type d 
vendor
vendor/src
vendor/src/github.com
vendor/src/github.com/gogo
vendor/src/github.com/gogo/protobuf
{% endhighlight %}

So 'gb vendor' didn't touch our src directory. Instead it created a vendor/src directory and pulled the gogoprotobuf sources in there.

Right now we don't use any gogoprotobuf code in our main.go file. Let's do a contrived import so we're sure it is being pulled into the binary.

{% highlight bash %}
$ cat <<EOF > src/myproject/cmd/project/main.go
package main

import (
	"fmt"
	"github.com/gogo/protobuf/proto"
)

func main() {
	dummy := "first gb project!"
	fmt.Printf("%s %p\n", dummy, proto.String(dummy))
}
{% endhighlight %}

Let's rebuild the binary

{% highlight bash %}
$ gb build
myproject/cmd/project
$ ./bin/project
first gb project! 0xc20800a360
{% endhighlight %}

Cool, that does what I thought it does. Let's recheck our src directory.

{% highlight bash %}
$ find src/ -maxdepth 3 -type d
src/
src/myproject
src/myproject/cmd
src/myproject/cmd/project
$ find vendor -maxdepth 4 -type d 
vendor
vendor/src
vendor/src/github.com
vendor/src/github.com/gogo
vendor/src/github.com/gogo/protobuf
{% endhighlight %}

It didn't touch our src directory. That's ... pretty damn cool. I'm so used to
having my src directory cluttered with shit I didn't write that I've forgotten
what it feels like to browse ./src and see only my code. It takes me back to my ruby days where third party code is shipped as opaque 'gems'. Dealing with third-party dependencies on the source level has definitely made me a better programmer and I would find it difficult to go back to a world where I simply trust the library maintainer not to screw up. 

Some early thoughts on gb follow. I don't think you can really like or disqualify tooling on a 10 line program - you need a few thousand lines at least, but I've already seen some cool stuff.

It seems like 'gb' gives me the best of both worlds: I have all the sources I'm depending on, but it is away from the code I wrote, the code I generally want to grep and sed through. What I enjoy quite a lot about gb's approach is that I can start out using it and if I don't like it six months down the line I can just fold the gb vendor/src directory into my regular src directory and I'm back in go tool land.

I'm about to switch back to my day job and a 500k+ loc go tool-based project but I'm a little sad to leave my clean src directory behind. I like gb's plugin scheme which allows something as integral as the vendor command to be implemented externally. I think go's authors are all about bringing new team members on board and have them hit the ground running. For that you want to strongly emphasize convention over configuration - in fact you want no configuration. Plugins allow different teams to slowly drift apart in their processes. If you're a small team and people aren't moving between teams in your company I think doing some manual tooling in the form of build system plugins is a fine idea. 

I'm a fan of git's update and post-receive hooks for checking that code is fmt'ed, imports are clean, and generated code hasn't changed by accident. These all count as per-project configuration that doesn't translate to other teams. It is also extremely useful and have prevented many bugs.

One thing I don't envy about maintaining gb is that people will expect it to track the go tool in functionality and Dave says he explicitly doesn't want to just wrap the go tool. I look forward to seeing where gb goes, especially now that the go tool also supports explicit vendoring of sources.
