---
layout: post
title: "Parallel I/O"
date: 2012-06-21 00:47
comments: true
categories:
---

Some applications of a MAF parser call for sequential processing of an
entire MAF file, but in many cases random access is necessary. I found
that a single-threaded parsing approach worked tolerably well for
sequential access, with OS-level readahead helping, but that
single-threaded performance with random access was very poor
indeed. With a test scenario, I was only able to parse 10.5 MB/s from
a SATA hard disk, and 34.3 MB/s from a MacBook Air SSD.

My next step was to parallelize random I/O for index-based
queries. With Ruby 1.9.3, this was pointless, since its GIL prevents
any useful parallelism, but with JRuby there was ample scope for
improvement. In the end, I implemented several parallel I/O
subsystems: one with a pool of independent threads reading and parsing
groups of blocks, one with separate pools of reader and parser
threads, and one using an [adaptive parallel I/O subsystem][] modeled
after Dmitry Vyukov's [Wide Finder entry][].

[adaptive parallel I/O subsystem]: https://github.com/csw/bioruby-maf/blob/0b7210716be6ce3eb47c167db2fecf6f5539a240/lib/bio/maf/parallel_io.rb
[Wide Finder entry]: http://www.1024cores.net/home/scalable-architecture/wide-finder-2

As test data for this, I used chr22.maf from the [hg18][] data set and
a set of 10,000 randomly generated genomic intervals from its
reference sequence, along with an index for the MAF file. I tested
performance with all three I/O subsystems on hard disk and SSD
storage, and of course with various numbers of threads, queue sizes,
and so forth.

[hg18]: http://hgdownload.cse.ucsc.edu/goldenPath/hg18/multiz28way/maf/

To my surprise, I found that my first implementation, with independent
worker threads reading and immediately parsing MAF blocks,
outperformed the others. Quite possibly this is because the data being
parsed has just been read into cache by the same thread. It is
possible that the adaptive implementation could be competitive with
it, but it was challenging to tune it properly for this workload in a
way that worked with slow and fast storage alike. Further work on this
might be promising.

In the end, the first implementation delivered a maximum of 26 MB/s
from hard disk, and 136 MB/s from SSD; the implementation with
separate reader and parser threads seemed to top out at 24 and 104
MB/s respectively.

Almost as striking, however, was the difference between a single run
and the third or fourth run repeated back-to-back in the same JVM
instance. In some cases, performance almost doubled, from 87 to 136
MB/s in a representative case. File system caching was ruled out, but
intensive HotSpot compiler activity was observed, so I would attribute
this to the combined optimization activity of the JRuby and HotSpot
JIT compilers.

## Index scanning

An unexpected bottleneck was observed: at peak performance,
single-threaded scanning of the 16 MB index to select the blocks to
parse takes 6.9 seconds, nearly as long as it took to parse the 869 MB
of MAF data. [Optimizing the match test][] used produced a significant
speedup, but past that, no optimization attempts have had much
success.

[Optimizing the match test]: https://github.com/csw/bioruby-maf/commit/4deee8548bfbc2899baae0c1f81893f64a10b5eb

Parallelizing the index scan showed a modest improvement with two
threads, topping out around 5.2 seconds; more than two threads have
resulted in negligible improvements. Currently, the two-thread
parallel scan is the default under JRuby.

Starting the scan [at the first block of interest][] rather than at
the start of the bin added noticeable complexity for no detectable
performance gain. [Running the index scan in parallel][] with the
block parsing seemed like a reasonable approach, but turned out to be
slower than performing the two operations sequentially.

[at the first block of interest]: https://github.com/csw/bioruby-maf/commit/50bbc0fdcab6906a80f00cd42bd4928aaec20403
[Running the index scan in parallel]: https://github.com/csw/bioruby-maf/commit/624447dd9d61f9b9b5acebfe669e10a7aae5596d

## JRuby and java.util.concurrent

A significant benefit of using JRuby is having access to the
[java.util.concurrent][] library. Its blocking queues made parallel
parsing simple to implement, and parallel index scans were even easier
with the ExecutorCompletionService. Since JRuby objects are JVM
objects like any other, and JRuby threads are implemented atop JVM
threads, it was perfectly easy to use these.

[java.util.concurrent]: http://docs.oracle.com/javase/7/docs/api/java/util/concurrent/package-summary.html

