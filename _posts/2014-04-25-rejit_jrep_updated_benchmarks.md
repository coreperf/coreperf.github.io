---
layout: post
title:  "Updated jrep benchmarks"
date:   2014-04-25
author: Alexandre Rames
categories: technical
---


In the comments of the [LWN article][lwn article] about jrep, readers asked
about performance against other grep engines. Quick tests showed that `jrep` was
doing very well, but was not performing as well as `git grep` for
multi-threading.  So I applied a few improvements and fixes to the
multi-threading code, and updated the `-j` option to automatically select the
number of threads to use.
<br />
Below are updated results for the grepping benchmarks (recursive grep through
the Linux kernel 3.13.5 sources for different regexps, ran with
`<rejit>/tools/benchmarks/gjrep.py`).

The benchmarks are run with the engines:

    GNU grep version 2.18
    git version 1.8.5.2
    rejit commit b13708c133047d00366bb92e8402e74d5786bfec

on a MacBook Air (2012) with OSX 10.9.2:

    Model Identifier: MacBookAir5,2
    Processor Name: Intel Core i7
    Processor Speed:  2 GHz
    Number of Processors: 1
    Total Number of Cores:  2
    L2 Cache (per Core):  256 KB
    L3 Cache: 4 MB
    Memory: 8 GB

Note that this machine has many fewer cores than the one used for the LWN
article. I will re-run the benchmarks on a more powerful machine and post the
results.

Multi-threaded runs only report the `real` time, while single threaded runs
split the time into `sys` and `user`.

![benchmarks of simple regexps for grep, git grep, and jrep][simple regexps]

Jrep now benefits from multi-threading even for very simple regular expressions.
The `-j` option can safely be used to speed up the search.

![benchmarks of simple alternations for grep, git grep, and jrep][simple alt]
![benchmarks of simple regexps for grep, git grep, and jrep][complex regexps]

Grepping for `[a-z]{3,10}.*gnu.*[a-z]{3.10}` highlights the fact that `git grep` does not fast forward for
`gnu` (see [here][rejit fast forwarding] for more details).

![benchmarks of simple regexps for grep, git grep, and jrep][alt]

Alternations of 8 or more elements is still a weak point of rejit, but the
performance is still decent.



[lwn article]: http://lwn.net/Articles/589009/
[rejit fast forwarding]: /projects/rejit/introduction.html#fastforwarding_mechanism
[simple regexps]: /resources/posts/2014-04-25-rejit_jrep_updated_benchmarks/simple.png
[simple alt]: /resources/posts/2014-04-25-rejit_jrep_updated_benchmarks/simple_alt.png
[complex regexps]: /resources/posts/2014-04-25-rejit_jrep_updated_benchmarks/complex.png
[alt]: /resources/posts/2014-04-25-rejit_jrep_updated_benchmarks/alt.png
