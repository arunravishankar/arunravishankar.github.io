---
category: Setup-Notes
---
Whenever I am testing a block of code that I plan to implement over a large dataset, I first test it with a small sample. I've found the following code snippet from [Huy Nguyen](https://www.huyng.com/posts/python-performance-analysis) to be invaluable for this task.

Below, the code snippet, which I have saved as timer.py:

```python
import time

class Timer(object):
    def __init__(self, verbose=False):
        self.verbose = verbose

    def __enter__(self):
        self.start = time.time()
        return self

    def __exit__(self, *args):
        self.end = time.time()
        self.secs = self.end - self.start
        self.msecs = self.secs * 1000  # millisecs
        if self.verbose:
            print 'elapsed time: %f ms' % self.msecs
```

To use it, per Nguyen's instructions, just wrap the block of code that you want to test with Python's with keyword and call the newly created Timer context manager. Below, my use in timing how long it would take to import a CSV file:

```python
with Timer() as t:
  csv_list = FldrLoopCSV(directory)
  LoadCSVs(dbfile, csv_list)

# Benchmarking
print "=> elasped time: %s s" % t.secs
```
