---
layout: post
title: "Kyoto Cabinet support for JRuby"
date: 2012-07-02 16:01
comments: true
categories:
---

[Kyoto Cabinet][] is a useful database, with a convenient
[Ruby binding][]; unfortunately, this uses Ruby's C extension API and
so does not work with JRuby. Since JRuby has a considerable
performance advantage over MRI for my [MAF parser][], I needed a way
of using Kyoto Cabinet from JRuby. I have released the
[kyotocabinet-java] gem to provide this. It is not complete, but it
wraps enough of the API for my needs, and the rest should be simple to
cover.

[Kyoto Cabinet]: http://fallabs.com/kyotocabinet/
[Ruby binding]: http://fallabs.com/kyotocabinet/rubydoc/
[kyotocabinet-java]: https://github.com/csw/kyotocabinet-java
[MAF parser]: https://github.com/csw/bioruby-maf

This gem contains a patched copy of the Kyoto Cabinet
[Java library][], along with a thin Ruby adaptation layer to provide
the usual Ruby API. Since the underlying Java library relies on native
code integrated with JNI, installing the gem compiles the native
library and associated Java code, and thus requires a JDK and working
C++ compiler as well as a Kyoto Cabinet installation.

[Java library]: http://fallabs.com/kyotocabinet/javapkg/

Mapping the Java API onto the Ruby API is generally quite simple.  The
chief issue is ensuring that Ruby strings are converted to Java byte
arrays rather then Java strings; the Java string versions of the API
methods fail completely with binary data. Also, the Java API lacks the
Ruby API's optional parameters. For instance, here is [DB#get][]:

[DB#get]: https://github.com/csw/kyotocabinet-java/blob/c51e23a4fee077229b0ffdf4b66d323b33635704/lib/kyotocabinet.rb#L105

``` ruby
    alias_method :_get, :get
    def get(key)
      ret_bytes(self._get(key.to_java_bytes))
    end
```

A few methods needed a tiny bit more work, such as `DB#set_bulk` and
`DB#cursor_process`, but as you can see from [kyotocabinet.rb][], it
wasn't much.

[kyotocabinet.rb]: https://github.com/csw/kyotocabinet-java/blob/master/lib/kyotocabinet.rb

Using this and the `kyotocabinet-ruby` library interchangeably for MRI
and JRuby support in a Bundler application is straightforward:

``` ruby
    gem "kyotocabinet-ruby", "~> 1.27.1", :platforms => [:mri, :rbx]
    gem "kyotocabinet-java", "~> 0.2.0", :platforms => :jruby
```

Unfortunately, if you are developing a gem as I am with
[bioruby-maf][], things are not quite so simple. While Bundler makes
it easy to require different gems depending on one's current Ruby
platform, [Gem::Specification][] provides no such feature. I ended up
making a [gemspec][] that builds two different versions of the gem: a
`java` platform variant that depends on `kyotocabinet-java`, and a
'normal' gem with no platform specified that depends on
`kyotocabinet-ruby`. This is a bit cumbersome; it requires running
`gem build` under both JRuby and MRI to build both versions of the
gem, and meant that I had to abandon use of Jeweler. Nonetheless, it
works; under JRuby, `gem install bio-maf` will pull in the Java
variant and `kyotocabinet-java`, while under MRI it will pull in the
'normal' gem and `kyotocabinet-ruby`. The relevant part of the gemspec
is:

[bioruby-maf]: https://github.com/csw/bioruby-maf
[Gem::Specification]: http://docs.rubygems.org/read/chapter/20
[gemspec]: https://github.com/csw/bioruby-maf/blob/eda7485e61aba4978ba16a6abcdc95b8094a722d/bio-maf.gemspec#L102

``` ruby
    if RUBY_PLATFORM == 'java'
      s.platform = 'java'
    end
    [...]
    if RUBY_PLATFORM == 'java'
      s.add_runtime_dependency('kyotocabinet-java', ["~> 0.2.0"])
    else    
      s.add_runtime_dependency('kyotocabinet-ruby', ["~> 1.27.1"])
    end
```

Ultimately, it might be worth contacting the maintainer of the
`kyotocabinet-ruby` gem and seeing about packaging this as a platform
variant, to sidestep all this. For the time being, though, this works.

A note on performance: I found JRuby's ObjectProxyCache to be a major
performance bottleneck, especially for multithreaded access to Kyoto
Cabinet. Starting JRuby with the `-Xji.objectProxyCache=false` option
gave about a 2.5x speedup for Kyoto Cabinet searches. If the rest of
your JRuby code works with this option (which will be the default in
JRuby 2.0, anyway), I highly recommend using it.
