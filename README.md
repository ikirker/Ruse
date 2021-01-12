# Ruse

Ruse is a command-line utility that periodically measures the resource use of a process and its subprocesses. It is intended to help you find out how much resource to allocate to your jobs on a remote cluster or supercomputer. With Ruse you can find the actual memory, execution time and cores that you need for individual programs or MPI processes, presented in a way that is straightforward to understand and apply to your [Slurm] job submissions.

Ruse periodically samples the process and its subprocesses and keeps track of the CPU, time and maximum memory use. It also optionally records the sampled values over time. The purpose or Ruse is not to profile processes in detail, but to follow jobs that run for many minutes, hours or days, with no performance impact and without changing the measured application in any way.

## Usage

To profile a command, run

```
ruse [FLAGS] command [ARG...]
```

Options are:

```
  -l, --label=LABEL      Prefix output with LABEL (default 'command')
      --stdout           Don't save to a file, but to stdout
      --no-header        Don't print a header line
      --no-summary       Don't print summary info
  -s, --steps            Print each sample step
  -p, --procs            Print process information (default)
      --no-procs         Don't print process information
  -t, --time=SECONDS     Sample every SECONDS (default 30)

      --rss              use RSS for memory estimation
      --pss              use PSS for memory estimation (default)

      --help             Print help
      --version          Display version
```

### Examples of use

Let's begin with a toy example.

**Measure the "sleep" command:**

```
ruse sleep 150
```

Ruse samples the CPU and memory use every 30 seconds and writes a file `sleep-<pid>.ruse` with the contents:

```
Time:           00:02:30
Mem:            0.6 MB
Cores:          4
Procs:          1
Total_procs:    1
Active_procs:   0
Proc(%): 
```

We see that "sleep" ran for 2 minutes and 30 seconds; it used about 0.6 MB memory; it had 4 cores available, but spawned only 1 process, and no process was ever active. Not surprising, as "sleep" does nothing at all very efficiently.

**Measure a computational chemistry calculation**

Here we'll run an example calculation on a molecule using a computational chemistry package. We run the command as:

```
$ ruse -st1 g09<Azulene.inp >azu.out
```

This tells Ruse to record each time step with the "-s" option, and to use a one-second time step with "-t1". Our result file looks like this (a bit abbreviated):

```
   time         mem   processes  process usage
  (secs)        (MB)  tot   act  (sorted, %CPU)
      0        13.7     1     0 
      1       273.4     9     8   34   7   7   6   6   6   4   4
      2       284.7     9     8   87  87  87  84  83  83  83  82
      3       301.4     9     8   99  99  99  98  98  98  98  90
      4       345.4     9     8   67  53  53  53  52  52  51  50
      5        65.9     2     1   31
      6       289.8     9     8   98  98  98  98  97  97  91  83
      7        14.7     2     1   75
      8        15.1     2     1   58
      9       280.7     9     8   62  44  43  37  37  33  19  19
     10       287.4     9     8   99  99  99  99  95  94  70  69
     ...
     32       397.8     9     8  100 100 100 100 100 100  99  98
     33       448.5     9     8   98  98  98  98  98  98  96  93
     34       209.1     9     8   79  14  14  14  14   7   6   5
     35       291.5     9     8   72  51  51  50  48  48  47  45
     36       291.5     9     8  100  99  99  99  99  99  99  98
     37       331.2     9     8   99  98  98  98  98  98  98  97
     38       168.7     9     8   90   5   5   5   2   2   2   2

Time:           00:00:39
Memory:         448.5 MB
Cores:          8
Total_procs:    9
Active_procs:   8
Proc(%): 74.8  50.9  50.8  50.3  49.6  49.0  45.5  44.0  
```

We have 8 available cores; 9 processes in total; and at most 8 processes are active at any one time. This is a common pattern: A parent process reads input files and sets everything up; then a set of worker threads do the actual calculations while the parent process is idle.

This application runs calculations in phases, with some phases using all available cores, and others using only a single core at a time. This pattern limits the effective speed you can get from using multiple cores. Memory use, too, varies substantially during different phases of calculation.

The CPU use is *not* ordered by process. The processes may be completely different from one step to the next. Each step displays the CPU use of the active processes or threads sorted from most active to the least. The final average is the sum of each time step, divided by the number of steps. 

If the number of cores is at least equal to the number of active processes, the CPU use will be the same as usage per core. This doesn't tell you about the performance of specific processes, but it does tell us how efficiently we use the available cores overall, and that's what we care about with Ruse.


## Build

Ruse uses the GNU Autotools. In most cases you can run "configure", then "make" and "make install" to build and install Ruse.  You will need the Linux software development tools (a C compiler such as GCC, library headers, the GNU Autotools and so on), but it needs no external dependencies.

Create a build directory. In that directory do 'configure', 'make' and 'make install:

```
$ mkdir build
$ cd build
$ ../configure
$ make && make install
```

Do `configure --help` to see the available options. In particular, you can use `configure --prefix=<some_path>` to install Ruse into the given path instead of system-wide.

If you cloned the git repository and need to recreate the build files, you can run `./autogen.sh` in the top directory to do so. Then you can do `configure`, `make` and `make install` as above.

## Options

*  -l LABEL, --label=LABEL  
  
  Ruse outputs a file named `<name of program>-<PID>.ruse` by default. If you don't want to use the name of the program, you can set a different name here.


* --stdout           
  
  Print the output on screen instead of saving into a file. Good for debugging and testing.


* --no-header        

  Don't print the header lines at the top of the file when printing each time step. This is good for when you want to read the output with a different program.


* --no-summary       

  Don't print the summary info at the end. This is useful when you just want to read the stepwise data with another program.

  
* -s, --steps  

  Print each sample step. Each line has the form:

  ``` Time(seconds)  Memory(MB)  total-processes  active-processes  sorted list of CPU use```

  Each value is separated by white space. The sorted list has as many elements as the number of active processes. If process information is turned off, this will print only time and memory.


* -p, --procs            

  Print process information. This finds the number of available cores; the total number of processes, the number of active processes and the CPU usage, in percent. 

  * Total processes is the number of child processes and threads the application has at the time of taking the sample. 

  * Active processes are child processes and threads that have a non-zero CPU use during the previous sample period.
  
  * The process list is a list of active processes' CPU usage, in percent, sorted from highest to lowest. 

  If the number of cores his higher than the number of active processes, the excess cores go unused. This generally means you have too many cores allocated.

  If the number of cores is lower, then some of those active processes are sharing a core between them. For some jobs that are IO bound this is fine, and an efficient use of resources. In other cases the job could benefit from more cores.


* --no-procs         

  Don't print process information. We still calculate it in the background; it is very efficient and there is little to gain by actively disabling it.


* -t SECONDS, --time=SECONDS

  Sample process and memory use every SECONDS. In general, a shorter interval may let you catch some transient events, or to measure a short-running application. But it comes at the potential cost of higher overhead and of much longer result files. If you are only collecting the summary data there is little reason to change this value.


* --rss              use RSS for memory estimation 
  --pss              use PSS for memory estimation 

  RSS and PSS  are two ways to measure the amount of memory used by a process. "RSS" (Resident Set Size) counts the amount of physical memory used by a process. It doesn't take shared memory into account, making the estimation pessimistic. PSS (Proportional Set Size) accounts for shared memory areas. But the estimation is slow; for a large process (hundreds of GB) it can take a lot of time.

  Ruse defaults to RSS due to the performance impact. In most cases the actual difference is fairly small. The estimation is meant to be a lower bound for job allocation, so being pessimistic is not really a problem. For clusters where [Slurm] uses RSS for tracking memory limits this is the best estimate. 

  You can enable PSS with the `--pss` option. You can also default to PSS at build time by passing `--enable-pss` to the configure invocation. Do note that this can have a significant performance impact; avoid using short time periods if you do this.


* --help, --version

  Display a short help text with the options, and show the version of Ruse.


## FAQ

#### Why is the sampling rate so low? 30 seconds is really long.

Ruse is not meant to profile quick programs on your local computer. It's meant to help allocate resources on compute clusters, where an individual job often takes hours, days or weeks to finish. 

When you run a job on a cluster you typically have to specify the resources — the number of nodes and cores, the amount of memory and the amount of time — you will need ahead of time. You neeed to know the peak memory usage, the number of cores you can make use of, and the total time so you can ask for a sensible amount of these resources.

Ruse uses 30 seconds by default because [Slurm] — the most common job scheduler — samples the resources used by jobs at that rate. Also, few applications will allocate and deallocate memory or threads so fast that we miss anything significant with a 30-second sample rate.

The lower sample limit is 1 second. With a faster rate, Ruse itself would start to take a non-trivial amount of CPU resources. If you need finer granularity than that, we suggest it is time for you to break out a real profiler.

#### Doesn't Slurm already tell you how much resources you use?

[Slurm] can be configured to show you the memory used. But the data is not simple to interpret unless you understand how [Slurm] works. This goes especially if your job has multiple subjobs, or uses MPI. 

[Slurm] will not report the memory use of individual commands, and will not give you a memory profile over time. [Slurm] will also not give you any information on the number of processes or their CPU usage. With modern high-core systems core allocation is becoming important to get right.

Ruse attempts to report the data in a way that is easy to interpret and use to create a reasonable job submission. We also aim for Ruse to be lightweight enough that you can use it on production jobs without any performance penalty.


#### How does Ruse measure the memory use?

By default, Ruse measures the RSS (Resident Set Size) each sample step. This is a simple and fast measurement that counts the amount of memory that is allocated and in use for the application. 

This is often an over-estimation. If multiple processes all want access to the same data, Linux will only store it in memory once, then let the processes share it. A very common case is shared libraries. 

For example, almost all processes in a Linux system will link to a library called "libc". This is about 2MB in size, and a Linux system easily has hundreds of processes running at any one time. If they all had their own copy in memory, you would need several hundred megabytes just for this one library. Instead, we have only a single (read-only) copy, and all processes in the system share that. 

RSS counts those 2MB of libc fully for each and every application. PSS (Proprtional Set Size) is an alternative that takes this into account. Linux keeps track of the number of processes that have allocated a given memory area, and can calculate the proportion of that memory "belonging to" any given process. If five processes all use a shared memory area, they would each be responsible for 1/5 of the total.

To get the PSS for a process, you (or the kernel) have to go through each and every allocated memory area, look up how many processes have allocated this, then calculate the proportion owed by this process. A large application has thousands of separate memory areas, and in extreme cases estimating this calue can take seconds.

Also, in practice this difference often doesn't matter much. For scientific applications, especially those that are memory hungry, shared data such libraries is only a small fraction of the memory needed. The vast bulk of allocted memory is used for input or working data structures that are unique to the running process. That memory isn't shared and doesn't differ between RSS and PSS.

We default to using RSS for these reasons. It is far more efficient; it will tend to be pessimistic, so you won't go wrong using it as basis for your memory allocations; and in practice the difference is usually not large enough to worry about. 


#### What's the deal with the process measurement?

We measure the amount of CPU used for each active process at each time step, including child processes and threads. We sort this in descending order. This is added to a running total. At the end, the totals are divided by the number of time steps to give you an average per-active process CPU use.

There are two things to keep in mind here:

* We don't care about the identity of these processes. The first process CPU value at each time step is the value for whatever process happened to use the most CPU during that time step. The first process final average is the average of the most active process at each time step, whichever processes that happened to be each time. 

* We don't track *core* usage against processes at all. Due to issues such as core migration it is not possible to get it reliably correct with a sampling approach such as this. You would need to use more intrusive profiling methods for that.

In practice, unless you are overallocating processes, the process use will correspond to core usage, so this tells you most of what you need to know. The percentages tell you how much CPU was used over time, and the number of simultaneous processes tell you the upper limit of the number of cores you could conceivably need.

Let's say you had a total that looks like this:

```
Cores:          4
Total_procs:    4
Avctive_procs:  4
Proc(%): 90.0 65.0 40.0 33.3 
```

Your used processes equal the number of cores, and they are in moderate use. You are making use of the cores you have allocated. It also looks like the third and fourth process is not contributing a huge amount, so increasing the number of cores is not that likely to help you a lot.

If you have something like this:

```
Cores:          4
Total_procs:    2
Active_procs:   1
Proc(%): 75.0 
```

You have multiple cores but only one active process. This tells you your application is not multithreaded. It can use only one core at a time, and allocating more is a waste of reasources. If it's supposed to to be able to use multiple cores, it's time to check the documentation or search online for how to enable that.

Finally, with something like this:

```
Cores:          16
Total_procs:    16
Active_procs:   16
Proc(%): 80.0 5.0 4.7 4.5 4.4 4.4 4.3 3.9 3.3 2.1 2.1 2.0 1.8 1.8 1.8 1.7  
```

One or a few processes are well-used, and the rest are used only very little. This tells you that your job can use multiple processes, but that this amount - 16 - is probably too much. It's spending almost all its time on single-threaded processing, and only a small part of time actually running in parallel. You most likely want to reduce the number of cores to 8 or even less. You will lose very little time, and will free up resources to, for instance, run another job like it in parallel.


#### Any future plans for Ruse?

We now have the ability to measure the process CPU use and to use RSS and PSS for memory estimation. This matches what I envisioned for Ruse, and I currently have no specific future plans

A couple of possible directions that could be interesting:

* Find a non-intrusive, low-resource way to track core usage per process. I don't see a way to do that now.

* Rewrite it in Rust :) No, seriously, I would have benefited from using Rust for this, and if I had started Ruse this year I probably would have. 

I am open to any suggestions, and patches are always welcome!


[Slurm]: https://slurm.schedmd.com/
