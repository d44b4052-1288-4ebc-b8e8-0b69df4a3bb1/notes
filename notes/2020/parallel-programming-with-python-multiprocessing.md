---
[meta]
title = "Parallel programming with python multiprocessing"
date = 2020-07-10
tags = ["python", "parallel", "multiprocessing"]
description = "A short introduction to python multiprocessing."
---
Any modern language has features to work with concurrency and parallelism, python
is no exception. In this short introduction I will show you how to use multiple cores
for data processing with python.

Let's start with a short example of multiprocessing:

    class Worker(Process):
        def run(self):
            pass
    
    worker_list = [Worker() for idx in range(cpu_count)]
    [worker.start() for worker in worker_list]
    [worker.join() for worker in worker_list]

Yes, with less than 10 lines of code you can take advantage of all cores available
in your computer. In a short breakdown what we have:

 - a class that inherit from multiprocessing.Process
 - a run() method override (the only thing required from the original abstraction)
 - a list of workers matching the number of cores in your machine.
 - a list of started workers
 - a list of finished workers joined back to the main process.

Let's move ahead and create something fancier. We are going to create a bunch of
workers to do sequence alignment, each worker will consume one record from a queue,
process it and pick another record. This will be repeated until the queue dry-off.

Let's start extending or Worker class, we need to include some logging (the print statements)
a infinity loop that reads from a queue and our expensive computation logic:


    REFERENCE = "GAGTTTGATCCTGGCTCAGATTGAACGCTGGCGGCATGCTTAACACATGCAAGTCGAACGGCAG"
    
    class Worker(Process):
        def __init__(self, name, queue):
            super().__init__()
            self.name = name
            self.queue = queue
    
        def run(self):
            print(f"Started {self.name}")
            count = 0
            while True:
                try:
                    sequence = self.queue.get(timeout=5)
                    self.compute(sequence)
                    count += 1
                except Empty:
                    break
            print(f"{self.name} done with {count} records")
    
        @staticmethod
        def compute(sequence):
            s1 = DNA(REFERENCE)
            s2 = DNA(sequence)
            global_pairwise_align_nucleotide(s1, s2, penalize_terminal_gaps=True)


We also need a function to populate our queue, for this short example it's going to create one thousand random sequences
with 64 base pairs:


    def load_record_into_queue(queue):
        for _ in range(1000):
            sequence = "".join(choice("ACTG") for _ in range(64))
            queue.put(sequence)


Next we need to glue everything together in the __main__:

    if __name__ == "__main__":
        worker_list = []
        queue = Queue()
        for idx in range(cpu_count()):
            worker = Worker(f"worker-{idx}", queue)
            worker_list.append(worker)
    
        [worker.start() for worker in worker_list]
        load_record_into_queue(queue)
        [worker.join() for worker in worker_list]

One important thing to notice is that the `load_record_into_queue` is call after we start the workers, this is important!
Since this function is running in the main process, if you call it before the workers start, it's going to block until 
everything is loaded in the queue, and if your function is actually populating the queue from a infinite stream of data 
your workers will never get the chance to start.

## Multiprocessing vs Threads

It's hard to tell when you should be using multiprocessing or multithreading. In theory it's simple: if you code is 
cpu bounded, it will likely benefit from multiprocessing, if your code is i/o bounded you may be better with threads.
Sadly real life is not so black and white as you may have a function that's 50/50. In general if you go with 
multiprocessing it's less likely that you will make a mistake. But you get other problems...

The number of process you can handle usually will be lower than the number of threads, as process are more expensive and will
eat more RAM. Other problem is your ability to share data, while threads will get you more options it also gives you more 
opportunities to shoot yourself in the foot. 

## A few considerations about computation

Not all computation are made equally. If you create two functions for two different computations, and those functions eat 
100% of a single core during the same amount of time, when you parallelize them, they will have different improvement over 
extra core additions.

Never expect that parallelism is going to give you linearity, usually the jump from one process to two process is where you get
the biggest improvement (maybe even cutting the execution time by half), but the 3rd core is going to cut less, and so on...
Most tools that implement some level of parallelism will have a plateau, at some point adding more cores will give the tool
a negligible gain in speed.

  

**That's it, the full version including all imports should look like this:**

    from _queue import Empty
    from multiprocessing import Process, Queue, cpu_count
    from random import choice
    from skbio import DNA
    from skbio.alignment import global_pairwise_align_nucleotide
    
    REFERENCE = "GAGTTTGATCCTGGCTCAGATTGAACGCTGGCGGCATGCTTAACACATGCAAGTCGAACGGCAG"
    
    
    class Worker(Process):
        def __init__(self, name, queue):
            super().__init__()
            self.name = name
            self.queue = queue
    
        def run(self):
            print(f"Started {self.name}")
            count = 0
            while True:
                try:
                    sequence = self.queue.get(timeout=5)
                    self.compute(sequence)
                    count += 1
                except Empty:
                    break
            print(f"{self.name} done with {count} records")
    
        @staticmethod
        def compute(sequence):
            s1 = DNA(REFERENCE)
            s2 = DNA(sequence)
            global_pairwise_align_nucleotide(s1, s2, penalize_terminal_gaps=True)
    
    
    def load_record_into_queue(queue):
        for _ in range(1000):
            sequence = "".join(choice("ACTG") for _ in range(64))
            queue.put(sequence)
    
    
    if __name__ == "__main__":
        worker_list = []
        queue = Queue()
        for idx in range(cpu_count()):
            worker = Worker(f"worker-{idx}", queue)
            worker_list.append(worker)
    
        [worker.start() for worker in worker_list]
        load_record_into_queue(queue)
        [worker.join() for worker in worker_list]