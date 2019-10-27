---
layout: post
status: publish
published: true
title: Parallelizing single-threaded batch jobs using Python's multiprocessing library.
author: Travis Mick
date: 2016-12-23
---

Suppose you have to run some program with 100 different sets of parameters. You might automate this job using a bash script like this:

```bash
ARGS=("-foo 123" "-bar 456" "-baz 789")
for a in "${ARGS[@]}"; do
  my-program $a
done
```

The problem with this type of construction in bash is that only one process will run at a time. If your program isn't already parallel, you can speed up execution by running multiple jobs at a time. This isn't easy in bash, but fortunately Python's `multiprocessing` library makes it quite simple.

<!-- more -->

One of the most powerful features in `multiprocessing` is the `Pool`. You specify the number of concurrent processes you want, a function representing the entry point of the process, and a list of inputs you need evaluated. The inputs are then mapped onto the processes in the `Pool`, one batch at a time.

You can combine this feature with a `subprocess` call to invoke an external program. For example:

```python
import subprocess, multiprocessing, functools

ARGS = ["-foo 123", "-bar 456", "-baz 789"]
NUM_CORES = 4

shell = functools.partial(subprocess.call, shell=True)
pool = multiprocessing.Pool(NUM_CORES)

pool.map(shell, ["my-program %s" % a for a in ARGS])
```

To break it down, we have:
* Translated the `ARGS` array from the bash script to Python syntax,
* Used `functools.partial` to give us a helper function that invokes `subprocess.call(..., shell=True)`,
* Created a pool of `NUM_CORES` processes,
* Used list comprehension to prepend the program name to each element in the `ARGS` list,
* And finally mapped the resulting list onto the process pool.

The result is that your program will be executed with each specified set of arguments, parallelized over `NUM_CORES` processes. There's only two more lines of code than the bash script, but the performance benefit can be manyfold.

