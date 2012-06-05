---
layout: post
title: "Indexed MAF access"
date: 2012-06-04 16:33
comments: true
categories:
---

My second week of working on this MAF library was more of a struggle
than the first week. I was implementing indexing of MAF files and
using those indexes to enable searching by genomic regions. In the
process of doing this, I ended up changing directions partway through,
but I eventually did implement the features I had planned to, and
ended up with a better implementation than I thought I would.

MAF files can be quite large, often hundreds of gigabytes. Frequently,
too, a user will want to extract a relatively small subset of that
data. Reading in and parsing an entire 200 GB file to find a few
alignments of interest is not a very useful approach; instead, we want
to build indexes to enable quick location of the alignments to parse.

This was, in fact, my [milestone][] for this week: to be able to build
an index for a MAF file, identify alignment blocks matching a region
of interest, and then selectively parse those blocks from the MAF
file. The two MAF implementations I'm aware of that use a similar
indexing scheme are [Biopython][] and [bx-python][]; Biopython uses the
SQLite embedded relational database to build its indexes, and
bx-python uses a custom [index file format][]. (I believe the UCSC
genome browser uses MySQL for indexing in general, but I'm not sure
whether this is applied to MAF files directly.)

[milestone]: https://github.com/csw/bioruby-maf/issues?milestone=2&state=closed
[Biopython]: https://github.com/polyatail/biopython/tree/alignio-maf
[bx-python]: https://bitbucket.org/james_taylor/bx-python/wiki/Home
[index file format]: https://bitbucket.org/james_taylor/bx-python/src/07aca5a9f6fc/lib/bx/interval_index_file.py

### SQLite vs. Kyoto Cabinet

The plan I started with was to use SQLite for indexing in a similar
way to Biopython, in order to bring up functional code with indexing
support as quickly as possible. I reasoned that being able to use SQL
for declarative queries would make this simpler than, say, using a
key-value store such as [Kyoto Cabinet][] or [Berkeley DB][], let
alone implementing a *sui generis* indexing system such as the one
bx-python uses.

[Kyoto Cabinet]: http://fallabs.com/kyotocabinet/
[Berkeley DB]: http://en.wikipedia.org/wiki/Berkeley_DB

My first task, though, was to understand the UCSC
[bin indexing system][], which is used by all MAF indexing
implementations. It was not particularly clear to me how it worked,
even after looking at several implementations of it, until I finally
read the [paper][] describing it and illustrating its hierarchical
nature. I still don't understand why Biopython [always searches][] bin
1, which was puzzling me for a while; I need to ask the author.

[bin indexing system]: http://genomewiki.ucsc.edu/index.php/Bin_indexing_system
[always searches]: https://github.com/polyatail/biopython/blob/alignio-maf/Bio/AlignIO/MafIO.py#L318
[paper]: http://genome.cshlp.org/content/12/6/996.full

I ran into a series of problems with using SQLite,
unfortunately. First, the standard `sqlite3` gem uses the C extension
API rather than FFI, and so does not work with JRuby. To use the same
code with the native and JDBC drivers, I needed to use the
unmaintained DBI/DBD library. (I subsequently learned that [Sequel][]
might have worked as well, but after I'd already decided to try
another tack.) The DBI interface also had the unfortunate feature of
returning all numeric values as strings, which was not what I would
hope for in performance-critical code.

[Sequel]: http://sequel.rubyforge.org/

Also, I discovered that SQLite does not have clustered indexes,
meaning that I needed to create a table just to hold the index data,
and then a separate index within SQLite for that data. This duplicates
the index data, which is also not ideal.

Most of all, though, SQL simply turned out to be too
cumbersome. Explicitly creating and indexing tables and checking for
their existence started consuming more of my effort than I would have
liked. Biopython's [implementation][] ended up running a SQL query for
each possible bin for each range of interest; I knew that it should be
possible to execute a search like this while only having to make a
single pass through the index data for each bin. It wasn't clear that
there was a good (and simple) way to do this in SQL, though.

[implementation]: https://github.com/polyatail/biopython/blob/alignio-maf/Bio/AlignIO/MafIO.py#L384

In frustration, I opened up the Kyoto Cabinet documentation, studied
the available features for prefix and range searches, and wrote out
the [algorithm][] I had in mind. My biggest decision point was how
exactly to store the index data. In the past, I've always favored
human-readable data representations; I've generally dealt with small
amounts of data in situations where being able to see the data
immediately with View Source or a packet trace is invaluable. Here,
though, after adding things up I realized I could use a binary
representation and use a mere 24 bytes per index record. This actually
may be the first time I've written something to output a binary data
representation, but I think it was well worth it.

[algorithm]: https://github.com/csw/bioruby-maf/blob/4b0fb7b97c6185d912515dbf1903c0f066d2b901/lib/bio/maf/index.rb#L122

Once I sat down to actually implement the Kyoto Cabinet version, I was
amazed how much simpler it was than the SQLite version. I could step
through the data just the way I wanted to, and do whatever computation
seemed appropriate on the way. I pulled together a running version
very quickly, and simply applied the same RSpec tests to both.

For both implementations of this indexing code, I found RSpec
invaluable. I made a fairly rigorous effort to implement this in
BDD/TDD style, writing [tests][] for each bit of behavior I could
think of to specify. This gave me a great deal of confidence in
optimizing and tweaking the index search code.

[tests]: https://github.com/csw/bioruby-maf/blob/4b0fb7b97c6185d912515dbf1903c0f066d2b901/spec/bio/maf/index_spec.rb

### Kyoto Cabinet results

I'm now very happy with the Kyoto Cabinet code; I also expect it to be
easily adapted to other key-value stores such as leveldb or Berkeley
DB.

Getting Kyoto Cabinet working on JRuby has posed a bit of a headache
too, unfortunately. Its Ruby library also uses the C extension API, so
I plan to use the Kyoto Cabinet Java library instead, with a thin
adaptation layer to present a compatible interface to the Ruby
library. This approach has been promising, except that a few of the
methods in the Java library seem to encounter string encoding problems
with binary data, returning garbage strings (probably containing
Unicode 'REPLACEMENT CHARACTER', I need to check) rather than the
expected data. Many of the Java methods have separate String and
byte[] versions, but not all. Ironically, the trickiest problems of
this kind are caused by `DB#match_prefix` calls in my unit tests,
rather than in the actual indexing code.

Another oddity has been that there don't appear to be any Ubuntu
packages in usable form for Kyoto Cabinet. This is strange, since it's
hardly the most obscure software I've wanted packages for. I don't
need these for development, but [Travis CI][] doesn't provide Kyoto
Cabinet, so I have to install it myself there to get CI working
again. Since I'm not about to teach myself Debian packaging just for
this, I've written a [horrible shell script][] instead for the time
being. (What would really be nice is an easy way to run user-provided
Chef recipes on Travis CI, but they don't appear to provide one.)

[Travis CI]: http://travis-ci.org/
[horrible shell script]: https://github.com/csw/bioruby-maf/blob/0c2eabdd5f2e2efe2c3fd3e9559e84bed6dd1dec/travis-ci/install_kc

### Random-access parsing

The trickiest part of the whole effort may have been figuring out how
to revise the parsing code to work in a truly random-access fashion,
and in a way that will be compatible with [BGZF][] compression later
on. What I ended up doing was extracting a separate [ChunkReader][]
class responsible for positioning within the file and reading
appropriately-sized chunks of it as requested while maintaining proper
chunk alignment. Together with other code to [merge requests][] for
adjoining or nearby alignment blocks, this keeps the basic parsing
logic intact but should avoid either excessive or excessively large
reads.

[BGZF]: http://blastedbio.blogspot.com/2011/11/bgzf-blocked-bigger-better-gzip.html
[ChunkReader]: https://github.com/csw/bioruby-maf/blob/4b0fb7b97c6185d912515dbf1903c0f066d2b901/lib/bio/maf/parser.rb#L57
[merge requests]: https://github.com/csw/bioruby-maf/blob/4b0fb7b97c6185d912515dbf1903c0f066d2b901/lib/bio/maf/parser.rb#L138

This also means that the indexing code simply provides the starting
offset of each block, and the parser is responsible for reading ahead
as appropriate. This should be perfectly compatible with a BGZF layer,
even if an alignment block spans the boundary between separate BGZF
blocks. (Alignment blocks, chunks, and now BGZF blocks... I may be
heading for a terminology problem here.)

### Next steps

This week, I'm focusing on providing additional logic for filtering
and querying for sequences and alignment blocks. I'm also working on a
few ideas to optimize index creation, and I hope to get my index code
running on JRuby soon.

