+++
title = "Parallel Strategies Discovered by a Parallel Noob"
date = "2023-09-17T20:46:42+0000"
author = "Aaron Knudtson"
authorTwitter = "" #do not include @
# cover = "bingo"
tags = ["parallelism", "channels", "CLI", "rust", "programming", "GNU Parallel"]
keywords = ["Parallel", "Beginner", ""]
# description = "This is a default description"
showFullContent = false
draft = true
+++

I have never intentionally spawned a thread. I have never written code that 
required an semaphore, mutex, or whatever other name you can find for a locking
mechanism. My professional work has consisted of low level programming in C and C++,
so it's not like it was impossible for me to get to that point, but it just hasn't
happened. Nevertheless, I wanted to understand the oh so sweet secrets of parallelism.

Recently I've grown appreciation for the amount you can get done via the command
line. Learning my environment and growing accustomed to using commands like cat, 
sed, grep, awk, and xargs, while chaining them together, has greatly improved
my understanding of the systems I work on daily. My crowning achievement so far 
is making an function that uses a make output to generate a compilation database 
for our C++ codebase. At work, I'm limited on the software available for use (no 
jq even), so my solution is a bash function with a chain of echo, grep, and sed, all 
orchestrated by an gorgeous xargs that applies the commands to each compilation
command.

## Enter the parallel lane
Recently, I learned about GNU parallel. For those unfamiliar, GNU parallel is a 
CLI tool that works similar to xargs. Pipe some input into stdin, specify a 
command to run on each piece of data, and parallel will execute the jobs as the
name implies, in parallel. It's a fantastic method of adding parallelism without
virtually any effort.

I timed my compilation database generation function - a whole 2 seconds. Surely I 
could do better by swapping xargs for parallel. I had 8 cores, so that should mean 
about 8 times faster right? That part of the build step wouldn't even be noticed!

...It was faster, but only twice as fast. The compilation database generation still 
took about a second. That's quick, fast enough to not even notice during the build
process. But I enjoy learning opportunities, so I asked the question:

> Can I make GNU parallel faster?

It's a bold question, and I have enough self-examination to know I was asking
that hubris wasn't clouding my judgement. GNU parallel is a fantastic tool, 
and there are surely reasons that it is designed in the way that it is. By creating
a similar CLI tool, I would probably make many of the same design decisions. 
So I set out to create a similar tool in what I think is the best language for CLI 
tools, rust. I love the CLAP crate, and have heard rust advertised as "fearless 
concurrency." This article follows my documented experimentation, findings, and lessons 
learned along the way. I'll also provide some benchmarks along the way, and compare 
to parallel and xargs.

# Initial Design
The first part was to come up with a name for my project. Inspired by ripgrep and 
the 2 letter command `rg`, I chose ripparallel and `rp`. As for design, I figure 
there are four main parts to this program. Parsing arguments, processing a command, 
printing to stdout, and doing most of all that in parallel. I figured the first place 
to look was rust's std::thread. It looks simple enough - just pass a closure to the 
thread::spawn function and you've got a worker. I've heard about channels for passing 
data between threads, and I know they are primitives in Go, but I thought I could
outthink their necessity - I could use an iterator, chunk an array based on the number of 
jobs to run in parallel, and pass a chunk to each thread.

After an hour of trying to implement this, I could already tell it was the wrong
solution. I really felt like I was trying to sneak around the borrow checker, and 
even had to use the once_cell crate to declare a static vector of strings that was
mutable. Pushing and getting lines from this vector required using the "unsafe" 
keyword twice - my first ever venture out of the safeties of normal rust. But 
I was determined - and I got it to work. 

The command I used to benchmark is quite simple. I opened up a terminal and ran:
```bash
# gtime is GNU time rather than bash's builtin time
seq 1 10000 | gtime -f "\nTime Elapsed: %e s\nMax Resident: %M kb" \
./target/release/rp -I {} echo {}
```
and subbed in parallel or xargs when running those tests. All tests are run on my 
Macbook M1 Max with 10 CPU cores. I ran the tests 3 times each to establish averages. 
It's a simple test, but it's a start. It does not necessarily test a more realistic 
workload like generating a compilation database or compressing a list of files. 
However, it should test the area where I think there is the most speed to be gained 
- the overhead necessary to spawn a process. Perl (GNU parallel is written in the 
wingings of programming, Perl) may be relatively fast and really good at string parsing, 
but rust is another level. Anyways, I ran the benchmark and came up with these results:
```bash
# xargs
$ seq 1 10000 | xargs -I {} echo {}
Time Elapsed: 11.05 s
Max Resident: 27120 kb
# parallel
$ seq 1 10000 | parallel -I {} -j 9 echo {}
Time Elapsed: 23.42 s
Max Resident: 19376 kb
# rp
$ seq 1 10000 | rp -I {} -j 9 echo {}
Time Elapsed: 4.00 s
Max Resident: 4032 kb
```
Look at that! One implementation in, and I've already achieved it! However, this
isn't a great comparison yet. It does not have an option to maintain output order with 
the input order, and outputs are not written to stdin until the end of the program. Instead, 
the output of each command is collected in to a vector of strings returned by joining 
the threads. Then, all strings from all vectors are written to stdout. You won't see 
any text on screen until the program is complete.

This also doesn't look like it will scale well. All output is kept in memory until the
end, and there is no indication that work is being done until the very end. I decided
it was worth checking out those channels after all.

It was around this time that I considered giving the responsibility of writing
to stdout to the worker threads. I had an idea of how this would work, but asked
chatGPT to just do the boilerplate code for me. It gave me this code:
```rust
use std::sync::{Arc, Mutex};
use std::thread;
use std::sync::mpsc::{channel, Sender};

fn main() {
    // Create a Mutex for stdout
    let stdout = Arc::new(Mutex::new(std::io::stdout()));

    // Define a list of 100 lines
    let lines: Vec<String> = (1..=100).map(|i| format!("Line {}", i)).collect();

    // Create a channel for threads to send lines
    let (sender, receiver) = channel();

    // Clone the Arc for each thread
    let thread_stdout = stdout.clone();

    // Create a thread pool with 10 threads
    let mut handles = vec![];

    for _ in 0..10 {
        let thread_receiver = receiver.clone();
        let thread_stdout = thread_stdout.clone();

        let handle = thread::spawn(move || {
            let mut stdout = thread_stdout.lock().unwrap();
            for line in thread_receiver {
                writeln!(stdout, "{}", line).unwrap();
            }
        });

        handles.push(handle);
    }

    // Send lines to the channel from the main thread
    for line in lines {
        sender.send(line).unwrap();
    }

    // Main thread can continue processing or exit without waiting for other threads
    // ...

    // Wait for all threads to finish
    for handle in handles {
        handle.join().unwrap();
    }
}
```
Notice the bug? Here, I'll zoom in.
```rust
    // ...
    for _ in 0..10 {
        let thread_receiver = receiver.clone();
        let thread_stdout = thread_stdout.clone();

        let handle = thread::spawn(move || {
            let mut stdout = thread_stdout.lock().unwrap(); // stdout lock acquired too early
            for line in thread_receiver {
                writeln!(stdout, "{}", line).unwrap();
            }
        });

        handles.push(handle);
    }
    // ...
```
The first threads spawned acquires a lock on stdout. Every other thread then waits
for this lock to be given up, causing each thread to block and only the first thread
can do any work. Effectively 10 threads are spawned, but 9 are sitting doing nothing
until the 1 thread finishes all of the work. ChatGPT gave me valid rust code and 
everything compiled, but the code still makes no sense. I tried to reprompt it and 
guide it to avoid acquiring a lock before entering the loop, but after giving it 3 
chances (even after I explicitly pointed it out) ChatGPT could never solve the issue. 
ChatGPT is a great tool, but sometimes gives code that looks right but introduces 
weird bugs that are not obvious and difficult to debug. I gave up on ChatGPT and 
moved on to my next idea.

# Equal Job Allocator
Next came what I called the "equal job allocator". Channels, still a foreign concept
to me, were surprisingly simple to implement. I created an unbounded channel for each
thread, where each thread waited until some line of input was sent. The command was
executed with the appropriate input, and the output was sent to a single receiver on
the main thread that wrote to stdout. This was also my first time using the Arc smart
pointer, and it took a minute until I understood the implications of borrowing between
threads. Nevertheless, I got it to work, and ran the benchmark.
```bash
$ seq 1 10000 | rp -I {} -j 9 echo {}
Time Elapsed: 3.99 s
Max Resident: 4032 kb
```
Well, I guess channels aren't much overhead after all. I was quite pleased with
this result. Outputs are printed to the screen while running the command, it was
just as fast as my first solution, and ordering seemed possible with the right setup.
This is where I really started to get creative. I did some printf debugging when 
a thread ran through all of its jobs and noticed some threads finished far earlier than
others. This is when I remembered about efficiency cores - and my computer has two.
I don't know enough about CPU design to tell you what makes the more efficient,
but on my machine efficiency cores run run at 2064 MHz compared to 3220 MHz for 
performance cores. I wanted to try an iteration of threads deciding when they are 
ready for a new job, rather than jobs being preallocated.

By this point I had spent some time cleaning up and abstracting some parts of my code. 
It ended up locking me into some design choices and difficult to change, so I ended 
up scrapping the whole abstraction. I made a decision to write as much as I could in 
main.rs, and let abstractions reveal themselves rather than trying to force it. 
The goal now was to just write the program, and try to not abstract anything until 
I settled on a design. 

# Single Producer, Multi Consumer
I was a little wary of implementing this, as it required threads to pull from the same
channel. I don't know exactly what the internal design of a channel is, but I imagine 
it requires acquiring a lock on the something before pulling a piece of data. I didn't 
like this idea because I've heard Arc<Mutex> and similar locking constructs can be a bottleneck. 
Regardless, I thought it was a good exercise in gaining experience with an unfamiliar 
strategy. So a thread is spawned to load a channel up with inputs, worker threads are 
created to pull the next input off the stack, execute the command, and 
send the output back via a shared sender to the stdout printer. 

I ran the tests, hoping the channel reading wouldn't be too much of a bottleneck...
```bash
$ seq 1 10000 | rp -I {} -j 9 echo {}
Time Elapsed: 2.19 s
Max Resident: 3972 kb
```
WOW. I should really stop making assumptions without testing them. This was shaping up
to be a fantastic lesson in programming already, and at this point I knew I needed to share
my findings and lessons learned. An almost 2x improvement, and I wasn't sure it was
going to make much of a difference in the first place.

This was also with the addition of an ordering function that consisted of an index of 
the last received job id, and stores all other jobs that come in until the next expected 
job id comes in. So if job #20 was the last received, jobs 22..n would be stored in 
a vector until job 21 came in. I could have used a hashmap, but I think Bjarne Stroustrup's
sentiment that you should just use a Vector until you see a reason to reach for another 
container.

The issue with this implementation may be that channels are initially filled. I'm
not sure how much of an issue this really is, but I think it's probably better to read 
from stdin when a new job is needed. This got me thinking about an implementation I call:

# The Broker
The broker is a separate thread which uses a line iterator coming from stdin to send 
jobs out to a worker thread when it requests a new one. Each worker thread has its own 
channel to receive work from, and the broker sends a new job based on the thread that 
just finished. Theoretically this should use less memory, as inputs are read in only 
when they are needed. I implemented the design, tested and tested my theory.
```bash
$ seq 1 10000 | rp -I {} -j 9 echo {}
Time Elapsed: 2.21 s
Max Resident: 3680 kb
```
This consistently shows lower memory and doesn't appear to have a measurable negative 
time impact. Great! I'm getting the hang of this now. It's at this point that I look 
more into bounded channels. I was hit with a wave of understanding. The title "channels"
and the crate name "crossbeam" previously made me feel like I was using some far fetched 
technology. It feels like I'm shooting laser beams in space like I'm in some sci-fi 
movie. But now I think of channels far simpler - basically a shared pointer to a
FIFO queue of messages, with a sender arm that can enqueue and a receiver arm that can 
dequeue. If that's true, bounded channels have a fixed size - surely leading to even 
LESS memory consumption. Time to test it out:
```bash
$ seq 1 10000 | rp -I {} -j 9 echo {}
Time Elapsed: 2.21 s
Max Resident: 3400 kb
```
Look at that, even better in terms of memory! I still don't like the idea of a separate 
thread for allocating work and printing to stdout. I ran some profiling with the cargo 
flamegraph tool, generated some flamegraphs and saw that both threads spent plenty of
time waiting for messages to come in from the channels. 

# Final Implementation
The final implementation combined many concepts we have already covered. Rather than 
each thread having its own channel, a single producer, multi consumer channel was 
used for job allocation, and a multi producer single consumer channel was used for 
outpus. When a new output comes in, the job allocator checks how many messages short 
the channel is and fills it back up. Then it prints the new output to stdout, or uses 
the waiting room construct to maintain order. 
```bash
$ seq 1 10000 | rp -I {} -j 9 echo {}
Time Elapsed: 2.19 s
Max Resident: 3330 kb
```
Based on my workload, the channel messaging is such low overhead that threads spend
almost no time at atll waiting for another thread to give up a lock to read the 
next message off the channel. My preconcieved notion that a shared lock is slow
would have led me to not only premature optimizations, but incorrect optimizations.
The threads spend most of their time running the process, not waiting for a channel 
message. I was glad I took this challenge on, and I believe I arrived at the best 
solution after iteration. I tried to avoid abstracting this as I went and just 
iterating as fast as I could - the code is ugly, but thinking through the flow of
the program allowed me to discover abstractions that I intend to implement. The 
first abstraction to pop into your head is not always the right one - implement your 
idea, then go back and make it maintainable.

# Rayon
I also tested the Rayon crate's par_iter, which iterates in parallel. There aren't
any knobs to tune and the order of outputs seems very jumbled up, but I ran it anyways.
```bash
$ seq 1 10000 | rp -I {} -j 9 echo {}
Time Elapsed: 2.18 s
Max Resident: 4080 kb
```
To be hones, I didn't expect that. Rayon is a major crate that has been downloaded > 
70 million times. I'm as grug brained as the next dev, but I managed to come up with
a solution that has a similar execution time. I think that speaks volumes to both rust, 
and being able to rapidly iterate through new ideas.

# Summary of Results
{{< chart 90 400 >}}
{
    type: 'line',
    data: {
        labels: ['parallel', 'xargs', 'rp-final'],
        datasets: [{
            label: 'Execution Time',
            data: [55.14, 11.05, 10.40],
            borderColor: 'rgba(255, 99, 132, 1)',
            borderWidth: 1,
            tension: 0.05
        }]
    },
    options: {
        title: {
            display: true,
            text: 'Printing 1 to 10,000 with 1 job',
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
{{< chart 90 400 >}}
{
    type: 'line',
    data: {
        labels: ['GNU parallel', 'initial_implementation', 'equal_job_allocator', 'SPMC', 'broker', 'broker_bounded', 'final', 'rayon'],
        datasets: [{
            label: 'Execution Time',
            yAxisID: 'A',
            data: [23.42, 4.00, 3.99, 2.19, 2.21, 2.21, 2.20, 2.18],
            borderColor: 'rgba(255, 99, 132, 1)',
            borderWidth: 1,
            tension: 0.05
        },
        {
            label: 'Max RSS',
            yAxisID: 'B',
            data: [19338, 4021, 4027, 3974, 3680, 3402, 3323, 4091],
            borderColor: 'rgba(54, 162, 235, 1)',
            borderWidth: 1,
            tension: 0.05
        }]
    },
    options: {
        title: {
            display: true,
            text: 'Printing 1 to 10,000 in parallel',
            fontSize: '20'
        },
        responsive: true,
        maintainAspectRatio: false,
        scales: {
            yAxes: [{
                id: 'A',
                scaleLabel: {
                    display: true,
                    labelString: 'Time (s)'
                },
                ticks: {
                    beginAtZero: true,
                },
            },
            {
                position: 'right',
                id: 'B',
                scaleLabel: {
                    display: true,
                    labelString: 'Memory (kb)'
                },
                ticks: {
                    beginAtZero: true,
                },
            }],
        }
    }
}
{{< /chart >}}
{{< chart 90 400 >}}
{
    type: 'line',
    data: {
        labels: ['GNU parallel', 'initial_implementation', 'equal_job_allocator', 'SPMC', 'broker', 'broker_bounded', 'final', 'rayon'],
        datasets: [{
            label: 'Line Chart',
            data: [2291.76, 382.89, 397.91, 218.03, 220.47, 220.21, 218.44, 215.97],
            borderColor: 'rgba(255, 99, 132, 1)',
            borderWidth: 1,
            tension: 0.05
        },
        {
            label: 'Max RSS',
            yAxisID: 'B',
            data: [19328, 103424, 51584, 67408, 3744, 3520, 3376, 49120],
            borderColor: 'rgba(54, 162, 235, 1)',
            borderWidth: 1,
            tension: 0.05
        }]
    },
    options: {
        title: {
            display: true,
            text: 'Printing 1 to 1,000,000 in parallel',
            fontSize: '20'
        },
        responsive: true,
        maintainAspectRatio: false,
        scales: {
            yAxes: [{
                id: 'A',
                scaleLabel: {
                    display: true,
                    labelString: 'Time (s)'
                },
                ticks: {
                    beginAtZero: true,
                },
            },
            {
                position: 'right',
                id: 'B',
                scaleLabel: {
                    display: true,
                    labelString: 'Memory (kb)'
                },
                ticks: {
                    beginAtZero: true,
                },
            }],
        }
    }
}
{{< /chart >}}

# Lessons Learned 
- Making early abstractions often leads to the wrong abstractions. Just hammer out 
an implementation first, and let the abstractions reveal themselves.
- Parallelism isn't as daunting as I once thought
- Exploring fundamentals of new technologies can help you understand other concepts.
After exploring channels, I think I can vizualize the plumbing for how futures might work.
- Do a cool project and document your findings. This was a fantastic use of time and 
really broadened my understanding of some core multithreading software principles. 
Learning in a class or video would be one thing, but finding a project and doing something 
cool with it provided a better learning opportunity than a book can give.

# Update
While writing this article, I stumbled upon another trick that I call "turbomode". 
The trick optimizes the spawning process of a command, and cuts an execution time 
of printing 1 to 10000 from 2.19 seconds down to 0.08 seconds. 

You bet there's an article coming about that.
