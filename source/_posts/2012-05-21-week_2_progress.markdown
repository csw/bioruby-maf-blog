---
layout: post
title: "GSoC Week 2"
date: 2012-05-21 11:13
comments: true
categories:
---

This was my second week of work on my GSoC project, and the last week
of the 'community bonding' period before the official start of
coding. A major focus of mine was BioRuby's
[phyloXML](http://phyloxml.org/) support; it uses libxml, which has
been causing unit test failures under JRuby. In the end, the best
course of action seemed to separate the phyloXML support as a separate
plugin, which I have done as the
[bio-phyloxml](https://github.com/csw/bioruby-phyloxml) gem. This will
remove BioRuby's dependency on XML libraries entirely and that JRuby
issue along with it. At the same time, users of the phyloXML code
should be able to continue using it with no substantive changes.

Separately, I began porting this phyloXML code to use
[Nokogiri](http://nokogiri.org/) instead of libxml-ruby, but ran into
difficulties with this effort. While it is possible, and the library
APIs are very similar, the code uses relatively low-level XML
processing APIs in ways that seem to be sensitive to subtle
differences in text node and namespace semantics between the two
libraries. Substantial restructuring of the code and the addition of
quite a few unit tests might be necessary to carry out such a port
with confidence that the resulting code would work well.

Also, someone else submitted a
[JRuby patch](https://github.com/jruby/jruby/pull/176) for
[JRUBY-6658](http://jira.codehaus.org/browse/JRUBY-6658), one of the
major causes of BioRuby's unit test failures with JRuby; once a fix is
integrated, we'll be close to having all the tests passing under
JRuby.

I identified another JRuby bug,
[JRUBY-6666](http://jira.codehaus.org/browse/JRUBY-6666), causing
several unit test failures. This one affects BioRuby's code for
running external commands, so it would be likely to be encountered in
production use. For this one, I also worked up a
[patch](https://github.com/jruby/jruby/pull/173).

I also spent some time preparing a performance testing environment,
for evaluating existing MAF implementations as well as my own. This
will be important, since I will be considering the use of an existing
C parser. I will also want to ensure that the performance of my code
is competitive with the alternatives. Lacking any hardware more
powerful than a MacBook Air, I am setting this up with Amazon EC2. To
simplify environment setup, I'll be using
[Chef](http://wiki.opscode.com/display/chef/Home). I've already set up
a [Chef repository](https://github.com/csw/chef-repo) with
configuration logic, and some
[rudimentary code](https://github.com/csw/ec2-launcher) to streamline
launching Ubuntu machines on EC2 and bootstrapping a Chef
environment. To save money, I plan to make use of
[EC2 Spot Instances](http://aws.amazon.com/ec2/spot-instances/), which
are perfect for instances that only need to run for a few hours for
batch tasks.
