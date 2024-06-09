+++
title = "How I made GNU Parallel 300x faster"
date = "2024-06-08T12:00:00+0000"
author = "knudtty"
authorTwitter = "" #do not include @
# cover = "bingo"
tags = ["parallelism", "channels", "CLI", "rust", "xargs", "programming", "GNU Parallel"]
keywords = ["Parallel", "Beginner", "GNU", "xargs", "optimize", "blog", ""]
# description = "This is a default description"
showFullContent = false
draft = false
+++

# Intro

I was somewhat of a parallel and concurrency noob (aka never even touched the
concepts) about a year ago. I had never spawned a thread, and hardly sniffed
even a goroutine outside of a tutorial. I had been vim-pilled about a year
earlier and kept having a hunger to dive into more challenging programming
concepts. I became a bit of a primeagen enjooyer (either
[this](https://www.youtube.com/@ThePrimeTimeagen) or
[this](https://www.youtube.com/@ThePrimeagen), which is *really* his main
channel anyways?). During one stream, he was raving about a program called GNU
Parallel, which allows you to [live life in the parallel
lane](https://www.gnu.org/software/parallel/).

## Experimenting

Fascinated by this tool that allows you to parallelize CLI tasks with minimal
effort, I decided to give it a shot! My test load was to copy all html files I
could find on my system and gzip them. 

First I tested with xargs:

```bash
$ alias time='gtime -f "\nTime Elapsed: %e s\nMax Resident: %M kb"'
$ find . -name "*.html" | time xargs -I {} gzip -k {}
Time Elapsed: 5.34 s
Max Resident: 10368 kb
```

Now with parallel:

```bash
$ find . -name "*.html" | time parallel -I {} gzip -k {}
Time Elapsed: 8.25 s
Max Resident: 20352 kb
```

Huh? How could running something in parallel be slower? I tested by setting the
number of cores with the -j flag, and yet single core xargs was superior. In
fact, single core parallel took nearly 30 seconds. It was then that I found an
excuse to use a blazingly fast modern programming language, rust, to improve
the performance of GNU parallel. The resulting project absorbed me for a week
or two outside of work hours and was one of the most valuable iterative
learning experiences I've had. Also I learned a new way to forkbomb my
computer.

> My Thesis

GNU Parallel is written in Perl and xargs is written in C. I thought that there
must be so much overhead in parsing args, and maybe there's just a lot of
overhead in the perl runtime. Spoiler alert: I was like 90% wrong.

> The Design

I named the project **ripparallel** based on [a legendary Rust
tool](https://github.com/BurntSushi/ripgrep) that came before me. Since the
gzip command does not test the stdout, I needed a command that does for
development purposes. As a result I used this command and these measurements
for benchmarking:

```bash
$ seq 1 10000 | xargs -I {} echo {}
Time Elapsed: 11.05 s
Max Resident: 27120 kb
$ seq 1 10000 | parallel -I {} -j 10 echo {}
Time Elapsed: 23.42 s
Max Resident: 19376 kb
```

When I'm in a learning phase I typically follow the "Code first, ask questions
as they come up" approach. I used Clap for arg parsing and the rust standard
library `std::process::Command` for running a command. I didn't know much about
concurrency, so I just ran with what I knew.

My first pass was a battle against the Rust compiler the entire way. I read all
input from stdin up front, pushed it into a static vector, chunked the vector
into the number of jobs I wanted to run, and returned out a vector of
JoinHandle's. Then I would join all of the threads and print the output to
stdout. Technically parallelism, but all I really did was chunk up the input
into n jobs and spawn n threads to do it. There was no ability to see stdout
until all jobs were completely done running. But I ran with it:

```bash
$ seq 1 10000 | rp -I {} -j 10 echo {}
Time Elapsed: 4.00 s
Max Resident: 4032 kb
```

Holy cripes that actually worked. But this program doesn't really behave like
the other two, so let's do something different. Next I investigated channels.

<style>
svg {
    width: 100%;
    height: auto;
    overflow: visible;
}
</style>

<div style="width: 100%; height: auto; overflow: visible;">
{{< svg "static/rp_iter1.svg" >}}
</div>

> Iteration 2: Channels

It is very unsafe to share memory to the same object between threads. Channels
help send and receive whatever memory you want to share between threads. I
created n unbounded (unlimited size) channels where n was the number of
parallel jobs to run at a time. Stdin was loaded into a vector up front, and
then round-robin filled all channels up for receivers to pull and process the
commands in a job. After a job was complete, a single shared channel was used
to send stdout of that particular process to my main thread. Each entry was
pulled off and printed to rp's stdout as they came in.

I was worried that this would produce a choke point. Each time a thread wants
to send something through a channel it must acquire a lock. This means every
other thread that also wants to send something through the channel will block
until the lock is freed. This is obviously sounds like something that cause
performance to drop. As I initially expected channels didn't speed anything up,
but they didn't make anything worse. My benchmark completed in 4 seconds again,
but this time lines were being printed out sequentially. However, things were
horribly out of order. I eventually came up with a solution to optionally keep
order with a `-k` flag, which is the same option that GNU parallel provides.

I also broke out `cargo flamegraph` to see what my program is actually doing.
To my surprise, waiting for a lock on the shared channel took virtually no time
at all. The process that my threads were running (in this case the echo
command) took so much more time to execute than sending a Vec<String> through a
channel that I was no longer worried about this choke point, at least at the 9
threads I was running with.

<div style="width: 100%; height: auto; overflow: visible;">
{{< svg "static/rp_iter2.svg" >}}
</div>

> Iteration 3: Scheduling Optimizations

This job allocation felt eerily similar to CPU scheduling, which I also know
nothing about. But I know enough that a round robin strategy probably isn't the
best. I also realized during this process that I have 2/10 of my CPU cores are
efficiency cores and run at 64% the clock speed of my performance cores. As the
saying goes "You're only as good as your weakest link" and if any of my threads
are allocated to an efficiency core, that alone is a 36% hit to productivity,
assumming equal job distribution and job time. It might be better to pull a job
off of a shared channel when you're ready.

```bash
$ seq 1 10000 | rp -I {} -j 10 echo {}
Time Elapsed: 2.19 s
Max Resident: 3330 kb
```

It turns out that made a big difference! I also made some other optimizations,
improved argument parsing, and had some micro-iterations between. I also tested
by swapping in Rayon's par_iter instead of my custom solution, and got about 1%
extra performance. Rayon's virtually identical performance was validation that
I did a good job and I was satisfied. I learned a lot and was ready to rest
with my new knowledge about threads, channels, and concurrency. But that's NOT
the 300x I promised you in the title.

<div style="width: 100%; height: auto; overflow: visible;">
{{< svg "static/rp_iter3.svg" >}}
</div>

I stumbled across [this
document](https://www.gnu.org/software/parallel/parallel_alternatives.pdf) of
similar programs to GNU parallel, and found out there was one in rust. I cloned
and build the [repo](https://github.com/mmstick/parallel) and gave it a shot.
The performance was actually worse than mine, but they did have far more
features and were closer to implementing the full GNU parallel featureset. But
interestingly, there was a note: "Rust parallel focuses on speed. It is almost
as fast as xargs, but not as fast as parallel-bash."

What kind of speeds could this mythical [parallel-bash](https://github.com/Akianonymus/parallel-bash) pull? I cloned that and tested it out:

```bash
$ seq 1 10000 | ./parallel-bash.bash -p 10 echo
Time Elapsed: 0.56 s
Max Resident: 3728 kb
```

WHADDOYOUMEAN 0.56 seconds?? How could rust POSSIBLY be slower than bash? From
what I read this is also doing round robin - I already determined that was
SLOWER than pulling a job off when you're ready.

This, [ladies and gentlebots](https://www.youtube.com/watch?v=eeMbBNBbH2s), is
when I learned about [bash
builtins](https://www.gnu.org/software/bash/manual/html_node/Bash-Builtins.html).
It sounds stupid to say but it's bear with me. Bash builtins are built into
bash. That means you are not starting a new process when you run "echo foo" in
a bash shell. When you start a process on your computer, there's a lot that
your kernel does under the hood. Check out [cpu.land](https://cpu.land/) if you
want a free, easily digestible resource to learn more about what your CPU and
kernel are actually doing no matter what technical level you are currently at.
But with bash builtins, bash will just call it's echo function with the
arguments you give it rather than starting a process and doing syscalls to the
kernel.

I looked a little more into what GNU parallel is doing. It actually runs your
command not just as a process, but as a process inside of a shell. That adds
extra overhead, as you first have to start your shell process, but then also
your actual command to run. This allows you to run bash functions, which
current implementation does not.

> Iteration 4 (Final)

So how do you beat bash builtins in your rust program? That's right - embrace
more bash. This article is getting lengthy so I won't go into full specifics,
but I effectively ran a bash repl in each thread at the beginning of my program:

```bash
while true; do read line; 
    eval "$line"; 
    printf {RAND_STRING}; 
    printf {RAND_STRING} >&2; 
done
```

In this case {RAND_STRING} is a 16 character long random string that I generate
in each thread. It looks super funky, but try running that while loop in your
shell. It will behave remarkably similar to a bash shell you're used to
running. A creative problem requred a creative solution. Each job that a thread
receives is fed in through stdin to this repl and then "\n" is written to
stdin. Stdout is pulled when available. If the last 16 characters do not match
my random string, the process is not done. All characters that are not the
random string are saved into a vector and pushed to the main thread to print to
stdout.

```bash
$ seq 1 10000 | time rp echo
Time Elapsed: 0.09 s
Max Resident: 3872 kb
```

And voila! Also, I found out xargs can do commands in parallel. For fun, let's
run of the different tools with an extra zero:

| seq 1 100000 \| *Command* | Time (s) | RSS (kb) |
| ----------- | ----------- | ----------- |
| rp -j 10 echo {}                | 0.82   | 3824 |
| rp -j 1 echo {}                 | 3.28   | 3168 |
| xargs -I {} echo {}             | 129.39 | 254720 |
| xargs -I {} -P 10 echo {}       | 61.31  | 254688 |
| ./parallel-bash.bash -p 10 echo | 7.67   | 6944 |
| parallel -j 10 echo {}          | 264.42 | 19456 |
| parallel -j 1 echo {}           | a looooong time | who knows |

The benefits of this solution include a blazingly fast language for parsing and
orchestrating concurrency, able to use bash functions and builtins, and I get
to say "I use rust BTW". This solution is pretty well optimized and required a
lot of creativity. I'm sure someone will find a better solution anyways some
day.

The title is definitely clickbaity - I suspect for longer lived processes,
this tool will yield diminishing returns over others. Other tools are far
fuller featured than this one as well - GNU parallel has a whole unique syntax
associated with it. I might implement some of the input modifiers at some
point, but I don't know anyone who uses the `:::` syntax from parallel. The
seq command isn't a super representative process for something you would
actually do, so I've included results for both the seq and gzip benchmarks
below.

Do you want to use this tool? Should I add more features? Stop on over at the
[ripparallel repo](https://github.com/knudtty/ripparallel) if that's the case.

# Benchmark Results

{{< chart 90 400 >}}
{
    type: 'bar',
    data: {
        labels: ['parallel -j 10', 'xargs -I {}', 'xargs -I {} -P 10', './parallel-bash.bash -p 10', 'rp -j 10', 'rp -j 1'],
        datasets: [{
            label: 'Execution Time (s)',
            data: [264.42, 129.39, 61.31, 7.67, 0.82, 3.28],
            borderColor: 'rgba(255, 99, 132, 1)',
            borderWidth: 1,
            tension: 0.05
        }]
    },
    options: {
        title: {
            display: true,
            text: 'seq 1 100000 | {command} echo {}',
            fontSize: '20'
        },
        responsive: true,
        maintainAspectRatio: false,
        scales: {
            x: {
                display: true,
            },
            y: {
                display: true,
                type: 'logarithmic',
            },
        }
    }
}
{{< /chart >}}

<br/>
<br/>
<br/>

{{< chart 90 400 >}}
{
    type: 'bar',
    data: {
        labels: ['parallel -j 10', 'xargs -I {}', 'xargs -I {} -P 10', './parallel-bash.bash -p 10', 'rp -j 10', 'rp -j 1'],
        datasets: [{
            label: 'Execution Time (s)',
            data: [8.28, 5.40, 1.67, 1.10, 1.03, 5.41],
            borderColor: 'rgba(54, 162, 235, 1)',
            borderWidth: 1,
            tension: 0.05
        }]
    },
    options: {
        title: {
            display: true,
            text: 'find . -name "*.html" | {command} gzip -k {}',
            fontSize: '20'
        },
        responsive: true,
        maintainAspectRatio: false,
        scales: {
            yAxes: [{
                scaleLabel: {
                    display: true,
                    labelString: 'Time (s)'
                },
                ticks: {
                    beginAtZero: true,
                },
            }],
        }
    }
}
{{< /chart >}}

P.S. I also ran tests with [pigz](https://zlib.net/pigz/), which does gzip in parallel using the command `find . -name "*.html" | time xargs pigz -k` and recorded a benchmark time of 1.06 seconds. That's virtually the same as rp + gzip!

# Final Sequence Diagram

<div style="width: 100%; height: auto; overflow: visible;">
{{< svg "static/rp_iter4.svg" >}}
</div>

This is roughly what's happening. If I add anymore it will look even more
complicated.

# Flamegraph for the Nerds
<div style="width: 100%; height: auto; overflow: visible;">
{{< svg "static/flamegraph.svg" >}}
</div>
