---
layout: post
title: "First milestone"
date: 2012-05-25 14:13
comments: true
categories:
---

It's the end of my first week of actual development work, and I'm
happy with my progress. My [first milestone][], for batch MAF
conversion to [FASTA][], is complete. I started with a simple
line-based parser, and then spent some time developing a faster one;
my MAF parser now appears to be quite competitive in performance. I
can now convert simple MAF files to FASTA with output matching that
from the [Galaxy][] tools. And I'm fairly confident in its
correctness, with RSpec [unit tests][] and Cucumber [features][]
specifying its behavior relatively completely.

[first milestone]: https://github.com/csw/bioruby-maf/issues?milestone=1&state=closed
[FASTA]: http://en.wikipedia.org/wiki/FASTA_format
[Galaxy]: http://galaxy.psu.edu/
[unit tests]: https://github.com/csw/bioruby-maf/blob/master/spec/bio/maf/parser_spec.rb
[features]: https://github.com/csw/bioruby-maf/blob/master/features/maf-parsing.feature

## Parser development

My initial approach was to simply process the file line-by-line,
calling [IO#readline][] each time. This turned out to be a little more
cumbersome than I expected. Also, there was a [discussion][] on the
Biopython list of the usefulness of [BGZF][] blocked compression for
MAF files, which would not fit well with such a linear line-oriented
approach. Inspired by Dmitry Vyukov's [Wide Finder 2 entry][], I wrote
a [new parser][] which reads the file in 8 MB chunks. It immediately
locates the start of the last MAF alignment block in that chunk, and
then proceeds to scan through the chunk, parsing an alignment block at
a time, until the last block is reached. When it reaches the last
block, it [joins the block fragments][#parse_block] at the end of one
chunk and the beginning of the next chunk.

[IO#readline]: http://www.ruby-doc.org/core-1.9.3/IO.html#method-i-readline
[discussion]: http://lists.open-bio.org/pipermail/biopython-dev/2012-April/009561.html
[BGZF]: http://blastedbio.blogspot.com/2011/11/bgzf-blocked-bigger-better-gzip.html
[Wide Finder 2 entry]: http://www.1024cores.net/home/scalable-architecture/wide-finder-2
[new parser]: https://github.com/csw/bioruby-maf/blob/d424245c047e3939d1e42b97c002c02fc8b578ac/lib/bio/maf/parser.rb#L61
[#parse_block]: https://github.com/csw/bioruby-maf/blob/d424245c047e3939d1e42b97c002c02fc8b578ac/lib/bio/maf/parser.rb#L130

For separating the alignment blocks within each chunk, using a
StringScanner and regexes worked well, but for parsing the individual
alignment blocks, using String#split to split the block into lines,
dispatching on the first character with a case statement and plain
string comparisons, and splitting lines into fields turned out to
perform substantially better. I suspect the overhead of setting up a
regex matching operation is the reason for this; profiling indicated
that it was spending considerable time in regex operations, at any rate.

After this optimization, the chunked parser was able to parse a 315
MB file and count its MAF blocks in 10.1 s, compared to 16.0 s for the
earlier line-based parser and 22.7 s for the parser from Galaxy's
bx-python library. This seems like very respectable performance.

Also, the upcoming JRuby 1.7 on Java 7 appears to have excellent
performance with the chunked parser, perhaps due to the new
[invokedynamic support][]. After warming up the JVM, it averaged 16
&mu;s per alignment block parsed, compared to 25 &mu;s for Ruby
1.9.3. See the [performance page][] on the project wiki for details.

[invokedynamic support]: http://blog.headius.com/2011/08/jruby-and-java-7-what-to-expect.html
[performance page]: https://github.com/csw/bioruby-maf/wiki/Performance

## Behavior-Driven Development

At the beginning of this week, I was not at all clear on when it would
make sense to use Cucumber and when to use RSpec. After writing a
[Cucumber feature][] to ensure that sequence data was exposed
properly, it didn't seem worthwhile to duplicate that in
RSpec. However, the chunked parser was much harder to get right than
the original line-based parser, and RSpec was an invaluable tool for
that. 

[Cucumber feature]: https://github.com/csw/bioruby-maf/blob/d77ef2e133edf449a4ac3b18ff7d21052c0fc14a/features/maf-parsing.feature

For most of the bugs I found, I developed an RSpec specification to
call for the correct behavior. Some of these could go into Cucumber
just as well, such as the one stipulating that a certain alignment
block should have ten sequences in it. Most, though, are about
low-level details such as correct parsing of
[sequence lines split across chunk boundaries][split s lines].

[split s lines]: https://github.com/csw/bioruby-maf/blob/b3b9e060c6ba40bf17930eb6564615eb3b5b31bf/spec/bio/maf/parser_spec.rb#L138

In some cases it was challenging to follow the BDD principle of
writing only the necessary code to make the next test pass. When I
started on the chunked parser, I had a very clear idea of how it
needed to work, and next to no idea how to specify that in terms of
tests. So I just started writing code and relying on the existing
parser tests. However, the set of RSpec tests I ended up with seems to
specify the chunked parser's behavior reasonably well; they specify
where the last block in a chunk should be detected, how many blocks
should be parsed in a file if chunk fragments are being joined
properly, what happens if a sequence line is split across a chunk
boundary, and so on.

This gives me a concrete idea of the kind of tests I might want to
write to specify something like this next time. We'll see how well I
can apply this next week, when I start working on indexed access.

## Tools

I knew that it was possible to set up [continuous integration][] as a
hosted service with [Travis-CI][], but it was a pleasant surprise to
find that I could also have [documentation][] generated the same
way. It would be nice to have Travis-CI do hosted coverage reporting
as well, but setting up [simplecov][] was very easy.

[continuous integration]: http://travis-ci.org/#!/csw/bioruby-maf
[Travis-CI]: http://travis-ci.org/
[documentation]: http://rubydoc.info/github/csw/bioruby-maf/
[simplecov]: https://github.com/colszowka/simplecov

I've been doing my development work with Emacs and using
[rspec-mode][], [cucumber.el][], and [Magit][]. These have all been
better than nothing, but also somewhat frustrating; I've already sent
in one [pull request][cucumber-21] for cucumber.el, and over the next
month or two I may explore some more radical ideas to make rspec-mode
and cucumber.el more useful. I've been gradually learning how to use
Magit, although I usually end up doing my more complex Git operations
from the command line.

[rspec-mode]: http://barelyenough.org/projects/rspec-mode/
[cucumber.el]: https://github.com/michaelklishin/cucumber.el
[Magit]: https://github.com/magit/magit
[cucumber-21]: https://github.com/michaelklishin/cucumber.el/pull/21
