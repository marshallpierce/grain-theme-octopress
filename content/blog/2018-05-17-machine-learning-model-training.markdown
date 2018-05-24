---
layout: post
title: "Profiling and Optimizing Machine Learning Model Training with PyTorch"
date: "2018-05-24 07:45"
author:
categories: ["machine-learning", "performance", "pytorch"]
comments: true
published: true
---

There's lots of innovation out there building better machine learning models with new neural net structures, regularization methods, etc. Groups like [fast.ai](http://www.fast.ai/) are [training complex models quickly on commodity hardware](http://www.fast.ai/2018/04/30/dawnbench-fastai/) by relying more on "algorithmic creativity" than on overwhelming hardware power, which is good news for those of us without data centers full of hardware. Rather than add to your stash of creative algorithms, this post takes a different (but compatible) approach to better performance. We'll step through measuring and optimizing model training from a systems perspective, which applies regardless of what algorithm you're using. As we'll see, there are nice speedups to be had with little effort. While keeping the same model structure, hyperparameters, etc, training speed improves by 26% by simply moving some work to another CPU core. You might find that you have easy performance gains waiting for you whether you're using an SVM or a neural network in your project.

We'll be using the [fast neural style](https://github.com/pytorch/examples/tree/master/fast_neural_style) example in [PyTorch's example projects](https://github.com/pytorch/examples) as the system to optimize. If you just want to see the few code changes needed, check out [this branch](https://github.com/pytorch/examples/compare/master...marshallpierce:perf-experiments?expand=1). Otherwise, to see how to get there step-by-step so that you can replicate this process on your own projects, read on.

If you want to follow along, start by downloading the [2017 COCO training dataset](http://images.cocodataset.org/zips/train2017.zip) (18GiB). You'll also need a Linux system with a recent kernel and a GPU (an nVidia one, if you want to use the provided commands as-is). My hardware for this experiment is an i7-6850K with 2x GTX 1070 Ti, though we'll only be using one GPU this time.

# Getting started

If you're a `virtualenv` user, you'll probably want to have a virtualenv with the necessary packages installed:

```
mkvirtualenv pytorch-examples
workon pytorch-examples
pip install torch torchvision numpy
```

Clone the [pytorch/examples](https://github.com/pytorch/examples) repo and go into the `fast_neural_style` directory, then start training a model. The batch size is left at the default (4) so it will be easier to replicate these results on smaller hardware, but of course feel free to increase the batch size if you have the hardware. We run `date` first so we have a timestamp to compare later timestamps with, and pass `--cuda 1` so that it will use a GPU. The directory to pass to `--dataset` should be the one _containing_ the `train2017` directory, not the path to `train2017` itself.

```
date ; python neural_style/neural_style.py \
    train --dataset /path/to/coco/ \
     --cuda 1 \
     --save-model-dir model
```

While that's running for the next few hours, let's dig in to its performance characteristics. `vmstat 2` isn't fancy, but it's a good place to start. Unsurprisingly, we have negligible `wa` (i/o wait) CPU usage and little block i/o, and we're seeing normal user CPU usage for one busy process on a 6-core, 12-thread system (10-12%):

```
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 2  0      0 978456 1641496 18436400    0    0   298     0 8715 33153 11  3 86  1  0
 1  0      0 977804 1641496 18436136    0    0   256     4 8850 33866 11  3 86  0  0
 3  0      0 966088 1641496 18436136    0    0  1536    12 9793 33106 18  3 79  0  0
 2  0      0 973500 1641496 18436540    0    0   256  2288 9795 36201 12  3 84  1  0
 1  0      0 973576 1641496 18436540    0    0   256     0 8433 32495 10  3 87  0  0
```

Moving on to the GPU, we'll use `nvidia-smi dmon -s u -i 0`. `dmon` periodically outputs GPU info, and we're limiting it to utilization (`-s u`) and we want the first GPU device (`-i 0`).

```
# gpu    sm   mem   enc   dec
# Idx     %     %     %     %
    0    67    40     0     0
    0    57    35     0     0
    0    69    41     0     0
    0    30    14     0     0
    0    95    52     0     0
    0    65    41     0     0
    0    68    27     0     0
    0    73    34     0     0
```

Now this is more interesting. GPU utilization as low as 30%? This workload should basically be waiting on the GPU the entire time, so failing to keep the GPU busy is a problem. 

To see if there's something seriously wrong, `perf stat` is a simple way to get a high-level view of what's going on. Just using 100% of a CPU core [doesn't mean much](https://www.youtube.com/watch?v=QkcBASKLyeU); it could be spending all of its time waiting for memory access or pipeline flushes.  We can attach to a running process (that's the `-p <pid>`) and aggregate performance counters. Note that if you're using a virtual machine, you may not have access to performance counters, and even my physical hardware doesn't support all the counters `perf stat` looks for, as you can see below.

```
sudo perf stat -p 15939
```

After letting that run for a couple of minutes, stopping it with `^C` prints a summary:

```text
     167611.675673      task-clock (msec)         #    1.060 CPUs utilized          
            22,843      context-switches          #    0.136 K/sec                  
             1,986      cpu-migrations            #    0.012 K/sec                  
                 7      page-faults               #    0.000 K/sec                  
   641,956,257,119      cycles                    #    3.830 GHz                    
   <not supported>      stalled-cycles-frontend  
   <not supported>      stalled-cycles-backend   
   803,246,123,727      instructions              #    1.25  insns per cycle        
   134,551,532,600      branches                  #  802.758 M/sec                  
       943,869,394      branch-misses             #    0.70% of all branches        

     158.057364231 seconds time elapsed
```

Key points:

- 1.25 instructions per cycle isn't awful, but it's not great either. In ideal circumstances, CPUs can retire many instructions per cycle, so if that was more like 3 or 4 then that would be a signal that the CPU was already being kept quite busy.
- 0.7% branch mispredicts is higher than I'd like, but it isn't catastrophic.
- Hundreds of context switches per second is not suitable for realtime systems, but is unlikely to affect batch workloads like this one.

Overall, there's nothing obvious like a 5% branch miss rate or 0.5 IPC, so this isn't telling us anything particularly compelling.

# Profiling

Capturing performance counter information with `perf stat` shows that there's some room to improve, but it's not providing any details on what to do about it. `perf record`, on the other hand, samples the program's stack while it's running, so it will tell us more about what specifically the CPU is spending its time doing.

```
sudo perf record -p 15939
```

After letting that run for a few minutes, that will have written a `perf.data` file. It's not human readable, but `perf annotate` will output disassembly with the percentage of samples where the CPU was executing that instruction. For my run, the [output](${r '/images/optimizing-machine-learning/annotate.txt'}) starts with a disassembly of `syscall_return_via_sysret` indicating that most of the time there is spent on `pop`. That's not particularly useful knowledge at the moment (the process makes a good number of syscalls, so we do expect to see some time spent there), so let's keep looking. The next item is for `jpeg_idct_islow@@LIBJPEG_9.0`, part of PIL (Python Imaging Library, aka Pillow). The output starts with a summary like this:

```
    2.52 libjpeg-4268453f.so.9.2.0[36635]
    1.93 libjpeg-4268453f.so.9.2.0[3624d]
    1.83 libjpeg-4268453f.so.9.2.0[3646a]
    1.53 libjpeg-4268453f.so.9.2.0[3648b]
    1.44 libjpeg-4268453f.so.9.2.0[36284]
    1.16 libjpeg-4268453f.so.9.2.0[36562]
```

That continues for about 50 lines or so. That tells us that rather than having a few hot instructions in this function, the cost is smeared out across many instructions (2.52% at offset 36635, 1.93% at 3624d, etc). Paging down to the disassembly, we find lots of this kind of info:

```
    0.34 :        35ec0:       cltq   
    0.11 :        35ec2:       mov    %rax,-0x40(%rbp)
    0.14 :        35ec6:       shlq   $0xd,-0x38(%rbp)
 libjpeg-4268453f.so.9.2.0[35ecb]    0.72 :       35ecb:       shlq   $0xd,-0x40(%rbp)
 libjpeg-4268453f.so.9.2.0[35ed0]    0.82 :       35ed0:       addq   $0x400,-0x38(%rbp)
 libjpeg-4268453f.so.9.2.0[35ed8]    0.94 :       35ed8:       mov    -0x40(%rbp),%rax
    0.30 :        35edc:       mov    -0x38(%rbp),%rdx
 libjpeg-4268453f.so.9.2.0[35ee0]    0.83 :       35ee0:       add    %rdx,%rax
    0.30 :        35ee3:       mov    %rax,-0x48(%rbp)
    0.28 :        35ee7:       mov    -0x40(%rbp),%rax
    0.00 :        35eeb:       mov    -0x38(%rbp),%rdx
    0.00 :        35eef:       sub    %rax,%rdx
```

You can see the percentage of samples in the left column (sometimes prefixed with the symbol name for extra busy samples) and offset. This snippet is telling us that 0.72% of cycles were spent on `shlq` (shift left), 0.82% on `addq` (integer addition), etc. Note that due to a phenomenon called [skid](https://perf.wiki.kernel.org/index.php/Tutorial) the usage per instruction may be incorrect by several or even dozens of instructions, so these percentage numbers should not be taken as gospel. In this case, for instance, it's unlikely that the two `shlq` instructions at 35ec6 and 35ecb are actually 5x different.

The next section of `perf annotate` output is of `__vdso_clock_gettime@@LINUX_2.6`. VDSO is a way to [speed up certain syscalls](https://www.linuxjournal.com/content/creating-vdso-colonels-other-chicken), notably `gettimeofday`. Not much to see here, other than to note that maybe we shouldn't be calling `gettimeofday(2)` as much.

The next section is of `ImagingResampleHorizontal_8bpc@@Base` from `_imaging.cpython-35m-x86_64-linux-gnu.so`, which has a large block of fairly hot instructions like this:

```
 _imaging.cpython-35m-x86_64-linux-gnu.so[21b3b]    3.39 :        21b3b:       imul   %ecx,%esi
 _imaging.cpython-35m-x86_64-linux-gnu.so[21b3e]    3.81 :        21b3e:       add    %esi,%r11d
 _imaging.cpython-35m-x86_64-linux-gnu.so[21b41]   10.14 :        21b41:       movzbl 0x1(%r8,%rdx,1>
 _imaging.cpython-35m-x86_64-linux-gnu.so[21b47]    2.50 :        21b47:       movzbl 0x2(%r8,%rdx,1>
 _imaging.cpython-35m-x86_64-linux-gnu.so[21b4d]    3.23 :        21b4d:       imul   %ecx,%esi
```

That's a pretty hot `movzbl` (zero-expand 1 byte into 4 bytes). At this point, we have a hypothesis: we're spending a lot of time decoding and scaling images. 

# Flame graphs

To get a clearer view of what paths through the program are the most relevant, we'll use a [flame graph](http://www.brendangregg.com/flamegraphs.html). There are other things we could do with `perf` like `perf report`, but flame graphs are easier to understand in my experience. If you haven't worked with flame graphs before, the general idea is that width = percentage of samples and height = call stack. As an example, a tall, skinny column means a deep call stack that doesn't use much CPU time, while a wide, short column means a shallow call stack that uses a lot of CPU.

Clone the [Pyflame repo](https://github.com/uber/pyflame) and follow their [compile instructions](https://pyflame.readthedocs.io/en/latest/installation.html#compiling). There's no need to install it anywhere (the `make install` step) -- just having a compiled `pyflame` binary in the build output is sufficient.

As with the other tools, we attach the compiled `pyflame` binary to the running process and let it run for 10 minutes to get good fidelity:

```
./src/pyflame -s 600 -p 15939 -o neural-style-baseline.pyflame
```

In the meantime, clone [FlameGraph](https://github.com/brendangregg/FlameGraph) so we can render `pyflame`'s output as an SVG. From the FlameGraph repo:

```
./flamegraph.pl path/to/neural-style-baseline.pyflame > neural-style-baseline.svg
```

The resulting flamegraph looks like this, which you'll probably want to open in a separate tab and zoom in on (the SVG has helpful mouseover info):

<a href="${r '/images/optimizing-machine-learning/neural-style-baseline.svg'}">
<img class='centered' src="${r '/images/optimizing-machine-learning/neural-style-baseline.svg'}"/>
</a>

Most of the time is [idle](https://pyflame.readthedocs.io/en/latest/faq.html#what-is-idle-time), which isn't interesting in this case. Back to pyflame, this time with `-x` to exclude idle time:

```
./src/pyflame -s 600 -p 15939 -x -o neural-style-baseline-noidle.pyflame
``` 

<a href="${r '/images/optimizing-machine-learning/neural-style-baseline-noidle.svg'}">
<img class='centered' src="${r '/images/optimizing-machine-learning/neural-style-baseline-noidle.svg'}"/>
</a>

That's much easier to see. If you spend some time mousing around the left third or so of the graph, you'll find that a significant amount of execution time is spent on image decoding and manipulation, starting with `neural_style.py:67`, which is this:

```python
for batch_id, (x, _) in enumerate(train_loader):
```

Put another way, it's spending enough time decoding images that it's a significant part of the overall execution, and it's all clumped together in one place rather than being spread across the whole program. In a way, that's good news, because that's a problem we can pretty easily do something about. There's still the other 2/3rds of the graph that we'd love to optimize away too, but that's mostly running the PyTorch model, and improving that will require much more surgery. There is a surprising amount of time spent in `vgg.py`'s usage of `namedtuple`, though -- that might be a good thing to investigate another  time. So, let's work on that first third of the graph: loading the training data. 

# Adding some parallelism

By now that `python` command has probably been running for a good long while. It helpfully outputs timestamps every 2000 iterations, and comparing the 20,000 iteration timestamp with the start timestamp I get 21m 40s elapsed. We'll use this duration later.

It's unlikely that we'll make huge strides in JPEG decoding, as that code is already written in a low level language and reasonably well optimized. What we can do, though, is move the CPU-intensive work of image decoding onto another core. Normally this would be a good place to use threads, but Python as a language and CPython as a runtime are not very well suited to multithreading. We can use another `Process` to avoid the GIL (Global Interpreter Lock) and lack of memory model, though, and even though we'll have more overhead between processes than we would between threads, it should still be a net win. Conveniently, the work we want to execute concurrently is already small and fairly isolated, so it should be easy to move to another process. The training data `DataLoader` is set up in `neural_style.py` with

```python
train_dataset = datasets.ImageFolder(args.dataset, transform)
train_loader = DataLoader(train_dataset, batch_size=args.batch_size)
```

and used with

```python
for batch_id, (x, _) in enumerate(train_loader):
```

So, all we need to do is move the loading to another process. We can do this with a [Queue](https://docs.python.org/3/library/multiprocessing.html#multiprocessing.Queue) (actually, one of [PyTorch's wrappers](https://pytorch.org/docs/stable/notes/multiprocessing.html)). Instead of enumerating `train_loader` in the main process, we'll have another process do that so that all the image decoding can happen on another core, and we'll receive the decoded data in the main process. To make it easy to enumerate a `Queue`, we'll start with a helper to make a `Queue` be iterable:

```python
class QueueIterator:
    """
    An Iterator over a Queue's contents that stops when it receives a None
    """
    def __init__(self, q: Queue):
        self.q = q

    def __iter__(self):
        return self

    def __next__(self):
        thing = self.q.get()

        if thing is None:
            raise StopIteration()

        return thing
```

We're making the simple assumption that if we pop `None` off the queue, the iteration is complete. Next, we need a function to invoke in another process that will populate the queue. This is made a bit more complicated by the fact that we can't spawn a process at the intuitive point in the training `for` loop. If we create a new `Process` where we enumerate over `train_loader` (as in the code snippet above), the child process hangs immediately and never is able to populate the queue. The PyTorch docs warn that [about such issues](https://pytorch.org/docs/stable/notes/multiprocessing.html#avoiding-and-fighting-deadlocks), but unfortunately using `torch.multiprocessing`'s wrappers or `SimpleQueue` did not help. Getting to the root cause of that problem will be a task for another day, but it's simple enough to rearrange the code to avoid the problem: fork a worker process earlier, and re-use it across multiple iterations. To do this, we use _another_ `Queue` as a simple communication mechanism, which is `control_queue` below. The usage is pretty basic: sending `True` through `control_queue` tells the worker to enumerate the loader and populate `batch_queue`, finishing with a `None` to signal completion to the `QueueIterator` on the other end, while sending `False` tells the worker that its job is done and it can end its loop (and therefore exit).

```python
def enqueue_loader_output(batch_queue: Queue, control_queue: Queue, loader):
    while True:
        ctrl = control_queue.get()
        if ctrl is False:
            break
        
        for batch_id, (x, _) in enumerate(loader):
            batch_queue.put((batch_id, x))
        batch_queue.put(None)

```

Now we have everything we need to wire it all together. Before the training loop, make some queues and fork a process:

```python
batch_queue = Queue(100)
control_queue = Queue()
Process(target=enqueue_loader_output, args=(batch_queue, control_queue, train_loader)).start()
```

And then iterate in the training loop with a `QueueIterator`:

```python
for batch_id, x in QueueIterator(batch_queue):
```

# Measuring

After stopping the previous training invocation and starting a new one, we can immediately see a good change in `nvidia-smi dmon`:

```
# gpu    sm   mem   enc   dec
# Idx     %     %     %     %
    0    97    44     0     0
    0    97    45     0     0
    0    99    50     0     0
    0    97    49     0     0
    0    97    49     0     0
    0    97    49     0     0
    0    97    47     0     0
    0    97    48     0     0
    0    99    50     0     0
    0    97    55     0     0
    0    97    56     0     0
```

The GPU is staying fully utilized. Here's the master process:

<a href="${r '/images/optimizing-machine-learning/neural-style-tmpq-master.svg'}">
<img class='centered' src="${r '/images/optimizing-machine-learning/neural-style-tmpq-master.svg'}"/>
</a>

What was previously the largest chunk of work in the flame graph is now the small peak on the left. Training logic dominates the graph, instead of image decoding, and `namedtuple` looks like increasingly low-hanging fruit at 13% of samples...

And the worker that loads training images:

<a href="${r '/images/optimizing-machine-learning/neural-style-tmpq-worker.svg'}">
<img class='centered' src="${r '/images/optimizing-machine-learning/neural-style-tmpq-worker.svg'}"/>
</a>

It's spending most of its time doing image decoding. There's some CPU spent in Python's Queue implementation, but the worker sits at about 40% CPU usage total anyway, so the inter-process communication isn't a major bottleneck in this case. 

More importantly, for our "time until 20,000 iterations" measurement, that improves from 21m40s to 17m9s, or about a 26% improvement in iterations/sec (15.4 to 19.4). Not bad for just a few lines of straightforward code.
