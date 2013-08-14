---
layout: project
title: CorePerf - Projects - Rejit
project_name: Rejit
project_github_url: https://github.com/coreperf/rejit
---

Rejit is a prototype of a non-backtracking, just-in-time, SIMD-able regular
expression compiler.
<br />
You can find the sources [on Github][rejit github].  It is available under the
GPLv3 licence.

We are looking for sponsors to help take this project further. We can focus
effort on new features, specific use cases or optimisations, port to other
architectures, help integrate Rejit into your projects, etc.
[Contact us][email alexandre] to discuss more.

The article below introduces the mechanisms used in Rejit and benchmark results.
<br />Here are a few useful links:
* [Github project page][rejit github]
* [Documentation][rejit wiki], on the Rejit wiki
* [Benchmarks results](#benchmarks) (below on this page)

### Introduction

I started working on Rejit after having read Russ Cox's excellent
[articles][Russ Cox articles].
Reading these can help understand some technical parts of the article, but is
not required.
All you should need is to see how regular expressions can be represented by
automata.

<div class="img_container">
<img src="resources/images/automaton_simple.png"
alt="An automaton for the regular expression '(a0|a1)z'."
class="img_no_border"/>
<div class="caption">
Example automaton for the regular expression `(a0|a1)z`.
</div>
</div>
Although it should behave fairly well, Rejit is still a prototype.
Development was done on my free time, so the effort went mostly into what I
thought was fun!

This article introduces a some technical information about Rejit, and then
presents benchmarks results.

<div style="margin:10px 0 0 40px">
  <ol>
  <li> <a href="#mechanisms">Mechanisms</a></li>
    <ol style="margin-bottom:0">
      <li style="list-style:lower-latin"> <a href="#matching_mechanism">Matching mechanism</a></li>
      <li style="list-style:lower-latin"> <a href="#looking_for_partial_matches">Looking for partial matches</a></li>
      <li style="list-style:lower-latin"> <a href="#fastforwarding_mechanism">Fast-forwarding mechanism</a></li>
      <li style="list-style:lower-latin"> <a href="#jit_code">JIT code</a></li>
    </ol>
  <li> <a href="#features">Features</a></li>
  <li> <a href="#benchmarks">Benchmarks</a></li>
    <ol style="margin-bottom:0">
      <li style="list-style:lower-latin"> <a href="#single_threaded_recursive_grep_through_the_linux_kernel_sources">Single threaded recursive grep through the Linux kernel sources</a></li>
      <li style="list-style:lower-latin"> <a href="#dna_matching_benchmark">DNA matching benchmark</a></li>
      <li style="list-style:lower-latin"> <a href="#rejit_benchmarks">Rejit benchmarks</a></li>
    </ol>
  <li> <a href="#future_work">Future work</a></li>
  </ol>
</div>



#### A few definitions

To avoid confusion later here are definitions for a few expressions used in
this article.

* **NFA**: Nondeterministic finite automaton ([Wikipedia][Wikipedia NFA]).
* **Full matching**: Testing if a regular expression matches a whole string.
  <br />
  The regular expression `abc` full-matches the string `abc` (the match is - of
  course - `abc`).
  <br />
  The regular expression `abc` does **not** full-match the string `0_abc_9`.
* **Partial matching**: Testing if a regular expression matches a substring of
  the text.
  <br />
  Partial matching the regular expression `[0a]` in the string `0128abcd` yields
  two matches: `0`, and `a`.
  <br />Partial matching can be sub-divided into 2 questions:
  <ol><ol>
    <li> Existence: is there a partial match of the regexp in the text?</li>
    <li> Location: Is there a partial match and, if so, where is it?
         <br />
         This is harder than the 'existence' question.
         We are looking for the leftmost longest match. We must keep track of the
         boundaries of the match, and cannot exit as soon as any match is found.
    </li>
  </ol></ol>
  Note the meaning associated with 'partial matches'.
  Looking for a partial match of a regular expression in a text means searching
  for sub-strings of the text that match the full regular expression. It has
  nothing to do with matching part of the regular expression.




### Mechanisms

The following paragraphs give a short description of Rejit's two main
mechanisms.

1. The code generated for the **matching** mechanism executes a (non-backtracking)
NFA simulation to check if a regular expression matches a sequence of
characters.  Full matching only uses this mechanism.
2. The code generated for **fast-forwarding** speeds up search for partial matches by
efficiently searching for dominating sub-regexps.



#### Matching mechanism

Rejit generates code able to execute an NFA simulation for the regexp considered.

The simulation allows the automaton to be in multiple states simultaneously.  It
executes either until no nodes are set and there is no match, or until we reach
the end of the string, at which point there is a match iff an exit node is set.

See the following diagram for an illustration, and Russ Cox's
[article][Russ Cox article 1] for a more detailed explanation.
<br />

<div class="img_container">
<img src="resources/images/automaton_simu_simple.png"
alt="Automaton simulation for a full match."
class="img_no_border"/>
<div class="caption">
Automaton simulation for a full match.
<br />The red pipe '<span style="color:red">|</span>' indicates the current position of the
algorithm in the string.
</div>
</div>

Full matching only use this mechanism. Execution results either in an early
exit when there is no match, or goes through scanning until the end of the
string.


#### Looking for partial matches

Assuming we have a matching mechanism (for example the one described above), how
should we search for partial matches?
<br />
Executing the matching mechanism successively for each character of the text
until a match is found is a trivial but very slow implementation.

Considering that an automaton is used for matching, a better solution can be to
execute the automaton simulation while setting the entry node of the automaton
at every character of the text searched. When the execution reaches an exit
node, a match is found.
<br />
Finding the leftmost match can easily be implemented by using the offsets `start
of match - start of string` to indicate the state of the nodes.

Below is a diagram illustrating the process to match the regular expression
`a(ab|b)a` in the text `_aaba_` with this method.

<div class="img_container">
<img src="resources/images/automaton_simu_offsets.png"
alt="Automaton simulation with nodes indexed by string offsets."
class="img_no_border"/>
<div class="caption">
Automaton simulation with nodes indexed by string offsets.
<br />The red pipe '<span style="color:red">|</span>' indicates the current position of the
algorithm in the string.
</div>
</div>

The diagram above is probably more explicit, but here are general rules to find
all leftmost longest matches using a one pass non-backtracking automaton
simulation:

<ul>
<li> A node is either empty (unset), or contains an offset from the beginning of the
string indicating where the match considered for this state started.</li>
<li> When multiple transitions can set a state, the lowest offset (leftmost match)
is preferred.</li>
When an exit node is reached:
<ul sytle="margin-bottom:0;">
  <li> A potential match is recorded (starting from the offset indicated in the
  exit node and finishing at the current string offset).</li>
  <li> It is possible that previously recorded potential matches start within the
  range of this newly recorded match; they should be unregistered.  (Consider
  for example matching `(a|aa)` in `aa`)</li>
  <li>  All nodes with a higher offset can be unset:
  they can only yield matches starting within the boundaries of the match just
  found (or the boundaries of future invalidating matches).</li>
</ul>
<li> All registered potential matches that start earlier than any node offset in
the graph are certain to be matches.</li>
</ul>

Rejit first used a similar mechanism. But even this can be painfully slow.

Consider for example the regular expression: `([complex]|(regexp)){2,7}abcdefgh(at|the|[e-nd]as well)`
<br />
The automaton for this regular expression is relatively complex - it has
many states and transitions.
<br />
The initial sub-regexps (especially the repeated bracket `[complex]`) match
easily and allow states to propagate through the nodes of the automaton.

In a situation where the regexp matches rarely (as in most use-cases), the
processing required to simulate the early nodes of the automaton is - most of the
time - lost, and translates into decreased performance.
<br />
Note that the issue applies to simpler (in terms of automaton complexity, not
speed) regexps like `[aeiouy]{1,10}bcdfgh`.

Notice however the sub-regexp `abcdefgh` in the middle. It most likely won't
match very often, but any match for the full regexp *must* have a substring
matching this particular sub-regexp. We can use it to significantly increase the
matching speed.




#### Fast-forwarding mechanism

The fast-forwarding mechanism uses the idea of dominating sub-regexps to
significantly speed up the code.

**Pseudo-definition**:

    ('a' is a dominating sub-regexp of 'b')
    iff
    ('b' matches the string 's' in text => 'a' matches a substring of 's')

In the regexp introduced above, `abcdefgh` and `(at|the|[e-nd]as well)` are
dominating sub-regexps.

Rejit choses what it thinks will be the easiest and fastest dominating
sub-regexp to search for, and generates efficient specialised code (using SIMD
if available). When this fast code finds a match, execution exits and continues
in the 'matching' code, which searches backward and forward for the boundaries
of a complete match.

In the benchmarks section you will find a a graph illustrating the speed-up
provided by this mechanism for this regular expression.
<br />
With this mechanism, the scanning speed for any regular expression can become
close to the scanning speed for the chosen dominating sub-regexp.


##### Choice of the dominating sub-regexp
The choice of the dominating sub-regexp to use for fast-forwarding is critical
for good performance.

We want a dominating sub-regexp that:

* is easy to look for, so we can generate efficient code for it.
* does not match often, so we don't have to fall back to the matching mechanism
to check for other parts of the regexp.

For example `.` is somehow easy to look for but would be a poor choice. It
matches nearly every character!
<br />On the other hand `abcd` is an excellent pattern. Assuming a text randomly
composed of 30 different characters, there is about one chance
in 30^4 to have a match at a particular offset, ie about 1 in 800KiB!

To chose a performant regexp, each elementary type of regular expression
(characters, brackets, anchors, etc.) is assigned a weight indicating how good
a choice it is believed to be.  After the regular expression has been
parsed, Rejit traverses the parse tree and uses those weights to identify the
fastest dominating sub-regexp it can use.



##### Backward matching

After fast-forwarding has found a potential match, the automaton simulation is
executed first backward to look for the start of a match, and then forward to
look for its end.

The code required to match backward is very similar to the code matching forward.
<br />Matching backward with submatches containing greed patterns (eg.
`(a*)(a*)root`) may look like an issue.
Although this is not implemented in Rejit, I believe it can easily be overcome:
<br />
When matching forward, the code updating the nodes of the automaton has to
prefer paths favoring the greedy patterns occurring earlier in the regexp: it
matches as many characters as possible for the earlier greedy patterns.
When matching backward this preference must be preserved, but will be translated
in matching as few characters as possible to greedy patterns occurring later in
the regexp.



#### JIT code

As a JIT compiler, Rejit generates native code tailored for each regular
expression for all the mechanisms described.

SIMD is currently only used to accelerate code paths for fast-forwarding, but
could be used in many more places.
<br />
When available on x86\_64 machines, Rejit uses SSE4.2 and the very convenient
string instructions that help performance and strongly facilitate
implementation, `pcmpistr[im]` (Packed Compare Implicit Length Strings, see
[Intel's developer manuals][Intel manuals]) in particular.  Earlier versions of
SSE are not supported yet.

The generated code is currently not self-modifiable. Using self-modifiable code
could allow changing the behaviour of the code at runtime depending on the text
scanned. It would certainly be a lot of fun to implement! However I am not
sure there would be many practical use-cases, and there is a lot of more
important work to do before that.

The SIMD optimised loops for the fast-forward code are probably (currently) the
most interesting and fun pieces of code generated. The code is a bit dry to
include in this article, but you can have a look at the commented code - for
example the
[code generation for alternation of standard characters strings][SIMD loop example]
(eg. `abcd|012345`).



### Features

Here are a few bullets detailing features that Rejit supports or is still
lacking. There are I believe no technical obstacles to supporting the missing
features. It just needs some more work!

* Rejit delivers high performance while being non-backtracking (like Re2).
This is its main advantage.
* It only supports x86\_64.
<br />Rejit should not be very hard to port to other architectures. The build
system and architecture abstraction mechanisms are similar to those used in V8.
The behaviour of the different assembly stubs is rather simple (usually
corresponding to an elementary type of regexp).
* Case insensitive search is not supported.
* Submatches and backreferences are not supported.
<br />This should be an interesting feature to work on. As detailed earlier I
think the design should allow for these without too much hurdle.
* It supports the ERE (extended regular expression) syntax, but lacks full
support for bracket expressions (eg. `[:digit:]`)
* It only supports ASCII characters, but not wide character types.



### Benchmarks

The results below were produced on a machine with a quad-core Intel Core
i5-2400 CPU @ 3.10GHz with 4GiB RAM on Fedora 17 (3.8.13-100.fc17.x86\_64).
It supports SSE4.2.

Results are reported for the following engines versions:

    GNU grep version 2.14, commit: 599f2b15bc152cdf19022c7803f9a5f336c25e65
    Rejit commit: b29ea4af1a3ae86dcb25bf961bc716029430c9b1
    V8 version 3.20.9, commit: 455bb9c2ab8af080aa15e0fbf4838731f45241e8
    Re2 commit: aa957b5e3374
    Python3 version 3.2.3
    Java version 1.7.0_25

The benchmarks below try to show the best performance achievable by different
engines. <br />
With this goal, comparing other engines to Rejit - which uses SSE4.2 - is fair.
Grep, Re2, and V8 have been recompiled on the tested system, and have the
possibility to use SIMD. In fact both V8 and Re2 use SIMD via the system
`memchr()` implementation (see Rejit benchmarked regexp 1).




#### Single threaded recursive grep through the Linux kernel sources

In both cases the grep program is run a first time without measurement to give
the OS a chance to preload the files. Without this we would be benchmarking the
time to access the files.

    $ CMD='grep -R regexp linux-3.10.6/'; $CMD > /dev/null && time $CMD > /dev/null
    real  0m0.622s
    user  0m0.356s
    sys   0m0.260s

`jrep` is a grep-like utility powered by Rejit.

    $ CMD='jrep -R regexp linux-3.10.6/'; $CMD > /dev/null && time $CMD > /dev/null
    real  0m0.370s
    user  0m0.101s
    sys   0m0.263s

The `jrep` utility performs 1.68 times faster than gnu grep in this very real
use-case!  The time spent in `sys` is equivalent, but Rejit spends 3 times less
time in `user` code.
<br />It is part of the sample programs in the rejit repository (see the
[wiki][rejit wiki]).  It is of course far behind grep in terms of features, but
supports searching for multi-lines patterns and has initial multithreading
support.

#### DNA matching benchmark

From the "[Computer Language Benchmarks Game][cpu_bench]", this
benchmark performs some DNA matching operations using regular expressions.

The tables below show performance (`real` running time) for different input
sizes, for

the
[fastest registered single-threaded implementation][cpu_bench single threaded fastest]
(V8), a single-threaded Rejit-powered implementation, and the fastest
single-threaded registered Python and Java 7 implementations.

    input size                 V8      Rejit     Python       Java
        50.000 (500KB)     0.034s     0.015s     0.173s     0.233s
       500.000 (  5MB)     0.217s     0.130s     1.357s     1.357s
     5.000.000 ( 50MB)     2.054s     1.246s    13.595s    12.304s
    50.000.000 (500MB)      (oom)    14.624s   140.286s      (oom)

    (oom): out of memory

the
[second fastest registered single threaded implementation][cpu_bench multi threaded fastest]
(Re2) (a quick go at running the first listed implementation would raise
failures), a multi-threaded Rejit-powered implementation, and the fastest
multi-threaded registered Python and Java 7 implementations.

    input size                Re2      Rejit     Python       Java
        50.000 (500KB)     0.022s     0.011s     0.167s     0.196s
       500.000 (  5MB)     0.183s     0.087s     0.897s     0.669s
     5.000.000 ( 50MB)     1.629s     0.971s     8.882s     5.166s
    50.000.000 (500MB)    20.693s    11.594s   (killed)      (oom)

    (oom): out of memory
    (killed): program stopped before finishing.

See performance for various engines and languages for
[single-thread][cpu_bench single threaded] and
[multi-threaded][cpu_bench multi threaded] implementations.
The rejit programs used to run these benchmarks are also part of the rejit
sample programs (see the rejit [wiki][rejit wiki]).



#### Rejit benchmarks

These are benchmarks from the Rejit repository.
Engines are benchmarked to find all (leftmost longest) matches of a regular
expression in a randomly generated text (with characters in the range `[0-z]`
if nothing else is specified).
<br />Performance is reported in bytes per second (size of the text matched
divided by the time to process it). This can be seen as the speed at which
files of a given size can be processed. It is easier to report and appreciate
than a delay close to zero.

Benchmark 4 illustrates the fast-forward mechanism, but other benchmarks show
Rejit performing well for more common patterns.

<br />
* '*best* ' performance graphs don't take compilation time into account.
* '*worst* ' performance graphs show performance for 1 run and 1 compilation.
* '*amortised* ' performance graphs show performance for 1 compilation and 100 runs.
  (Due to Javascript `Date` precision in milliseconds, V8 text sizes under
  128KiB use 20.000 runs instead.)


<script language="javascript" type="text/javascript" src="/resources/js/flot/jquery.min.js"> </script>
<script language="javascript" type="text/javascript" src="/resources/js/flot/jquery.flot.min.js"> </script>
<script language="javascript" type="text/javascript" src="/resources/js/flot/jquery.flot.time.min.js"> </script>
<script language="javascript" type="text/javascript" src="/resources/js/flot_utils.js"> </script>

<table>
  <tr>
    <td>
      <div>
        <strong>Regexp 1:</strong> <code>abcdefg</code>
        <p>
        Search for a simple concatenation of characters.
        The text is randomly generated from charactres in the range <code>[b-z]</code>, so the
        starting character <code>a</code> is <strong>not</strong> present in the text searched.
        </p>
        <p>
        V8 and Re2 use <code>memchr()</code> to look for the starting character.
        The absence of the starting character in the text matched allows to stay in
        the <code>memchr()</code> SIMD optimised loop, yielding excellent performances.
        </p>
        <p>
        Rejit generates a custom SSE4.2 loop, which does not look only for the initial <code>a</code>,
        but instead for the whole <code>abcdefg</code> pattern. It does not perform as fast in this
        rather uncommon use-case.
        </p>
      </div>
    </td>
  </tr>
  <tr>
    <td style="padding-bottom:50px;">
      <div>
        <div class="graph" id="plot_parallel_0_chars_null_density" style="float:left;"> </div>
        <div style="float:left;"> <ul id="plot_parallel_0_chars_null_density_choices" style="list-style: none;" class="flot_choices"> </ul> </div>
      </div>
      <script language="javascript" type="text/javascript">
        <!--
        $(function () {
          var data_0_chars_null_density_re2_amortised = [[8,1.90476e+07],[16,3.72093e+07],[32,7.44186e+07],[64,1.45455e+08],[128,2.90909e+08],[256,5.81818e+08],[512,1.11304e+09],[1024,2.0898e+09],[2048,3.79259e+09],[4096,5.68889e+09],[8192,9.99024e+09],[16384,1.36533e+10],[32768,1.45636e+10],[65536,1.54202e+10],[131072,1.64251e+10],[262144,1.98745e+10],[524288,2.41942e+10],[1048576,2.64458e+10],[2097152,2.64959e+10],[4194304,2.47335e+10],[8388608,1.08717e+10],[16777216,9.26309e+09],];
          var data_0_chars_null_density_re2_best = [[8,2.42424e+07],[16,4.70588e+07],[32,9.41176e+07],[64,1.88235e+08],[128,3.76471e+08],[256,7.31429e+08],[512,1.42222e+09],[1024,2.62564e+09],[2048,4.65455e+09],[4096,6.60645e+09],[8192,1.13778e+10],[16384,1.48945e+10],[32768,1.52409e+10],[65536,1.57538e+10],[131072,1.66335e+10],[262144,2.00263e+10],[524288,2.42501e+10],[1048576,2.64792e+10],[2097152,2.65127e+10],[4194304,2.47408e+10],[8388608,1.08734e+10],[16777216,9.2637e+09],];
          var data_0_chars_null_density_re2_worst = [[8,432666],[16,870511],[32,1.74672e+06],[64,3.48205e+06],[128,6.92641e+06],[256,1.3913e+07],[512,2.77507e+07],[1024,5.58038e+07],[2048,1.10108e+08],[4096,2.22246e+08],[8192,4.31612e+08],[16384,8.65962e+08],[32768,1.60313e+09],[65536,3.267e+09],[131072,4.94611e+09],[262144,6.75281e+09],[524288,1.63635e+10],[1048576,2.14389e+10],[2097152,2.35768e+10],[4194304,2.29888e+10],[8388608,1.04394e+10],[16777216,9.09522e+09],];
          var data_0_chars_null_density_rejit_amortised = [[8,1.95122e+07],[16,3.72093e+07],[32,6.95652e+07],[64,1.33333e+08],[128,2.06452e+08],[256,5.12e+08],[512,9.30909e+08],[1024,1.76552e+09],[2048,2.5284e+09],[4096,4.31158e+09],[8192,5.76901e+09],[16384,6.68735e+09],[32768,7.56767e+09],[65536,8.02154e+09],[131072,8.44536e+09],[262144,1.68041e+10],[524288,1.44671e+10],[1048576,1.76083e+10],[2097152,1.75818e+10],[4194304,1.72513e+10],[8388608,1.04481e+10],[16777216,9.04998e+09],];
          var data_0_chars_null_density_rejit_best = [[8,4e+07],[16,7.27273e+07],[32,1.28e+08],[64,2.66667e+08],[128,3.55556e+08],[256,9.48148e+08],[512,1.70667e+09],[1024,2.84444e+09],[2048,3.47119e+09],[4096,5.53514e+09],[8192,6.71475e+09],[16384,7.5156e+09],[32768,7.9922e+09],[65536,8.24352e+09],[131072,8.5612e+09],[262144,1.69125e+10],[524288,1.45152e+10],[1048576,1.76409e+10],[2097152,1.75965e+10],[4194304,1.72598e+10],[8388608,1.04507e+10],[16777216,9.05105e+09],];
          var data_0_chars_null_density_rejit_worst = [[8,353201],[16,721046],[32,1.46252e+06],[64,2.84951e+06],[128,5.86618e+06],[256,1.16735e+07],[512,2.3136e+07],[1024,4.65243e+07],[2048,9.19623e+07],[4096,1.80202e+08],[8192,3.57886e+08],[16384,6.82951e+08],[32768,1.2526e+09],[65536,2.16505e+09],[131072,3.31996e+09],[262144,9.49109e+09],[524288,1.02041e+10],[1048576,1.31203e+10],[2097152,1.56914e+10],[4194304,1.60118e+10],[8388608,9.78024e+09],[16777216,8.80883e+09],];
          var data_0_chars_null_density_v8_amortised = [[8,80000000],[16,160000000],[32,320000000],[64,640000000],[128,853333333.3333333],[256,2560000000],[512,3413333333.333333],[1024,6826666666.666666],[2048,10240000000],[4096,20480000000],[8192,27306666666.666664],[16384,36408888888.888885],[32768,36408888888.888885],[65536,35424864864.86487],[131072,13107200000],[262144,26214400000],[524288,26214400000],[1048576,26214400000],[2097152,23301688888.888885],[4194304,24672376470.588234],[8388608,10754625641.02564],[16777216,9218250549.45055],];
          var datasets = [ {data: data_0_chars_null_density_re2_amortised, label: "re2_amortised", color: "#DEBD00", },{data: data_0_chars_null_density_re2_best, label: "re2_best", color: "#E0D48D", },{data: data_0_chars_null_density_re2_worst, label: "re2_worst", color: "#E0D48D", },{data: data_0_chars_null_density_rejit_amortised, label: "rejit_amortised", color: "#277AD9", },{data: data_0_chars_null_density_rejit_best, label: "rejit_best", color: "#94B8E0", },{data: data_0_chars_null_density_rejit_worst, label: "rejit_worst", color: "#94B8E0", },{data: data_0_chars_null_density_v8_amortised, label: "v8_amortised", color: "#00940A", }, ];
          var choiceContainer = $("#plot_parallel_0_chars_null_density_choices");
          $.each(datasets, function(key, val) {
             choiceContainer.append('<li style="list-style: none;"><input type="checkbox" name="' + key +
                                    '" checked="checked" id="id' + key + '">' +
                                    '<label for="id' + key + '">'
                                    + val.label + '</label></li>');
          });
          plot_according_to_choices("plot_parallel_0_chars_null_density", datasets, choiceContainer);
          $("#plot_parallel_0_chars_null_density").bind("plothover", plothover_func);
          function replot() { plot_according_to_choices("plot_parallel_0_chars_null_density", datasets, choiceContainer); }
          choiceContainer.find("input").change(replot);
          $('.legendColorBox > div').each(function(i){
                                          $(this).clone().prependTo(choiceContainer.find("li").eq(i));
                                          });
        });
        -->
      </script>
    </td>
  </tr>
  <tr>
    <td>
      <div>
        <strong>Regexp 2:</strong> <code>abcdefg</code>
        <p>
          Same as before, except that the text to match is composed of random (equally
          probable) characters in the range [0-z].
          Performance deceases with the higher density of potential
          matches as engines are forced to exit their optimised loop more often.
        </p>
      </div>
    </td>
  </tr>
  <tr>
    <td style="padding-bottom:50px;">
      <div>
        <div class="graph" id="plot_parallel_1_chars_average_density" style="float:left;"> </div>
        <div style="float:left;"> <ul id="plot_parallel_1_chars_average_density_choices" style="list-style: none;" class="flot_choices"> </ul> </div>
      </div>
      <script type="text/javascript">
        <!--
        $(function () {
          var data_1_chars_average_density_re2_amortised = [[8,4e+07],[16,3.63636e+07],[32,7.44186e+07],[64,1.3617e+08],[128,2.56e+08],[256,4.33898e+08],[512,6.2439e+08],[1024,8.60504e+08],[2048,9.52558e+08],[4096,1.16034e+09],[8192,1.26615e+09],[16384,1.28e+09],[32768,1.0639e+09],[65536,1.72327e+09],[131072,1.56654e+09],[262144,1.50467e+09],[524288,1.44464e+09],[1048576,1.43446e+09],[2097152,1.44064e+09],[4194304,1.43739e+09],[8388608,1.40606e+09],[16777216,1.40903e+09],];
          var data_1_chars_average_density_re2_best = [[8,5e+07],[16,4.70588e+07],[32,9.69697e+07],[64,1.72973e+08],[128,3.2e+08],[256,5.22449e+08],[512,7.0137e+08],[1024,9.3945e+08],[2048,9.99024e+08],[4096,1.1907e+09],[8192,1.28401e+09],[16384,1.2911e+09],[32768,1.06736e+09],[65536,1.72554e+09],[131072,1.56747e+09],[262144,1.5051e+09],[524288,1.44484e+09],[1048576,1.43458e+09],[2097152,1.4407e+09],[4194304,1.43741e+09],[8388608,1.40609e+09],[16777216,1.40904e+09],];
          var data_1_chars_average_density_re2_worst = [[8,871460],[16,867209],[32,1.74577e+06],[64,3.39883e+06],[128,6.84126e+06],[256,1.31687e+07],[512,2.44158e+07],[1024,4.67794e+07],[2048,8.77464e+07],[4096,1.58208e+08],[8192,2.82873e+08],[16384,4.37724e+08],[32768,5.73068e+08],[65536,7.23276e+08],[131072,1.23281e+09],[262144,1.3596e+09],[524288,1.35974e+09],[1048576,1.40312e+09],[2097152,1.41654e+09],[4194304,1.42881e+09],[8388608,1.39908e+09],[16777216,1.40623e+09],];
          var data_1_chars_average_density_rejit_amortised = [[8,1.95122e+07],[16,3.80952e+07],[32,6.80851e+07],[64,1.3617e+08],[128,2.61224e+08],[256,4.65455e+08],[512,9.14286e+08],[1024,1.6254e+09],[2048,2.40941e+09],[4096,3.62478e+09],[8192,4.55111e+09],[16384,5.5539e+09],[32768,5.94701e+09],[65536,6.21784e+09],[131072,6.32587e+09],[262144,1.25248e+10],[524288,1.16534e+10],[1048576,1.17871e+10],[2097152,1.15533e+10],[4194304,1.1462e+10],[8388608,9.26611e+09],[16777216,8.69998e+09],];
          var data_1_chars_average_density_rejit_best = [[8,3.80952e+07],[16,7.27273e+07],[32,1.28e+08],[64,2.66667e+08],[128,5.12e+08],[256,9.14286e+08],[512,1.65161e+09],[1024,2.56e+09],[2048,3.30323e+09],[4096,4.4043e+09],[8192,5.1522e+09],[16384,6.00147e+09],[32768,6.171e+09],[65536,6.34424e+09],[131072,6.39688e+09],[262144,1.26578e+10],[524288,1.1682e+10],[1048576,1.18003e+10],[2097152,1.15603e+10],[4194304,1.14664e+10],[8388608,9.26908e+09],[16777216,8.70102e+09],];
          var data_1_chars_average_density_rejit_worst = [[8,360848],[16,720721],[32,1.46789e+06],[64,2.93175e+06],[128,5.85544e+06],[256,1.16047e+07],[512,2.31151e+07],[1024,4.65666e+07],[2048,9.22938e+07],[4096,1.80919e+08],[8192,3.49488e+08],[16384,6.62783e+08],[32768,1.20871e+09],[65536,2.01773e+09],[131072,2.90368e+09],[262144,3.78001e+09],[524288,8.71055e+09],[1048576,9.93158e+09],[2097152,1.07183e+10],[4194304,1.08498e+10],[8388608,8.73868e+09],[16777216,8.47026e+09],];
          var data_1_chars_average_density_v8_amortised = [[8,80000000],[16,160000000],[32,213333333.3333333],[64,426666666.6666666],[128,640000000],[256,1280000000],[512,1706666666.6666665],[1024,1575384615.3846157],[2048,1517037037.0370371],[4096,1575384615.3846157],[8192,1606274509.8039217],[16384,1567846889.9521532],[32768,1564105011.9331744],[65536,1520556844.5475638],[131072,1456355555.5555553],[262144,1456355555.5555553],[524288,1456355555.5555553],[1048576,1476867605.633803],[2097152,1476867605.633803],[4194304,1476867605.633803],[8388608,1443822375.2151463],[16777216,1441341580.7560136],];
          var datasets = [ {data: data_1_chars_average_density_re2_amortised, label: "re2_amortised", color: "#DEBD00", },{data: data_1_chars_average_density_re2_best, label: "re2_best", color: "#E0D48D", },{data: data_1_chars_average_density_re2_worst, label: "re2_worst", color: "#E0D48D", },{data: data_1_chars_average_density_rejit_amortised, label: "rejit_amortised", color: "#277AD9", },{data: data_1_chars_average_density_rejit_best, label: "rejit_best", color: "#94B8E0", },{data: data_1_chars_average_density_rejit_worst, label: "rejit_worst", color: "#94B8E0", },{data: data_1_chars_average_density_v8_amortised, label: "v8_amortised", color: "#00940A", }, ];
            var choiceContainer = $("#plot_parallel_1_chars_average_density_choices");
            $.each(datasets, function(key, val) {
               choiceContainer.append('<li style="list-style: none;"><input type="checkbox" name="' + key +
                                      '" checked="checked" id="id' + key + '">' +
                                      '<label for="id' + key + '">'
                                      + val.label + '</label></li>');
            });
            plot_according_to_choices("plot_parallel_1_chars_average_density", datasets, choiceContainer);
            $("#plot_parallel_1_chars_average_density").bind("plothover", plothover_func);
            function replot() { plot_according_to_choices("plot_parallel_1_chars_average_density", datasets, choiceContainer); }
            choiceContainer.find("input").change(replot);
            $('.legendColorBox > div').each(function(i){
                                            $(this).clone().prependTo(choiceContainer.find("li").eq(i));
                                            });
          });
        -->
      </script>
    </td>
  </tr>
  <tr>
    <td>
      <div>
        <strong>Regexp 3:</strong> <code>abcdefg</code>
        <p>
        Same as before, but this time the character range is [a-j].
        Performance decreases further.
        </p>
      </div>
    </td>
  </tr>
  <tr>
    <td style="padding-bottom:50px;">
      <div>
        <div class="graph" id="plot_parallel_2_chars_high_density" style="float:left;"> </div>
        <div style="float:left;"> <ul id="plot_parallel_2_chars_high_density_choices" style="list-style: none;" class="flot_choices"> </ul> </div>
      </div>
      <script type="text/javascript">
        <!--
        $(function () {
          var data_2_chars_high_density_re2_amortised = [[8,1.73913e+07],[16,3.55556e+07],[32,6.66667e+07],[64,1.12281e+08],[128,1.70667e+08],[256,2.81319e+08],[512,4e+08],[1024,4.35745e+08],[2048,5.10723e+08],[4096,4.91127e+08],[8192,4.51599e+08],[16384,4.05043e+08],[32768,7.20018e+08],[65536,6.95859e+08],[131072,6.89599e+08],[262144,6.78602e+08],[524288,6.73753e+08],[1048576,6.65091e+08],[2097152,6.73027e+08],[4194304,6.70371e+08],[8388608,6.63583e+08],[16777216,6.62763e+08],];
          var data_2_chars_high_density_re2_best = [[8,2.22222e+07],[16,4.57143e+07],[32,8.42105e+07],[64,1.33333e+08],[128,1.96923e+08],[256,3.16049e+08],[512,4.30252e+08],[1024,4.53097e+08],[2048,5.23785e+08],[4096,4.97087e+08],[8192,4.54102e+08],[16384,4.06047e+08],[32768,7.2081e+08],[65536,6.96155e+08],[131072,6.8978e+08],[262144,6.78708e+08],[524288,6.73797e+08],[1048576,6.65116e+08],[2097152,6.7304e+08],[4194304,6.70378e+08],[8388608,6.6359e+08],[16777216,6.62767e+08],];
          var data_2_chars_high_density_re2_worst = [[8,418191],[16,845666],[32,1.70485e+06],[64,3.16832e+06],[128,5.70664e+06],[256,1.08245e+07],[512,2.0975e+07],[1024,3.75092e+07],[2048,6.9518e+07],[4096,1.19417e+08],[8192,1.82735e+08],[16384,2.32992e+08],[32768,2.74301e+08],[65536,5.5985e+08],[131072,6.02519e+08],[262144,6.36844e+08],[524288,6.52246e+08],[1048576,6.58943e+08],[2097152,6.67713e+08],[4194304,6.67604e+08],[8388608,6.62518e+08],[16777216,6.63218e+08],];
          var data_2_chars_high_density_rejit_amortised = [[8,1.70213e+07],[16,3.63636e+07],[32,6.95652e+07],[64,1.33333e+08],[128,2.5098e+08],[256,4.33898e+08],[512,8.12698e+08],[1024,1.34737e+09],[2048,2.22609e+09],[4096,2.69474e+09],[8192,3.47119e+09],[16384,4.14785e+09],[32768,4.3749e+09],[65536,4.49801e+09],[131072,7.70558e+09],[262144,7.57204e+09],[524288,7.01107e+09],[1048576,6.77069e+09],[2097152,6.7539e+09],[4194304,6.71798e+09],[8388608,6.12209e+09],[16777216,6.09928e+09],];
          var data_2_chars_high_density_rejit_best = [[8,2.96296e+07],[16,7.27273e+07],[32,1.28e+08],[64,2.56e+08],[128,4.74074e+08],[256,7.52941e+08],[512,1.31282e+09],[1024,1.8963e+09],[2048,2.92571e+09],[4096,3.12672e+09],[8192,3.81023e+09],[16384,4.36907e+09],[32768,4.49492e+09],[65536,4.56379e+09],[131072,7.79726e+09],[262144,7.59618e+09],[524288,7.02046e+09],[1048576,6.7755e+09],[2097152,6.75629e+09],[4194304,6.71938e+09],[8388608,6.12312e+09],[16777216,6.09979e+09],];
          var data_2_chars_high_density_rejit_worst = [[8,363306],[16,728929],[32,1.46052e+06],[64,2.92237e+06],[128,5.85812e+06],[256,1.16895e+07],[512,2.31674e+07],[1024,4.58781e+07],[2048,9.11032e+07],[4096,1.78865e+08],[8192,3.40058e+08],[16384,6.29669e+08],[32768,1.11003e+09],[65536,1.75418e+09],[131072,2.43628e+09],[262144,5.7894e+09],[524288,5.34225e+09],[1048576,6.08894e+09],[2097152,6.3475e+09],[4194304,6.4393e+09],[8388608,5.90273e+09],[16777216,5.95167e+09],];
          var data_2_chars_high_density_v8_amortised = [[8,80000000],[16,160000000],[32,213333333.3333333],[64,426666666.6666666],[128,512000000],[256,731428571.4285715],[512,1024000000],[1024,1024000000],[2048,1204705882.352941],[4096,1170285714.2857141],[8192,1170285714.2857141],[16384,1166120996.441281],[32768,1092266666.6666667],[65536,989222641.5094339],[131072,936228571.4285715],[262144,936228571.4285715],[524288,919803508.7719297],[1048576,1872457142.857143],[2097152,1855886725.6637168],[4194304,934143429.844098],[8388608,918796056.9550931],[16777216,2777684768.2119207],];
            var datasets = [ {data: data_2_chars_high_density_re2_amortised, label: "re2_amortised", color: "#DEBD00", },{data: data_2_chars_high_density_re2_best, label: "re2_best", color: "#E0D48D", },{data: data_2_chars_high_density_re2_worst, label: "re2_worst", color: "#E0D48D", },{data: data_2_chars_high_density_rejit_amortised, label: "rejit_amortised", color: "#277AD9", },{data: data_2_chars_high_density_rejit_best, label: "rejit_best", color: "#94B8E0", },{data: data_2_chars_high_density_rejit_worst, label: "rejit_worst", color: "#94B8E0", },{data: data_2_chars_high_density_v8_amortised, label: "v8_amortised", color: "#00940A", }, ];
            var choiceContainer = $("#plot_parallel_2_chars_high_density_choices");
            $.each(datasets, function(key, val) {
               choiceContainer.append('<li style="list-style: none;"><input type="checkbox" name="' + key +
                                      '" checked="checked" id="id' + key + '">' +
                                      '<label for="id' + key + '">'
                                      + val.label + '</label></li>');
            });
            plot_according_to_choices("plot_parallel_2_chars_high_density", datasets, choiceContainer);
            $("#plot_parallel_2_chars_high_density").bind("plothover", plothover_func);
            function replot() { plot_according_to_choices("plot_parallel_2_chars_high_density", datasets, choiceContainer); }
            choiceContainer.find("input").change(replot);
            $('.legendColorBox > div').each(function(i){
                                            $(this).clone().prependTo(choiceContainer.find("li").eq(i));
                                            });
          });
        -->
      </script>
    </td>
  </tr>
  <tr>
    <td>
      <div>
        <strong>Regexp 4:</strong> <code>([complex]|(regexp)){2,7}abcdefgh(at|the|[e-nd]as well)</code>
        <p>
        This illustrate the "fast forward" mechanism used in Rejit.
        <br />
        Rejit choses to scan the text for <code>abcdefgh</code> and, when a match
        is found, scans backward and forward for the limits of the regular expression.
        <br />The performance profile for Rejit looks clearly like for regexp 2 above.
        </p>
      </div>
    </td>
  </tr>
  <tr>
    <td style="padding-bottom:50px;">
      <div>
        <div class="graph" id="plot_parallel_3_complex_simple_ff" style="float:left;"> </div>
        <div style="float:left;"> <ul id="plot_parallel_3_complex_simple_ff_choices" style="list-style: none;" class="flot_choices"> </ul> </div>
      </div>
      <script type="text/javascript">
        <!--
        $(function () {
          var data_3_complex_simple_ff_re2_amortised = [[8,7.14286e+06],[16,1.36752e+07],[32,2.0915e+07],[64,6.73684e+07],[128,8.47682e+07],[256,1.1179e+08],[512,1.01587e+08],[1024,1.45455e+08],[2048,1.54217e+08],[4096,1.58945e+08],[8192,1.62669e+08],[16384,1.62717e+08],[32768,1.64738e+08],[65536,1.65583e+08],[131072,1.65805e+08],[262144,1.65638e+08],[524288,1.65929e+08],[1048576,1.66431e+08],[2097152,1.65816e+08],[4194304,1.66008e+08],[8388608,1.66101e+08],[16777216,1.66252e+08],];
          var data_3_complex_simple_ff_re2_best = [[8,1.45455e+07],[16,2.42424e+07],[32,3.13725e+07],[64,1.01587e+08],[128,1.00787e+08],[256,1.27363e+08],[512,1.09168e+08],[1024,1.5081e+08],[2048,1.57176e+08],[4096,1.60564e+08],[8192,1.63513e+08],[16384,1.63155e+08],[32768,1.6497e+08],[65536,1.65717e+08],[131072,1.65868e+08],[262144,1.65671e+08],[524288,1.65945e+08],[1048576,1.66442e+08],[2097152,1.65821e+08],[4194304,1.6601e+08],[8388608,1.66102e+08],[16777216,1.66253e+08],];
          var data_3_complex_simple_ff_re2_worst = [[8,106369],[16,196415],[32,320738],[64,562242],[128,1.90024e+06],[256,3.12309e+06],[512,4.1687e+06],[1024,6.74839e+06],[2048,1.2904e+07],[4096,2.20156e+07],[8192,3.60897e+07],[16384,5.20044e+07],[32768,7.33902e+07],[65536,9.66494e+07],[131072,1.18346e+08],[262144,1.34615e+08],[524288,1.47274e+08],[1048576,1.55664e+08],[2097152,1.603e+08],[4194304,1.62882e+08],[8388608,1.64401e+08],[16777216,1.64899e+08],];
          var data_3_complex_simple_ff_rejit_amortised = [[8,4.65116e+06],[16,9.35673e+06],[32,3.90244e+07],[64,7.80488e+07],[128,1.20755e+08],[256,1.92481e+08],[512,3.55556e+08],[1024,6.5641e+08],[2048,1.11304e+09],[4096,1.71381e+09],[8192,5.76901e+09],[16384,8.31675e+09],[32768,8.55561e+09],[65536,8.30621e+09],[131072,8.22284e+09],[262144,1.25069e+10],[524288,1.23217e+10],[1048576,1.17553e+10],[2097152,1.15121e+10],[4194304,1.14671e+10],[8388608,9.30899e+09],[16777216,8.68939e+09],];
          var data_3_complex_simple_ff_rejit_best = [[8,1.12676e+07],[16,2.16216e+07],[32,8.88889e+07],[64,1.82857e+08],[128,2.78261e+08],[256,4.49123e+08],[512,6.82667e+08],[1024,1.40274e+09],[2048,2.06869e+09],[4096,2.88451e+09],[8192,8.53333e+09],[16384,1.07789e+10],[32768,1.00515e+10],[65536,9.05193e+09],[131072,8.61183e+09],[262144,1.27813e+10],[524288,1.24564e+10],[1048576,1.18176e+10],[2097152,1.15425e+10],[4194304,1.14834e+10],[8388608,9.31581e+09],[16777216,8.69245e+09],];
          var data_3_complex_simple_ff_rejit_worst = [[8,79594.1],[16,159952],[32,332640],[64,1.31741e+06],[128,2.08911e+06],[256,3.5286e+06],[512,7.06597e+06],[1024,1.38998e+07],[2048,2.44596e+07],[4096,4.2075e+07],[8192,8.75027e+07],[16384,3.32333e+08],[32768,5.42606e+08],[65536,9.57849e+08],[131072,1.50657e+09],[262144,2.98196e+09],[524288,5.49856e+09],[1048576,7.27571e+09],[2097152,8.90737e+09],[4194304,9.6817e+09],[8388608,8.28857e+09],[16777216,8.23891e+09],];
          var data_3_complex_simple_ff_v8_amortised = [[8,80000000],[16,106666666.66666666],[32,128000000],[64,182857142.85714287],[128,232727272.72727272],[256,243809523.8095238],[512,269473684.2105263],[1024,230112359.5505618],[2048,240941176.4705882],[4096,169256198.34710747],[8192,166504065.04065043],[16384,163349950.14955133],[32768,155963826.74916705],[65536,155261786.30656242],[131072,154202352.94117644],[262144,155114792.89940828],[524288,155114792.89940828],[1048576,155344592.5925926],[2097152,154315820.45621783],[4194304,155000147.81966],[8388608,154600221.15739036],[16777216,154457889.89136437],];
            var datasets = [ {data: data_3_complex_simple_ff_re2_amortised, label: "re2_amortised", color: "#DEBD00", },{data: data_3_complex_simple_ff_re2_best, label: "re2_best", color: "#E0D48D", },{data: data_3_complex_simple_ff_re2_worst, label: "re2_worst", color: "#E0D48D", },{data: data_3_complex_simple_ff_rejit_amortised, label: "rejit_amortised", color: "#277AD9", },{data: data_3_complex_simple_ff_rejit_best, label: "rejit_best", color: "#94B8E0", },{data: data_3_complex_simple_ff_rejit_worst, label: "rejit_worst", color: "#94B8E0", },{data: data_3_complex_simple_ff_v8_amortised, label: "v8_amortised", color: "#00940A", }, ];
            var choiceContainer = $("#plot_parallel_3_complex_simple_ff_choices");
            $.each(datasets, function(key, val) {
               choiceContainer.append('<li style="list-style: none;"><input type="checkbox" name="' + key +
                                      '" checked="checked" id="id' + key + '">' +
                                      '<label for="id' + key + '">'
                                      + val.label + '</label></li>');
            });
            plot_according_to_choices("plot_parallel_3_complex_simple_ff", datasets, choiceContainer);
            $("#plot_parallel_3_complex_simple_ff").bind("plothover", plothover_func);
            function replot() { plot_according_to_choices("plot_parallel_3_complex_simple_ff", datasets, choiceContainer); }
            choiceContainer.find("input").change(replot);
            $('.legendColorBox > div').each(function(i){
                                            $(this).clone().prependTo(choiceContainer.find("li").eq(i));
                                            });
          });
        -->
      </script>
    </td>
  </tr>
  <tr>
    <td>
      <div>
        <strong>Regexp 5:</strong> <code>(abcd---|efgh___)</code>
        <p>
        Search for an alternation of two words with no substring in common.
        </p>
        <p>
        Rejit has no choice but to look for both strings. The performance is about
        half the performance to search for single string.
        <br />It is slightly under that because the different prefixes for the
        two strings introduce more chances to exit from the <code>pcmpistri</code>
        based code loop. Compare with the following benchmark.
        </p>
      </div>
    </td>
  </tr>
  <tr>
    <td style="padding-bottom:50px;">
      <div>
        <div class="graph" id="plot_parallel_4_alternation_2_np" style="float:left;"> </div>
        <div style="float:left;"> <ul id="plot_parallel_4_alternation_2_np_choices" style="list-style: none;" class="flot_choices"> </ul> </div>
      </div>
      <script type="text/javascript">
        <!--
        $(function () {
          var data_4_alternation_2_np_re2_amortised = [[8,1.53846e+07],[16,2.71186e+07],[32,4.38356e+07],[64,6.59794e+07],[128,7.52941e+07],[256,9.62406e+07],[512,1.04065e+08],[1024,1.08245e+08],[2048,1.11123e+08],[4096,1.11033e+08],[8192,1.17095e+08],[16384,2.18017e+08],[32768,2.38036e+08],[65536,2.39375e+08],[131072,2.40054e+08],[262144,2.39436e+08],[524288,2.38309e+08],[1048576,2.38703e+08],[2097152,2.39298e+08],[4194304,2.39262e+08],[8388608,2.38271e+08],[16777216,2.38283e+08],];
          var data_4_alternation_2_np_re2_best = [[8,2.10526e+07],[16,3.55556e+07],[32,5.42373e+07],[64,7.61905e+07],[128,8.15287e+07],[256,1.01587e+08],[512,1.07113e+08],[1024,1.09753e+08],[2048,1.11974e+08],[4096,1.11456e+08],[8192,1.17313e+08],[16384,2.18221e+08],[32768,2.38157e+08],[65536,2.39445e+08],[131072,2.4012e+08],[262144,2.39451e+08],[524288,2.38318e+08],[1048576,2.38708e+08],[2097152,2.39301e+08],[4194304,2.39263e+08],[8388608,2.38272e+08],[16777216,2.38284e+08],];
          var data_4_alternation_2_np_re2_worst = [[8,351648],[16,713967],[32,1.33891e+06],[64,2.54271e+06],[128,4.81928e+06],[256,8.93855e+06],[512,1.51659e+07],[1024,2.55171e+07],[2048,4.02912e+07],[4096,5.72787e+07],[8192,7.61126e+07],[16384,1.80381e+08],[32768,1.97219e+08],[65536,2.14654e+08],[131072,2.27244e+08],[262144,2.32634e+08],[524288,2.35098e+08],[1048576,2.37213e+08],[2097152,2.38395e+08],[4194304,2.38572e+08],[8388608,2.37715e+08],[16777216,2.38355e+08],];
          var data_4_alternation_2_np_rejit_amortised = [[8,1.95122e+07],[16,3.33333e+07],[32,5.92593e+07],[64,7.90123e+07],[128,1.62025e+08],[256,3.45946e+08],[512,6.09524e+08],[1024,1.10108e+09],[2048,1.34737e+09],[4096,1.71381e+09],[8192,2.17872e+09],[16384,2.48997e+09],[32768,2.69695e+09],[65536,2.8127e+09],[131072,5.86714e+09],[262144,5.24918e+09],[524288,5.11401e+09],[1048576,4.90401e+09],[2097152,4.79448e+09],[4194304,4.76886e+09],[8388608,4.50608e+09],[16777216,4.48573e+09],];
          var data_4_alternation_2_np_rejit_best = [[8,5e+07],[16,7.61905e+07],[32,1.10345e+08],[64,1.14286e+08],[128,2.37037e+08],[256,5.56522e+08],[512,8.67797e+08],[1024,1.57538e+09],[2048,1.6384e+09],[4096,1.93208e+09],[8192,2.3339e+09],[16384,2.58831e+09],[32768,2.75593e+09],[65536,2.84321e+09],[131072,5.89883e+09],[262144,5.26394e+09],[524288,5.1215e+09],[1048576,4.907e+09],[2097152,4.79601e+09],[4194304,4.76978e+09],[8388608,4.50685e+09],[16777216,4.48605e+09],];
          var data_4_alternation_2_np_rejit_worst = [[8,295858],[16,599251],[32,1.18827e+06],[64,2.36949e+06],[128,4.63265e+06],[256,9.21858e+06],[512,1.85979e+07],[1024,3.69542e+07],[2048,7.20619e+07],[4096,1.41877e+08],[8192,2.68678e+08],[16384,4.87184e+08],[32768,8.20842e+08],[65536,1.28025e+09],[131072,1.91262e+09],[262144,3.90851e+09],[524288,4.08324e+09],[1048576,4.43523e+09],[2097152,4.53301e+09],[4194304,4.55754e+09],[8388608,4.34254e+09],[16777216,4.37495e+09],];
          var data_4_alternation_2_np_v8_amortised = [[8,80000000],[16,160000000],[32,213333333.3333333],[64,640000000],[128,853333333.3333333],[256,2560000000],[512,2560000000],[1024,4096000000],[2048,4551111111.111111],[4096,5461333333.333334],[8192,5649655172.413793],[16384,5201269841.269841],[32768,4615211267.605634],[65536,3788208092.4855485],[131072,2621440000],[262144,2912711111.1111107],[524288,2912711111.1111107],[1048576,2912711111.1111107],[2097152,2995931428.5714283],[4194304,2974683687.943262],[8388608,2863006143.34471],[16777216,2848423769.1001697],];
            var datasets = [ {data: data_4_alternation_2_np_re2_amortised, label: "re2_amortised", color: "#DEBD00", },{data: data_4_alternation_2_np_re2_best, label: "re2_best", color: "#E0D48D", },{data: data_4_alternation_2_np_re2_worst, label: "re2_worst", color: "#E0D48D", },{data: data_4_alternation_2_np_rejit_amortised, label: "rejit_amortised", color: "#277AD9", },{data: data_4_alternation_2_np_rejit_best, label: "rejit_best", color: "#94B8E0", },{data: data_4_alternation_2_np_rejit_worst, label: "rejit_worst", color: "#94B8E0", },{data: data_4_alternation_2_np_v8_amortised, label: "v8_amortised", color: "#00940A", }, ];
            var choiceContainer = $("#plot_parallel_4_alternation_2_np_choices");
            $.each(datasets, function(key, val) {
               choiceContainer.append('<li style="list-style: none;"><input type="checkbox" name="' + key +
                                      '" checked="checked" id="id' + key + '">' +
                                      '<label for="id' + key + '">'
                                      + val.label + '</label></li>');
            });
            plot_according_to_choices("plot_parallel_4_alternation_2_np", datasets, choiceContainer);
            $("#plot_parallel_4_alternation_2_np").bind("plothover", plothover_func);
            function replot() { plot_according_to_choices("plot_parallel_4_alternation_2_np", datasets, choiceContainer); }
            choiceContainer.find("input").change(replot);
            $('.legendColorBox > div').each(function(i){
                                            $(this).clone().prependTo(choiceContainer.find("li").eq(i));
                                            });
          });
        -->
      </script>
    </td>
  </tr>
  <tr>
    <td>
      <div>
        <strong>Regexp 6:</strong> <code>(abcd---|abcd___)</code>
        <p>
        Search for an alternation of two words sharing a common prefix.
        <br />
        Re2 spots the common prefix and uses the information to speed up the search,
        probably by interpreting it as <code>abcd(---|___)</code>.
        Rejit and (apparently) V8 do not do that yet. With it performance would
        jump back to something like for regexp 2.
        </p>
      </div>
    </td>
  </tr>
  <tr>
    <td style="padding-bottom:50px;">
      <div>
        <div class="graph" id="plot_parallel_5_alternation_2_p" style="float:left;"> </div>
        <div style="float:left;"> <ul id="plot_parallel_5_alternation_2_p_choices" style="list-style: none;" class="flot_choices"> </ul> </div>
      </div>
      <script type="text/javascript">
        <!--
        $(function () {
          var data_5_alternation_2_p_re2_amortised = [[8,1.6e+07],[16,3.26531e+07],[32,6.4e+07],[64,1.18519e+08],[128,2.32727e+08],[256,4e+08],[512,5.88506e+08],[1024,8.25806e+08],[2048,1.024e+09],[4096,1.13149e+09],[8192,1.26811e+09],[16384,1.2911e+09],[32768,1.11267e+09],[65536,1.61141e+09],[131072,1.55152e+09],[262144,1.49668e+09],[524288,1.43917e+09],[1048576,1.4412e+09],[2097152,1.4349e+09],[4194304,1.43654e+09],[8388608,1.41314e+09],[16777216,1.40814e+09],];
          var data_5_alternation_2_p_re2_best = [[8,2.35294e+07],[16,4.84848e+07],[32,9.41176e+07],[64,1.68421e+08],[128,3.28205e+08],[256,5.33333e+08],[512,7.21127e+08],[1024,9.48148e+08],[2048,1.11304e+09],[4096,1.18382e+09],[8192,1.30032e+09],[16384,1.30863e+09],[32768,1.11913e+09],[65536,1.61458e+09],[131072,1.55299e+09],[262144,1.49745e+09],[524288,1.43952e+09],[1048576,1.4414e+09],[2097152,1.435e+09],[4194304,1.43659e+09],[8388608,1.41319e+09],[16777216,1.40817e+09],];
          var data_5_alternation_2_p_re2_worst = [[8,312256],[16,636436],[32,1.25984e+06],[64,2.4961e+06],[128,4.95164e+06],[256,9.84615e+06],[512,1.90335e+07],[1024,3.69142e+07],[2048,6.88172e+07],[4096,1.30571e+08],[8192,2.38417e+08],[16384,3.94795e+08],[32768,5.22116e+08],[65536,1.22934e+09],[131072,1.02657e+09],[262144,1.28634e+09],[524288,1.35126e+09],[1048576,1.39833e+09],[2097152,1.41214e+09],[4194304,1.42183e+09],[8388608,1.40395e+09],[16777216,1.4041e+09],];
          var data_5_alternation_2_p_rejit_amortised = [[8,1.90476e+07],[16,3.47826e+07],[32,5.33333e+07],[64,8e+07],[128,1.56098e+08],[256,4e+08],[512,7.42029e+08],[1024,1.21905e+09],[2048,1.35629e+09],[4096,2.02772e+09],[8192,2.55202e+09],[16384,2.96812e+09],[32768,3.19065e+09],[65536,3.39917e+09],[131072,3.60088e+09],[262144,6.62482e+09],[524288,7.01201e+09],[1048576,6.49957e+09],[2097152,6.20606e+09],[4194304,6.24422e+09],[8388608,5.73949e+09],[16777216,5.71569e+09],];
          var data_5_alternation_2_p_rejit_best = [[8,4.70588e+07],[16,8e+07],[32,9.69697e+07],[64,1.16364e+08],[128,2.24561e+08],[256,6.73684e+08],[512,1.16364e+09],[1024,1.79649e+09],[2048,1.66504e+09],[4096,2.31412e+09],[8192,2.76757e+09],[16384,3.10892e+09],[32768,3.27026e+09],[65536,3.44564e+09],[131072,3.62678e+09],[262144,6.64665e+09],[524288,7.02422e+09],[1048576,6.50481e+09],[2097152,6.20845e+09],[4194304,6.2458e+09],[8388608,5.74047e+09],[16777216,5.7162e+09],];
          var data_5_alternation_2_p_rejit_worst = [[8,297508],[16,598802],[32,1.17345e+06],[64,2.3845e+06],[128,4.68178e+06],[256,9.36015e+06],[512,1.87408e+07],[1024,3.71688e+07],[2048,7.27531e+07],[4096,1.41828e+08],[8192,2.72885e+08],[16384,4.99056e+08],[32768,8.70101e+08],[65536,1.38203e+09],[131072,1.9444e+09],[262144,4.86262e+09],[524288,5.24498e+09],[1048576,5.80318e+09],[2097152,5.8808e+09],[4194304,5.94861e+09],[8388608,5.50037e+09],[16777216,5.57573e+09],];
          var data_5_alternation_2_p_v8_amortised = [[8,160000000],[16,160000000],[32,320000000],[64,640000000],[128,1280000000],[256,1706666666.6666665],[512,3413333333.333333],[1024,3413333333.333333],[2048,5120000000],[4096,5461333333.333334],[8192,5851428571.428572],[16384,5957818181.818181],[32768,5416198347.107439],[65536,4983726235.741445],[131072,4369066666.666666],[262144,4369066666.666666],[524288,4032984615.384616],[1048576,4032984615.384616],[2097152,4194304000],[4194304,4194304000],[8388608,3938313615.023474],[16777216,3929090398.1264634],];
            var datasets = [ {data: data_5_alternation_2_p_re2_amortised, label: "re2_amortised", color: "#DEBD00", },{data: data_5_alternation_2_p_re2_best, label: "re2_best", color: "#E0D48D", },{data: data_5_alternation_2_p_re2_worst, label: "re2_worst", color: "#E0D48D", },{data: data_5_alternation_2_p_rejit_amortised, label: "rejit_amortised", color: "#277AD9", },{data: data_5_alternation_2_p_rejit_best, label: "rejit_best", color: "#94B8E0", },{data: data_5_alternation_2_p_rejit_worst, label: "rejit_worst", color: "#94B8E0", },{data: data_5_alternation_2_p_v8_amortised, label: "v8_amortised", color: "#00940A", }, ];
            var choiceContainer = $("#plot_parallel_5_alternation_2_p_choices");
            $.each(datasets, function(key, val) {
               choiceContainer.append('<li style="list-style: none;"><input type="checkbox" name="' + key +
                                      '" checked="checked" id="id' + key + '">' +
                                      '<label for="id' + key + '">'
                                      + val.label + '</label></li>');
            });
            plot_according_to_choices("plot_parallel_5_alternation_2_p", datasets, choiceContainer);
            $("#plot_parallel_5_alternation_2_p").bind("plothover", plothover_func);
            function replot() { plot_according_to_choices("plot_parallel_5_alternation_2_p", datasets, choiceContainer); }
            choiceContainer.find("input").change(replot);
            $('.legendColorBox > div').each(function(i){
                                            $(this).clone().prependTo(choiceContainer.find("li").eq(i));
                                            });
          });
        -->
      </script>
    </td>
  </tr>
</table>




### Future work

Performance already looks good, and the fast-forward mechanism enables stable
high performance for a wide range of regular expressions, but no real work has
yet gone into fine optimisation.
The handling of many 'tricky' regular expressions could be improved to yield
much faster code. But even simple word searches like in benchmarks 1, 2, and 3
may benefit from a rewrite: `pcmp_str_` instructions are convenient, but well
designed code using AVX `pcmp` may turn out to be faster.

Before any further performance work, important features like
case-insensitive search, submatches, backward-references, handling of other
character types, etc.  may be required to make rejit more attractive for general
purposes.

For questions, feedback, suggestions, to talk about regular expression matching,
optimisation, or other related topics, please use the [rejit-users group][rejit_users]
or [email me][email alexandre].



  [rejit github]: https://github.com/coreperf/rejit
  [rejit wiki]: https://github.com/coreperf/rejit/wiki
  [Russ Cox articles]: http://swtch.com/~rsc/regexp/
  [Russ Cox article 1]: http://swtch.com/~rsc/regexp/regexp1.html
  [Wikipedia NFA]: http://en.wikipedia.org/wiki/Nondeterministic_finite_automaton
  [Intel manuals]: http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html
  [SIMD loop example]: https://github.com/arames/rejit/blob/master/src/x64/codegen-x64.cc#L1064
  [cpu_bench]: http://benchmarksgame.alioth.debian.org/
  [cpu_bench single threaded fastest]: http://benchmarksgame.alioth.debian.org/u64/program.php?test=regexdna&lang=v8&id=2
  [cpu_bench single threaded]: http://benchmarksgame.alioth.debian.org/u64/benchmark.php?test=regexdna&lang=all&data=u64
  [cpu_bench multi threaded fastest]: http://benchmarksgame.alioth.debian.org/u32q/program.php?test=regexdna&lang=gpp&id=2
  [cpu_bench multi threaded]: http://benchmarksgame.alioth.debian.org/u64q/benchmark.php?test=regexdna&lang=all&data=u64q
  [rejit_users]: https://groups.google.com/forum/#!forum/rejit-users
  [email alexandre]: mailto:alexandre@coreperf.com
