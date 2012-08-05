---
layout: post
title: "BGZF and testing"
date: 2012-08-05 18:16
comments: true
categories:
---

This post was originally going to be the bio-maf 1.0 release
announcement. I've added a new command-line [maf_extract(1)][] tool as
well as support for [BGZF][] compression of MAF
files, which allows random access to compressed files, saving disk
space and also I/O bandwidth. However, in the process of testing my
tools against UCSC's [multiz46way][] set of MAF files, I've discovered
a number of issues requiring a bit more work.

## maf_extract

With some of the other MAF libraries such as bx-python, there are a
variety of command-line tools for extracting MAF alignments, each
offering a different query or post-processing mode. I created
[maf_extract(1)][] to serve as a generic random-access MAF extraction
tool, exposing most of the capabilities of bio-maf with a simple
command-line interface. The man page has plenty of examples, but a few
representative invocations are:

    $ maf_extract -d test/data --interval mm8.chr7:80082592-80082766 --only-species hg18,mm8,rheMac2

    $ maf_extract -d test/data --mode slice --interval mm8.chr7:80082592-80082766

I have already found this useful for ad hoc data extraction from MAF
files, without writing any code at all.

## BGZF

[BGZF][] is the Blocked GZip Format, defined in the
[SAM/BAM specification][]. Blocks of up to 64 KB of data are
separately compressed with gzip and concatenated into a single
file. For random access, data is addressed with a *virtual offset*
scheme: a virtual offset is a 64-bit quantity where the first 48 bits
give the starting offset of the BGZF block within the file, and the
last 16 bits give the offset within the decompressed data. Thus, given
a virtual offset, one can read the relevant BGZF block, decompress it,
and seek to the appropriate position within the decompressed data.

[Artem Tarasov](https://github.com/lomereiter) wrote a Ruby
implementation of basic BGZF compression and decompression, which I
forked and extended for my needs as [bio-bgzf][]. This offers
rudimentary [IO][]-like interfaces for reading and writing BGZF files.

To use BGZF optimally with bio-maf, I've added the [maf_bgzip(1)][]
tool to compress (or recompress, in the case of gzipped files) MAF
files, optionally building indexes on them in the process. The idea is
that this can be used as a 'intake' tool to prepare a set of MAF files
such as those from UCSC for random-access querying.

## Stress testing

In light of that concept, I decided to recompress and index the entire
set of MAF files from UCSC. It immediately became clear that I should
have taken that step earlier, and not just tested with selected
subsets of the 31 GB (compressed!) data set.

I discovered a memory leak ultimately traceable to JRuby's very
aggressive sharing of backing store for strings. This is a great
feature, until you write code that holds references to occasional
small strings parsed from a multi-GB file. I wasn't happy with my
[fix](https://github.com/csw/bioruby-maf/commit/891dbfa8d4d7885f23ff16f16f5085d2a368eb48);
I suspect there is a better alternative for obtaining a totally
independent copy of a string than `"" << string`, but if there is, I
haven't found it yet.

I also found, to my surprise, that doing gzip decompression in the
parser thread performs substantially better than opening a pipe to
`gzip -dc`, cutting elapsed time by about 25%. I suspect I/O buffering
may be the culprit, but it didn't seem worth digging into further.

The slightly nonstandard `upstream*.maf` files from UCSC led me to
[add](https://github.com/csw/bioruby-maf/commit/20b146173e55b2825a8018278133c61193ddd490)
a `:strict` parser option, defaulting to off, since the parser was not
expecting their `r` lines. Where possible, I'm adding non-strict
parsing.

The performance of [maf_bgzip(1)][] on large files turned out to be
poor, but with a
[few](https://github.com/csw/bioruby-maf/commit/d96288413a782af00877f17da76ce2a9600b7cc9)
[improvements](https://github.com/csw/bioruby-maf/commit/28a55a5c9a0d096de253ec885d35d12392436e13)
I managed to double the speed of recompressing and indexing files.

The next sticky issue I need to confront is the need for extended UCSC
[bin indexes][]; at least one sequence in the UCSC alignments has an
genomic position beyond 2<sup>29</sup>, causing this
[unfortunate surprise](https://github.com/csw/bioruby-maf/issues/106).

## Next steps

After getting a more thoroughly tested release out the door, I'm going
to focus on integrating my tools into the [Galaxy Tool Shed][], to
make it even easier for people to use them. I'll post more on that
once I've gotten them integrated with a local Galaxy instance.

[BGZF]: http://blastedbio.blogspot.com/2011/11/bgzf-blocked-bigger-better-gzip.html
[maf_extract(1)]: http://csw.github.com/bioruby-maf/man/maf_extract.1.html
[multiz46way]: http://hgdownload.cse.ucsc.edu/goldenpath/hg19/multiz46way/
[SAM/BAM specification]: http://samtools.sourceforge.net/SAM1.pdf
[bio-bgzf]: https://github.com/csw/bioruby-bgzf
[IO]: http://www.ruby-doc.org/core-1.9.3/IO.html
[maf_bgzip(1)]: http://csw.github.com/bioruby-maf/man/maf_bgzip.1.html
[bin indexes]: http://genomewiki.ucsc.edu/index.php/Bin_indexing_system
[Galaxy Tool Shed]: http://wiki.g2.bx.psu.edu/Tool%20Shed
