---
layout: post
title: "Basic MAF parsing"
date: 2012-05-21 13:43
comments: true
categories: 
---

Today is the official start of coding for GSoC. Following my
[initial plan](https://github.com/csw/bioruby-maf/wiki/Original-proposal),
I'm going to begin by implementing the basics of MAF parsing in
Ruby. I've already got Cucumber features defined for
[basic MAF parsing](https://github.com/csw/bioruby-maf/blob/master/features/maf-parsing.feature)
and
[conversion to FASTA](https://github.com/csw/bioruby-maf/blob/master/features/maf-to-fasta.feature),
along with
[reference data](https://github.com/csw/bioruby-maf/tree/master/test/data)
as processed by bx-python.

This should be fairly straightforward to implement, but I'm going to
wait until I've got this basic functionality running before I define
anything else in detail. I'm still weighing whether it will make sense
to have a 'raw' representation of sequences and alignment blocks
rather than just using (and presumably subclassing) the
[bio-alignment](https://github.com/pjotrp/bioruby-alignment)
representations. 

Once basic MAF parsing works, my next area to focus on will be indexed
access. Many use cases for MAF involve pulling a few alignments out of
very large data files, rather than batch-processing the whole
file. I'll be focusing on the indexed-access API at first, and
building a simple interim indexing scheme similar to that used by
Biopython, probably using SQLite in a similar way. In developing the
API, I'll study those provided by bx-python and Biopython, the two
other MAF implementations providing persistent indexing.

Ultimately, I plan to revisit my actual indexing method, and
potentially implement support for bx-python's
[interval index files](https://bitbucket.org/james_taylor/bx-python/src/07aca5a9f6fc/lib/bx/interval_index_file.py). I'll
also take a careful look at other database alternatives such as
Berkeley DB and Tokyo Cabinet.

P.S. For my next blog post, I think I will try using Markdown's
[reference-style links](http://daringfireball.net/projects/markdown/syntax#link),
since the raw source for these posts is getting unwieldy with inline
links.
