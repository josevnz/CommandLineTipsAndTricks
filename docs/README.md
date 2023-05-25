# A few command line tricks you can learn faster than drinking your early coffee

In this short tutorial I want to share with you a few tricks and tips to deal with some common situations. We will cover this:

* find
* xargs and nproc
* taskset
* numactl
* watch
* inotify-tools

I will present you with a challenge and the tools demonstrating how to solve each problem.

## What you will need for this tutorial?

* A Linux distribution
* Curiosity

## Dealing with directories with very large number of files

You may have encountered this problem once or twice on your lifetime: You tried to do a ```ls``` on a directory with 
a very large number of files and the command throws an 'argument list too long' error:

```shell
josevnz@orangepi5:/data/test_xargs$ ls *
-bash: /usr/bin/ls: Argument list too long
```
The reason is that [POSIX](https://en.wikipedia.org/wiki/POSIX) compatible systems have a limit for the maximum number of bytes you can
pass as an argument:

```shell
[josevnz@dmaf5 Documents]$ getconf ARG_MAX
2097152
```
2 Million bytes, it may be not enough depending on who you ask, but also is a protection against attacks or innocent 
mistakes with bad consequences.

In any case, how you can bypass this limitation? Well, there are many ways to skin this rabbit

### Using Shell built-in

Bash built-in doesn't have the ARG_MAX limitation

```shell
josevnz@orangepi5:/data/test_xargs$ echo *|ls
...
test_file055554  test_file111110  test_file166666  test_file222222  test_file277778  test_file333334  test_file388890  test_file444446
test_file055555  test_file111111  test_file166667  test_file222223  test_file277779  test_file333335  test_file388891  test_file444447
test_file055556  test_file111112  test_file166668  test_file222224  test_file277780  test_file333336  test_file388892  test_file444448
```

This is probably the simplest solution, but let's see another way.

### Using find when you want formatting options

Or you can use this well known `find` flag:

```shell
find /data/test_xargs -type f -ls -printf '%name'
```

Or with _formatting_, to mimic `ls`:
```shell
find /data/test_xargs -type f -printf '%f\n
```

This is fast and also the most complete solution. But before moving on I'll show you yet another way.

### Using xargs

The following works:

```shell
find /data/test_xargs -type f -print0 | xargs -0 ls
```

But it is inefficient, you are forking 3 processes to display the contents of the directory, on top of that xargs
_is throttling_ how many files will be passed to the ls command. 

Let's move on with a different problem.

## Running more programs without crushing the server

### First you walk then you run: Do it serially
So say that you want to compress all the files on the given directory from our previous example. A first try would be like this:

```shell
gzip *
```

Which will take a long time as gzip will process one file at the time.

You _could think_ to do something like this to compress files in parallel:

```shell
josevnz@orangepi5:/data/test_xargs$ for file in $(ls data/test_xargs/*); do gzip $file &; done
-bash: /usr/bin/ls: Argument list too long
```
Again, ARG_MAX strikes again.

We know xargs or find now, so what if we do this:

```shell
for file in $(find $PWD); do echo gzip $file &; done
wait
echo "All files compressed?"
```

That will either make your **server run out of memory or crush it under very heavy CPU utilization** because you are forking a gzip instance for every file found.

### Our first attempt to parallelism and throttling (the art of self control)

What you need is a way to _throttle_ your compression requests, so you don't launch more processes than the number of CPUS you have.

Let's try that again with `find` and `xargs`:

```shell
find /data/test_xargs -type f -print0| xargs -0 -P $(($(nproc)-1)) -I % gzip %
```

Oh. That looks like a fancy one-liner. Let me explain how it works:

1. Use find to get all files on the given directory, use the null character as separator to be able to process weird named ones.
2. `nproc` will tell you how many CPUS you have, then subtract 1 using Bash arithmetic like this using sub-shells: `$(($(nproc)-1))`
3. Finally, xargs will run no more than -P processes (In my case 8 CPUS - 1 = 7 jobs), replacing the '%' with the name of the file to compress

Note: _There are other ways to get the number of CPUS on the machine_, like parsing `/proc/cpuinfo`. There are other more efficient compression out there but gzip is available on pretty much any Linux/ Unix out there.

OK, time to see our next problem.

### CPU Affinity with taskset to maximize execution time

Despite limiting the number of CPUS, some intensive jobs can slow down other processes on your machine when looking for resources, there are a few things you can do to keep the performance of the server under control like using [taskset](https://github.com/util-linux/util-linux/blob/master/schedutils/taskset.c):

> The taskset command is used to set or retrieve the CPU affinity
       of a running process given its pid, or to launch a new command
       with a given CPU affinity. CPU affinity is a scheduler property
       that "bonds" a process to a given set of CPUs on the system.


In general, we always want to leave one of the CPUS 'free' for operating system tasks. The Kernel is normally pretty good keeping running processes glued to a specific CPU to avoid context switching, but if you want to enforce on which CPUS your process will run you can use `tasket`

```shell
taskset -c 1,2,3,4,5,6,7 find /data/test_xargs -type f -print0| xargs -0 -P $(($(nproc)-1)) -I % gzip %
```

### taskset the only game in town? not so _numactl_ fast!

What is [NUMA and why you should care](https://documentation.suse.com/sles/12-SP4/html/SLES-all/cha-tuning-numactl.html)?

> There are physical limitations to hardware that are encountered when many CPUs and lots of memory are required. 
> The important limitation is that there is limited communication bandwidth between the CPUs and the memory. 
> One architecture modification that was introduced to address this is Non-Uniform Memory Access (NUMA).

So most simple desktop machines only have a single NUMA node, like mine:

```shell
[josevnz@dmaf5 ~]$ numactl --hardware
available: 1 nodes (0)
node 0 cpus: 0 1 2 3 4 5 6 7
node 0 size: 15679 MB
node 0 free: 5083 MB
node distances:
node   0 
  0:  10
# Or with lscpu
[josevnz@dmaf5 ~]$ lscpu |rg NUMA
NUMA node(s):                    1
NUMA node0 CPU(s):               0-7
```

If you have more than one NUMA node, you may want to 'pin' or set the affinity of your program to use the CPUS and memory of the same node.

For example, on a machine with 16 cores, 0-7 on node 0, 8-15 on node 1, we could force our compression program to run on all the CPUS on node 1, and use the memory of node 1 like this:

```shell
numactl --physcpubind 8-15 --membind=1 find /data/test_xargs -type f -print0| xargs -0 -P $(($(nproc)-1)) -I % gzip %
```

## Keeping an eye on things

### Just watch what I do

The [watch](https://www.man7.org/linux/man-pages/man1/watch.1.html) command allows you to periodically run a command, and even show you the differences before calls:

```shell
Every 10.0s: ls                                                                                                         orangepi5: Wed May 24 22:46:33 2023

test_file000001.gz
test_file000002.gz
test_file000003.gz
test_file000004.gz
test_file000005.gz
test_file000006.gz
test_file000007.gz
test_file000008.gz
test_file000009.gz
test_file000010.gz
...
```

Shows me the output of the `ls` command every 10 seconds. To detect changes on a directory this is simple, but not easy to automate and definitely not efficient.

Wouldn't be nice if the kernel was able to tall me about changes on my directories?

### A better way to know about changes on the filesystem, with inotify-tools

You may need to install this separately, but it should be easy to do. On Ubuntu:

```shell
sudo apt-get install inotify-tools
```

On Fedora:

```shell
sudo dnf install -y inotify-tools
```

So how we can monitor for events on a given directory?

On one terminal we can run inotifywait:

```shell
josevnz@orangepi5:/data/test_xargs$ inotifywait --recursive /data/test_xargs/
Setting up watches.  Beware: since -r was given, this may take a while!
Watches established.
```

And on another terminal we can touch some files to simulate an event:

```shell
josevnz@orangepi5:/data/test_xargs$ pwd
/data/test_xargs
josevnz@orangepi5:/data/test_xargs$ touch test_file285707.gz test_file357136.gz test_file428565.gz
```

The original terminal will get the first event and exit:

```shell
Watches established.
/data/test_xargs/ OPEN test_file285707.gz
```

To make it listen for even forever we do this:

```shell
josevnz@orangepi5:/data/test_xargs$ inotifywait --recursive --monitor /data/test_xargs/
```

If we touch the file again on a separate terminal then this time we will see all the events:

```shell
Setting up watches.  Beware: since -r was given, this may take a while!
Watches established.
/data/test_xargs/ OPEN test_file285707.gz
/data/test_xargs/ ATTRIB test_file285707.gz
/data/test_xargs/ CLOSE_WRITE,CLOSE test_file285707.gz
/data/test_xargs/ OPEN test_file357136.gz
/data/test_xargs/ ATTRIB test_file357136.gz
/data/test_xargs/ CLOSE_WRITE,CLOSE test_file357136.gz
/data/test_xargs/ OPEN test_file428565.gz
/data/test_xargs/ ATTRIB test_file428565.gz
/data/test_xargs/ CLOSE_WRITE,CLOSE test_file428565.gz
```

This is less taxing to the operating system than asking for directory changes every time, and filtering just the differences ourselves.


## What is next

There is _so much more to explore_. The tips above introduced you to some important concepts, so why not to learn much more about them?

* The [Ubuntu forum](https://askubuntu.com/questions/217764/argument-list-too-long-when-copying-files) has a great conversation about _xargs_, _find_, _ulimit_ and other things. Knowledge is power.
* RedHat as [a nice page](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/monitoring_and_managing_system_status_and_performance/configuring-an-operating-system-to-optimize-cpu-utilization_monitoring-and-managing-system-status-and-performance) about NUMA, taskset, interrupt handling. If you are serious about fine-tunning the performance of your processes, please read it.
* You liked [inotify](https://en.wikipedia.org/wiki/Inotify) and want to use it from your Python script. Then take a look at [pynotify](https://github.com/seb-m/pyinotify/wiki).
* Find may be intimidating, but [this tutorial](https://www.digitalocean.com/community/tutorials/how-to-use-find-and-locate-to-search-for-files-on-linux) will make it easier to understand.
* Source code for this tutorial can be found [here](https://github.com/josevnz/CommandLineTipsAndTricks).