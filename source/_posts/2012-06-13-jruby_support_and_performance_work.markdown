---
layout: post
title: "JRuby support and I/O performance"
date: 2012-06-13 22:54
comments: true
categories:
---

## I/O performance and the Magic Number

My MAF parser is designed to read data from disk in relatively large
chunks before parsing it, to minimize read(2) calls. I started out
using a chunk size of 8 MB, but using the [jvisualvm][] profiler with
JRuby, I observed that during index builds with `maf_index`, it was
spending about 37% of its time in
`org.jruby.util.io.ChannelDescriptor.read()` calls. (I had observed
MRI spending a similar amount of time in read calls as well.) Since
Mac OS X has DTrace, I used [iosnoop][] to watch the I/O activity at
the OS level, and observed that reads were actually being issued with
sizes of 128k:

[jvisualvm]: 
[iosnoop]: http://www.brendangregg.com/DTrace/iosnoop

```
  501 55424 R 1139635624 131072       java ??/maf/chr22.maf
  501 55424 R 1139635880 131072       java ??/maf/chr22.maf
  501 55424 R 1139636136 131072       java ??/maf/chr22.maf
  501 55424 R 1139636392 131072       java ??/maf/chr22.maf
  501 55424 R 1139636904 131072       java ??/maf/chr22.maf
```

Suspecting that merging the data from these read calls into an 8 MB
buffer might be a problem, I changed the chunk size constant
(`SEQ_CHUNK_SIZE`) to 128k to match, and saw the processing rate jump
from 19 MB/s to 30 MB/s, with only 2.4% of CPU time spent in
`ChannelDescriptor.read()`.

MRI 1.9.3 exhibited a similar improvement, from 18 to 24 MB/s.

It turns out, though, that 128k is not, as I guessed, a hard-coded
Ruby setting. I later ran the same test on my MacBook Air's built-in
SSD instead of a USB-attached SATA disk, and saw I/O sizes of 1048576
instead. Here, changing the chunk size to match showed no significant
improvement. Presumably this is due to the lower I/O latency of the
SSD.

Requiring users to divine the best I/O size for their environment to
achieve the best parsing performance seems a bit arcane; I wonder if
it might be worthwhile to try to automatically detect the
best-performing chunk size at runtime by defaulting to 128k, varying
it in both directions, and tracking average latencies.

## JRuby support with Kyoto Cabinet

Kyoto Cabinet has turned out to perform well and be easy to
use. However, while the parser code runs substantially faster under
JRuby, there is not a straightforward out-of-the-box way to use Kyoto
Cabinet with Ruby. The standard `kyotocabinet-ruby` gem uses the C
Extension API, so it isn't really practical to use with
JRuby. (Technically, JRuby has partial support for the C Extension
API, but at least on my machine, it's broken in several ways under
RVM, needs to be enabled with a specific runtime option, and is
generally frowned upon.) However, Kyoto Cabinet also has a Java API;
both it and the Ruby API are based on the same underlying C++ library,
so the APIs are quite similar. I wrote a small Ruby [wrapper][] around
the Java library, so it can be used under JRuby in an identical way to
the native Ruby library.

[wrapper]: https://github.com/csw/bioruby-maf/blob/8e90c5d5f58524052e5d19d9ea6956ec361bf85b/lib/kyotocabinet-java.rb

Unfortunately, this still requires users to build and install the
JNI-based [Java library][] for Kyoto Cabinet as well as Kyoto Cabinet
itself. This isn't ideal, but shouldn't be too difficult. I plan to
separate this out as its own gem.
