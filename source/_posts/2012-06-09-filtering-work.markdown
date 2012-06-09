---
layout: post
title: "filtering-work"
date: 2012-06-09 11:46
comments: true
categories:
---

I think I made good progress in the end this week, although it was a
bit frustrating along the way, with a few dead ends and some
challenges in figuring out the right approaches to things. My main
goal was to implement MAF file filtering on queries, as specified in a
[Cucumber feature][] and [Github issue][].

[Cucumber feature]: https://github.com/csw/bioruby-maf/blob/86e9f0fb30723a0df19f948b1aa9c9bd36def8f8/features/maf-querying.feature
[Github issue]: https://github.com/csw/bioruby-maf/issues/22

This is all working now, and I've managed to do all the alignment
block filtering using the index, to avoid having to parse blocks to
test whether they meet criteria for e.g. presence of species. I'm
storing things like the number of sequences, the text size, and a bit
vector of species present in the index, and checking those when
building the list of MAF blocks to read from disk. I think this is an
improvement over bx-python, which does this sort of thing by a
combination of post-parsing filtering and writing out modified MAF
files.

I tried using external libraries to provide somewhat nicer interfaces
to binary record access ([bindata][]) and bit vector manipulation
([bitstring][]); they are both nice and well-thought out libraries but
turned out to have awful performance, so I ended up ripping them both
out. (I got about a 120% performance boost when building indexes by
using raw Array#pack calls instead of bindata, and then about 100% by
using Integers and shifts instead of bitstring.)

[bindata]: http://rdoc.info/gems/bindata
[bitstring]: http://bitstring.rubyforge.org/rdoc/

I think in the future I will do microbenchmarks before integrating
things like these; I wouldn't have expected such a high performance
penalty for such conceptually simple operations.

Ultimately, I wrote my own little [library][] for building format strings,
after making a few arithmetic mistakes when calculating byte offsets
in my head. I also wrote a [bit of code][] to dump out binary indexes in
human-readable form, to help with debugging.

[library]: https://github.com/csw/bioruby-maf/blob/master/spec/bio/maf/struct_spec.rb
[bit of code]: https://github.com/csw/bioruby-maf/blob/7ebeeb03c7b3e9755978f4c4307312e13451b0e6/lib/bio/maf/index.rb#L177

The [API] for this filtering code could perhaps use some work. I'll
need to think about that further.

[API]: https://github.com/csw/bioruby-maf/blob/master/features/step_definitions/query_steps.rb

I also made some progress on getting Kyoto Cabinet and JRuby working
together, although it's not quite there yet. Along the way, I
discovered what looks like an interesting [JRuby bug].

[JRuby bug]: https://github.com/csw/bioruby-maf/issues/31

And I've got tests passing on Travis CI again, after adding a script
to build and install Kyoto Cabinet there.

After working through and thinking about things this week, and
particularly in light of Francesco's comments, I have a few questions
I need to look into about MAF and MAF workflows:

* Is it possible for alignment blocks to overlap, at least with
  respect to the reference sequence?
* Is it useful to build indexes on other sequences besides the
  reference sequence?
* What is the longest (in terms of text size, i.e. sequence data
  including dashes) alignment block I should be prepared to encounter?
  (That is, does its length need to be 32 bits? 16? 64?)
