---
title: Reading Large CSVs in Python Without Running Out of Memory
published: 2026-06-11T15:51:24.749Z
description: ''
updated: ''
tags:
  - Python
draft: false
pin: 0
toc: true
lang: ''
abbrlink: ''
---

# Introduction

I recently had a task at work where I needed to process a huge CSV file to generate a report summary.
This is something I'd done for my team countless times before — whip up a quick Python script,
read the file, crunch the numbers. But with this file being significantly larger than anything
I'd processed before, I wanted to see how my usual approach would behave. Out of curiosity,
I decided to benchmark a few different approaches to see exactly how they handle performance
and memory under pressure.

While I can't share the actual dataset due to data privacy, I wrote a short Python script to
generate a representative CSV of 5 million rows with four columns: an ID, a timestamp, a transaction amount, and a category.
The task is to caculate how much was spent for each category.
The generated file comes in at around 216MB, which is enough to make the differences between approaches very visible.
You can find the full script and follow along with the complete codebase in the accompanying
[GitHub Repository](https://github.com/dennis-nw/python-csv-reader-experiments).

## Approach 1: The Naive Pandas Baseline

[View full benchmark script on GitHub](https://github.com/dennis-nw/python-csv-reader-experiments/blob/main/read_csv_naive.py)

In this approach, we take the most straightforward path: loading the entire dataset directly into a Pandas DataFrame all at once.

```python
import pandas as pd

df = pd.read_csv(FILE)

result = df.groupby("category")["amount"].sum()
```

### The Results

Using `tracemalloc` to track the footprint of our ~216MB data file, here's how the naive approach performed:

| Metric | Value |
| :--- | ---: |
| Wall Time | 3.51s |
| Peak Memory | 638.9MB |

Notice something alarming here? Our raw CSV file is only about 216MB on disk,
but loading it into Pandas consumed 638.9 MB of RAM. That's nearly three times the file size!

This happens because Pandas converts raw text into rich Python objects and NumPy data types,
which carry a lot of memory overhead. While 640MB is fine for a modern computer, this exact behavior
is a ticking time bomb. If that file scales up to 3GB in production, your script will easily demand around
9GB of RAM, likely triggering an `out-of-memory` crash on your server.

## Approach 2: Native CSV Module

[View full benchmark script on GitHub](https://github.com/dennis-nw/python-csv-reader-experiments/blob/main/read_csv_streaming.py)

Instead of pulling the entire dataset into an in-memory data structure, this approach leverages
Python’s built-in `csv` module. The native reader acts as an iterator, parsing and yielding exactly
one row at a time. This keeps our memory footprint completely flat, regardless of whether the file is 200MB or 200GB.
I used `DictReader` specifically, which reads the header row automatically and gives you named
column access on each row, making the code clean without any extra setup.

```python
import csv

# Streaming the file line-by-line
with open(FILE, "r") as file:
  reader = csv.DictReader(f)
  for row in reader:
    # perform processing/computations
```

### The Results

| Metric | Value |
| :--- | ---: |
| Wall Time | 21.36s |
| Peak Memory | 0.1MB |

Look at that memory footprint! We dropped from 638.9 MB down to a mere 0.1 MB (100 KB).
Because Python only keeps a single row in flight at any given time, the memory overhead is virtually non-existent.

However, notice the massive trade-off: **Time**. The execution time jumped from 3.51 seconds to 21.36 seconds.
Because we are looping through 20 million lines line-by-line in pure Python, we lose the blazing-fast,
vectorized C-optimizations that Pandas uses under the hood.

## Approach 3: Pandas with Chunking

[View full benchmark script on GitHub](https://github.com/dennis-nw/python-csv-reader-experiments/blob/main/read_csv_pandas_chunks.p)

What if we want the speed of Pandas' vectorized operations but can't afford to load the
entire file at once? Enter chunking. By passing the chunksize parameter to `pd.read_csv()`,
Pandas splits the massive file into manageable batches. Instead of returning a single giant DataFrame,
it returns an iterable object. Each iteration yields a smaller DataFrame of a fixed size,
allowing us to process data efficiently without blowing up our RAM.

```python
import pandas as pd

# Chunking the file
chunk_size = 100_000
for chunk in pd.read_csv(FILE, chunksize=chunk_size):
  # perform processing/computations on each chunk
```

### The Results

| Metric | Value |
| :--- | ---: |
| Wall Time | 3.46s |
| Peak Memory | 22.4 MB |

By batching the data, we got the absolute best of both worlds. The execution time actually edged
out the naive approach because Python didn't have to struggle to allocate one massive, contiguous block of memory all at once.
At the same time, our peak memory stayed locked at a safe, predictable ceiling of just 22.4 MB.

Before wrapping up, I wanted to see what would happen if I dialed down the chunk size even further.
I reran the exact same Pandas chunking experiment, but cut the batch size in half from 100,000 rows to 50,000 rows.

Here is what happened:

| Metric | Value |
| :--- | ---: |
| Wall Time | 3.82s |
| Peak Memory | 11.6 MB |

This demonstrates the linear relationship between chunksize and memory ceilings. Therefore, if you're running
your program on a machine with little RAM, you can confidently lower the `chunksize` to stay under the memory limits
with minimal performance penalty.

## Conclusion

Here's a summary of how the different approaches we talked about perform:

| Approach | Wall Time | Peak Memory | Notes |
| :--- | ---: | ---: | :--- |
| **Pandas (Naive)** | `3.51s` | `638.9 MB` | ❌ **Not safe.** High risk of a catastrophic `MemoryError` on large files. |
| **Native CSV** | `21.36s` | `0.1 MB` | ⚠️ **Safe but slow.** Keeps memory flat, but loop overhead is a bottleneck. |
| **Pandas with Chunks** | **`3.46s`** | **`22.4 MB`** |  **The Sweet Spot.** Optimal balance of vectorized speed and safety. |
| **Pandas with Chunks (50K)** | `3.82s` | **`11.6 MB`** |  **Ultra Lean.** Perfect for tight, resource-constrained environments. |

If you need raw pure-Python logic with zero dependencies and speed is not a huge concern,
the native CSV reader is incredibly safe. But if you want to keep the blazing-fast speed of data science
tools without the fear of a `MemoryError`, Pandas chunksize is your best friend.
It's a simple change that can be the difference between a script that runs and one that doesn't.

A library worth mentioning here is [Polars](https://pola.rs/).
It has been gaining massive traction as a modern, performant alternative to Pandas.
While I haven't benchmarked it for this specific experiment yet, it’s high on my radar.
When I dive into it, I’ll be sure to do a deep-dive write-up right here to see how
it stacks up against our Pandas with chunking results.

Happy Building!
