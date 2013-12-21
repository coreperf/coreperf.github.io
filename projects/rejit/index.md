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

  <ol>
    <li> Existence: is there a partial match of the regexp in the text?</li>
    <li> Location: Is there a partial match and, if so, where is it?
         <br />
         This is harder than the 'existence' question.
         We are looking for the leftmost longest match. We must keep track of the
         boundaries of the match, and cannot exit as soon as any match is found.
    </li>
  </ol>

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
in 30^4 to have a match at a particular offset, ie. about 1 in 800KiB!

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
<br />Matching backward with submatches containing greed patterns (e.g.
`(a*)(a*)root`) may look like an issue.
Although this is not implemented in Rejit, I believe it can easily be overcome:
<br />
When matching forward, the code updating the nodes of the automaton has to
prefer paths favouring the greedy patterns occurring earlier in the regexp: it
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
(e.g. `abcd|012345`).



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
support for bracket expressions (e.g. `[:digit:]`)
* It only supports ASCII characters, but not wide character types.



### Benchmarks

The results below were produced on a machine with a quad-core Intel Core
i5-2400 CPU @ 3.10GHz with 4GiB RAM on Fedora 17 (3.8.13-100.fc17.x86\_64).
It supports SSE4.2.

Results are reported for the following engines versions:

    GNU grep version 2.14, commit: 599f2b15bc152cdf19022c7803f9a5f336c25e65
    Rejit commit: 6cd1ca642783ac9968085682b1a82b6961ad9109
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

`jrep` is a grep-like utility powered by Rejit.

The commands used to benchmarks follow the form below.
In both cases the grep program is run a first time without measurement to give
the OS a chance to preload the files. Without this we would be benchmarking the
time to access the files.

    $ REGEXP="re"; CMD="jrep -R $REGEXP /work/linux/linux-3.10.6 "; \
    $CMD > /dev/null && time $CMD > /tmp/res

    $ REGEXP="re"; CMD="grep -R $REGEXP /work/linux/linux-3.10.6 "; \
    $CMD > /dev/null && time $CMD > /dev/null

For concision, the REGEXP variable is reminded only in the extended regular
expression (ERE) format. (Grep uses BRE.)

<table style="width:100%; margin-top:20px;">
  <tr>
    <td>
      <strong>GNU grep</strong>
    </td>
    <td>
      <strong>jrep</strong>
    </td>
    <td>
      <strong>Speedup</strong>
      (time for grep / time for jrep)
    </td>
  </tr>
  <tr>
    <td colspan="3">
      <strong>Regexp</strong>: <code>regexp</code>
    </td>
  </tr>
  <tr>
    <td>
      <pre style="width:80%"><code>real  0m0.611s
user  0m0.346s
sys   0m0.259s</code></pre>
    </td>
    <td>
      <pre style="width:80%"><code>real  0m0.373s
user  0m0.112s
sys   0m0.257s</code></pre>
    </td>
    <td>
      <span style="color:green">1.64</span>
    </td>
  </tr>
  <tr>
    <td colspan="3">
      <strong>Regexp</strong>: <code>"(\s\+void.*\(\))|(union.*\{)"</code>
    </td>
  </tr>
  <tr>
    <td>
      <pre style="width:80%"><code>real  0m1.702s
user  0m1.422s
sys   0m0.272s</code></pre>
    </td>
    <td>
      <pre style="width:80%"><code>real  0m0.523s
user  0m0.224s
sys   0m0.293s</code></pre>
    </td>
    <td>
      <span style="color:green">3.25</span>
    </td>
  </tr>
</table>

The `jrep` utility performs significantly faster than grep in this very real
use-cases!  To see benchmark results and comments for a wider range of regular
expressions, please see [this article][grep jrep benchs article].

`jrep` is part of the sample programs in the rejit repository (see the
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
        The text is randomly generated from characters in the range <code>[b-z]</code>, so the
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
          var data_0_chars_null_density_re2_amortised = [[8,1.86047e+07],[16,3.72093e+07],[32,7.27273e+07],[64,1.45455e+08],[128,2.90909e+08],[256,5.12e+08],[512,1.11304e+09],[1024,2.13333e+09],[2048,3.01176e+09],[4096,6.4e+09],[8192,9.99024e+09],[16384,1.30032e+10],[32768,1.51005e+10],[65536,1.5384e+10],[131072,1.4278e+10],[262144,2.92898e+10],[524288,2.40609e+10],[1048576,2.63726e+10],[2097152,2.64092e+10],[4194304,2.52426e+10],[8388608,1.08875e+10],[16777216,9.26447e+09],];
          var data_0_chars_null_density_re2_best = [[8,2.35294e+07],[16,4.84848e+07],[32,9.14286e+07],[64,1.88235e+08],[128,3.76471e+08],[256,6.2439e+08],[512,1.38378e+09],[1024,2.62564e+09],[2048,3.47119e+09],[4096,7.44727e+09],[8192,1.12219e+10],[16384,1.41241e+10],[32768,1.583e+10],[65536,1.57161e+10],[131072,1.44512e+10],[262144,2.94544e+10],[524288,2.41163e+10],[1048576,2.64058e+10],[2097152,2.64258e+10],[4194304,2.52517e+10],[8388608,1.08892e+10],[16777216,9.26514e+09],];
          var data_0_chars_null_density_re2_worst = [[8,434311],[16,875274],[32,1.72136e+06],[64,3.452e+06],[128,7.00602e+06],[256,1.39358e+07],[512,2.76906e+07],[1024,5.54113e+07],[2048,1.1179e+08],[4096,2.20571e+08],[8192,4.3482e+08],[16384,8.60956e+08],[32768,1.61578e+09],[65536,3.24596e+09],[131072,4.70636e+09],[262144,7.03176e+09],[524288,1.63076e+10],[1048576,2.12822e+10],[2097152,2.34319e+10],[4194304,2.29372e+10],[8388608,1.0457e+10],[16777216,9.11013e+09],];
          var data_0_chars_null_density_rejit_amortised = [[8,2e+07],[16,3.2e+07],[32,6.95652e+07],[64,1.3617e+08],[128,2.66667e+08],[256,4.92308e+08],[512,9.48148e+08],[1024,1.76552e+09],[2048,2.84444e+09],[4096,4.35745e+09],[8192,5.42517e+09],[16384,6.85523e+09],[32768,7.44727e+09],[65536,7.94376e+09],[131072,8.20739e+09],[262144,8.30358e+09],[524288,1.75818e+10],[1048576,1.7485e+10],[2097152,1.75201e+10],[4194304,1.73973e+10],[8388608,1.04078e+10],[16777216,9.02015e+09],];
          var data_0_chars_null_density_rejit_best = [[8,4e+07],[16,7.27273e+07],[32,1.28e+08],[64,2.56e+08],[128,5.12e+08],[256,9.84615e+08],[512,1.70667e+09],[1024,2.84444e+09],[2048,4.096e+09],[4096,5.53514e+09],[8192,6.25344e+09],[16384,7.5156e+09],[32768,7.97275e+09],[65536,8.15124e+09],[131072,8.31675e+09],[262144,8.36719e+09],[524288,1.76409e+10],[1048576,1.75201e+10],[2097152,1.75347e+10],[4194304,1.74059e+10],[8388608,1.04105e+10],[16777216,9.02122e+09],];
          var data_0_chars_null_density_rejit_worst = [[8,361174],[16,730927],[32,1.45521e+06],[64,2.89331e+06],[128,5.84475e+06],[256,1.1647e+07],[512,2.322e+07],[1024,4.64189e+07],[2048,9.27116e+07],[4096,1.8245e+08],[8192,3.56329e+08],[16384,6.77025e+08],[32768,1.25886e+09],[65536,2.15366e+09],[131072,3.31576e+09],[262144,4.49184e+09],[524288,1.21532e+10],[1048576,1.35545e+10],[2097152,1.57078e+10],[4194304,1.59868e+10],[8388608,9.72299e+09],[16777216,8.77295e+09],];
          var data_0_chars_null_density_v8_amortised = [[8,80000000],[16,106666666.66666666],[32,320000000],[64,640000000],[128,1280000000],[256,1706666666.6666665],[512,3413333333.333333],[1024,10240000000],[2048,10240000000],[4096,20480000000],[8192,27306666666.666664],[16384,32768000000],[32768,38550588235.29411],[65536,36408888888.888885],[131072,13107200000],[262144,26214400000],[524288,26214400000],[1048576,26214400000],[2097152,23301688888.888885],[4194304,24672376470.588234],[8388608,10618491139.240507],[16777216,9167877595.628414],];
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
          var data_1_chars_average_density_re2_amortised = [[8,1.90476e+07],[16,3.72093e+07],[32,7.44186e+07],[64,1.33333e+08],[128,2.61224e+08],[256,4.33898e+08],[512,6.2439e+08],[1024,8.60504e+08],[2048,1.00392e+09],[4096,1.15706e+09],[8192,1.27403e+09],[16384,1.26031e+09],[32768,1.05026e+09],[65536,1.72327e+09],[131072,1.5671e+09],[262144,1.49591e+09],[524288,1.44241e+09],[1048576,1.43869e+09],[2097152,1.43438e+09],[4194304,1.44283e+09],[8388608,1.41167e+09],[16777216,1.40975e+09],];
          var data_1_chars_average_density_re2_best = [[8,2.42424e+07],[16,4.70588e+07],[32,9.69697e+07],[64,1.68421e+08],[128,3.28205e+08],[256,5.22449e+08],[512,7.11111e+08],[1024,9.3945e+08],[2048,1.05026e+09],[4096,1.18725e+09],[8192,1.29415e+09],[16384,1.26909e+09],[32768,1.05363e+09],[65536,1.72554e+09],[131072,1.56803e+09],[262144,1.49634e+09],[524288,1.44261e+09],[1048576,1.43881e+09],[2097152,1.43444e+09],[4194304,1.44286e+09],[8388608,1.4117e+09],[16777216,1.40976e+09],];
          var data_1_chars_average_density_re2_worst = [[8,433839],[16,874317],[32,1.73819e+06],[64,3.40788e+06],[128,6.81213e+06],[256,1.33264e+07],[512,2.44391e+07],[1024,4.65878e+07],[2048,8.79725e+07],[4096,1.57357e+08],[8192,2.82385e+08],[16384,4.40667e+08],[32768,5.71468e+08],[65536,6.26899e+08],[131072,1.13531e+09],[262144,1.31308e+09],[524288,1.36073e+09],[1048576,1.40368e+09],[2097152,1.41815e+09],[4194304,1.42249e+09],[8388608,1.4027e+09],[16777216,1.40691e+09],];
          var data_1_chars_average_density_rejit_amortised = [[8,1.86047e+07],[16,3.72093e+07],[32,5.42373e+07],[64,1.3617e+08],[128,2.61224e+08],[256,4.92308e+08],[512,9.14286e+08],[1024,1.67869e+09],[2048,2.0898e+09],[4096,3.53103e+09],[8192,4.65455e+09],[16384,5.44319e+09],[32768,5.93623e+09],[65536,6.24747e+09],[131072,9.05193e+09],[262144,1.19156e+10],[524288,1.15865e+10],[1048576,1.18309e+10],[2097152,1.15679e+10],[4194304,1.14339e+10],[8388608,9.31933e+09],[16777216,8.7235e+09],];
          var data_1_chars_average_density_rejit_best = [[8,3.80952e+07],[16,6.95652e+07],[32,1.23077e+08],[64,2.56e+08],[128,5.12e+08],[256,9.14286e+08],[512,1.65161e+09],[1024,2.62564e+09],[2048,2.76757e+09],[4096,4.26667e+09],[8192,5.28516e+09],[16384,5.8306e+09],[32768,6.171e+09],[65536,6.3751e+09],[131072,9.19158e+09],[262144,1.197e+10],[524288,1.16199e+10],[1048576,1.18456e+10],[2097152,1.1575e+10],[4194304,1.14377e+10],[8388608,9.3215e+09],[16777216,8.7245e+09],];
          var data_1_chars_average_density_rejit_worst = [[8,358744],[16,716846],[32,1.46453e+06],[64,2.8907e+06],[128,5.81818e+06],[256,1.16152e+07],[512,2.34004e+07],[1024,4.61054e+07],[2048,9.12656e+07],[4096,1.79886e+08],[8192,3.4756e+08],[16384,6.55622e+08],[32768,1.20737e+09],[65536,1.99562e+09],[131072,2.93818e+09],[262144,7.91737e+09],[524288,7.91378e+09],[1048576,1.01194e+10],[2097152,1.06921e+10],[4194304,1.08833e+10],[8388608,8.79927e+09],[16777216,8.48714e+09],];
          var data_1_chars_average_density_v8_amortised = [[8,53333333.33333333],[16,160000000],[32,320000000],[64,426666666.6666666],[128,853333333.3333333],[256,1280000000],[512,1462857142.857143],[1024,2048000000],[2048,1638400000],[4096,1606274509.8039217],[8192,1531214953.271028],[16384,1575384615.3846157],[32768,1527645687.6456878],[65536,1510046082.9493086],[131072,1456355555.5555553],[262144,1456355555.5555553],[524288,1456355555.5555553],[1048576,1476867605.633803],[2097152,1456355555.5555553],[4194304,1461429965.1567943],[8388608,1438869296.7409947],[16777216,1427848170.212766],];
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
          var data_2_chars_high_density_re2_amortised = [[8,1.73913e+07],[16,3.47826e+07],[32,6.4e+07],[64,1.10345e+08],[128,1.66234e+08],[256,2.66667e+08],[512,4.12903e+08],[1024,4.14575e+08],[2048,5.12e+08],[4096,4.97087e+08],[8192,4.61521e+08],[16384,4.06047e+08],[32768,7.2208e+08],[65536,6.99872e+08],[131072,6.94348e+08],[262144,6.82152e+08],[524288,6.73373e+08],[1048576,6.72751e+08],[2097152,6.7318e+08],[4194304,6.73045e+08],[8388608,6.65113e+08],[16777216,6.65597e+08],];
          var data_2_chars_high_density_re2_best = [[8,2.16216e+07],[16,4.44444e+07],[32,8e+07],[64,1.33333e+08],[128,1.88235e+08],[256,2.97674e+08],[512,4.45217e+08],[1024,4.32068e+08],[2048,5.23785e+08],[4096,5.03194e+08],[8192,4.64136e+08],[16384,4.07056e+08],[32768,7.22877e+08],[65536,7.00171e+08],[131072,6.94568e+08],[262144,6.82258e+08],[524288,6.73425e+08],[1048576,6.72772e+08],[2097152,6.73193e+08],[4194304,6.73053e+08],[8388608,6.6512e+08],[16777216,6.656e+08],];
          var data_2_chars_high_density_re2_worst = [[8,423057],[16,859753],[32,1.68955e+06],[64,3.1746e+06],[128,5.72707e+06],[256,1.0889e+07],[512,2.1044e+07],[1024,3.76056e+07],[2048,7.04264e+07],[4096,1.20506e+08],[8192,1.8409e+08],[16384,2.33158e+08],[32768,2.58077e+08],[65536,5.60472e+08],[131072,6.06955e+08],[262144,6.395e+08],[524288,6.49595e+08],[1048576,6.62097e+08],[2097152,6.68343e+08],[4194304,6.70006e+08],[8388608,6.63343e+08],[16777216,6.64633e+08],];
          var data_2_chars_high_density_rejit_amortised = [[8,3.47826e+07],[16,6.66667e+07],[32,6.53061e+07],[64,1.3913e+08],[128,2.46154e+08],[256,4.33898e+08],[512,8.12698e+08],[1024,1.31282e+09],[2048,2.20215e+09],[4096,2.86434e+09],[8192,3.45654e+09],[16384,4.08579e+09],[32768,4.25006e+09],[65536,4.4311e+09],[131072,4.39102e+09],[262144,8.28783e+09],[524288,7.07064e+09],[1048576,6.82223e+09],[2097152,6.78932e+09],[4194304,6.70852e+09],[8388608,6.16094e+09],[16777216,6.13505e+09],];
          var data_2_chars_high_density_rejit_best = [[8,7.27273e+07],[16,1.23077e+08],[32,1.18519e+08],[64,2.66667e+08],[128,4.57143e+08],[256,7.31429e+08],[512,1.31282e+09],[1024,1.8963e+09],[2048,2.88451e+09],[4096,3.33008e+09],[8192,3.81023e+09],[16384,4.31158e+09],[32768,4.36907e+09],[65536,4.49492e+09],[131072,4.42213e+09],[262144,8.31939e+09],[524288,7.08115e+09],[1048576,6.82711e+09],[2097152,6.79174e+09],[4194304,6.71003e+09],[8388608,6.16189e+09],[16777216,6.13557e+09],];
          var data_2_chars_high_density_rejit_worst = [[8,440286],[16,1.26283e+06],[32,1.46453e+06],[64,2.82935e+06],[128,5.79186e+06],[256,1.14439e+07],[512,2.30943e+07],[1024,4.57347e+07],[2048,9.03795e+07],[4096,1.78787e+08],[8192,3.34095e+08],[16384,6.30154e+08],[32768,1.10442e+09],[65536,1.74391e+09],[131072,2.42771e+09],[262144,3.73744e+09],[524288,5.62239e+09],[1048576,6.05658e+09],[2097152,6.45297e+09],[4194304,6.45446e+09],[8388608,5.90984e+09],[16777216,5.94561e+09],];
          var data_2_chars_high_density_v8_amortised = [[8,80000000],[16,160000000],[32,320000000],[64,426666666.6666666],[128,853333333.3333333],[256,640000000],[512,930909090.9090909],[1024,1024000000],[2048,1170285714.2857141],[4096,1137777777.7777777],[8192,1161985815.6028368],[16384,1149754385.9649124],[32768,1086832504.145937],[65536,982548725.6371814],[131072,936228571.4285715],[262144,936228571.4285715],[524288,936228571.4285715],[1048576,936228571.4285715],[2097152,940426905.8295963],[4194304,938323042.5055928],[8388608,923855506.6079296],[16777216,1843650109.89011],];
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
          var data_3_complex_simple_ff_re2_amortised = [[8,7.76699e+06],[16,1.36752e+07],[32,4.63768e+07],[64,4.74074e+07],[128,6.4e+07],[256,7.85276e+07],[512,8.61953e+07],[1024,1.54683e+08],[2048,1.49708e+08],[4096,1.59688e+08],[8192,1.63317e+08],[16384,1.6335e+08],[32768,1.6487e+08],[65536,1.66534e+08],[131072,1.66551e+08],[262144,1.66966e+08],[524288,1.6648e+08],[1048576,1.66761e+08],[2097152,1.66937e+08],[4194304,1.66947e+08],[8388608,1.66997e+08],[16777216,1.67016e+08],];
          var data_3_complex_simple_ff_re2_best = [[8,1.48148e+07],[16,2.38806e+07],[32,6.95652e+07],[64,6.53061e+07],[128,7.85276e+07],[256,8.98246e+07],[512,9.27536e+07],[1024,1.60754e+08],[2048,1.52608e+08],[4096,1.6126e+08],[8192,1.64168e+08],[16384,1.63807e+08],[32768,1.65103e+08],[65536,1.66669e+08],[131072,1.66616e+08],[262144,1.66999e+08],[524288,1.66496e+08],[1048576,1.6677e+08],[2097152,1.66941e+08],[4194304,1.66949e+08],[8388608,1.66999e+08],[16777216,1.67017e+08],];
          var data_3_complex_simple_ff_re2_worst = [[8,106087],[16,195623],[32,666806],[64,774443],[128,1.26532e+06],[256,2.08639e+06],[512,3.21628e+06],[1024,5.41112e+06],[2048,1.28474e+07],[4096,2.19695e+07],[8192,3.63378e+07],[16384,5.20077e+07],[32768,7.33016e+07],[65536,9.68307e+07],[131072,1.1895e+08],[262144,1.35506e+08],[524288,1.48164e+08],[1048576,1.56432e+08],[2097152,1.61398e+08],[4194304,1.63983e+08],[8388608,1.65167e+08],[16777216,1.65986e+08],];
          var data_3_complex_simple_ff_rejit_amortised = [[8,4.65116e+06],[16,9.1954e+06],[32,1.87135e+07],[64,3.69942e+07],[128,1.52381e+08],[256,3.12195e+08],[512,4.65455e+08],[1024,7.58519e+08],[2048,1.24121e+09],[4096,1.923e+09],[8192,2.61725e+09],[16384,7.58519e+09],[32768,8.85622e+09],[65536,8.40205e+09],[131072,8.23834e+09],[262144,1.25668e+10],[524288,1.2264e+10],[1048576,1.1758e+10],[2097152,1.15133e+10],[4194304,1.1489e+10],[8388608,9.2464e+09],[16777216,8.66623e+09],];
          var data_3_complex_simple_ff_rejit_best = [[8,1.11111e+07],[16,2.16216e+07],[32,4.21053e+07],[64,8.53333e+07],[128,3.45946e+08],[256,6.91892e+08],[512,1.024e+09],[1024,1.55152e+09],[2048,2.30112e+09],[4096,3.25079e+09],[8192,3.77512e+09],[16384,9.58129e+09],[32768,1.01764e+10],[65536,9.20449e+09],[131072,8.61749e+09],[262144,1.28502e+10],[524288,1.24033e+10],[1048576,1.18229e+10],[2097152,1.15444e+10],[4194304,1.15057e+10],[8388608,9.25292e+09],[16777216,8.66923e+09],];
          var data_3_complex_simple_ff_rejit_worst = [[8,79051.4],[16,158825],[32,316926],[64,633663],[128,1.35478e+06],[256,5.26749e+06],[512,8.26606e+06],[1024,1.40409e+07],[2048,2.78639e+07],[4096,4.88084e+07],[8192,8.35066e+07],[16384,1.73836e+08],[32768,6.44405e+08],[65536,1.01167e+09],[131072,1.51214e+09],[262144,3.01384e+09],[524288,5.44149e+09],[1048576,7.24805e+09],[2097152,8.8637e+09],[4194304,9.75215e+09],[8388608,8.25699e+09],[16777216,8.19696e+09],];
          var data_3_complex_simple_ff_v8_amortised = [[8,80000000],[16,80000000],[32,213333333.3333333],[64,256000000],[128,284444444.4444444],[256,301176470.58823526],[512,284444444.4444444],[1024,252839506.1728395],[2048,210051282.05128208],[4096,183677130.04484305],[8192,168213552.3613963],[16384,162217821.78217822],[32768,156224076.28128725],[65536,156205458.22905496],[131072,154202352.94117644],[262144,156038095.23809522],[524288,155114792.89940828],[1048576,155114792.89940828],[2097152,155690571.640683],[4194304,155748384.70107687],[8388608,155315830.40177745],[16777216,155402148.94405335],];
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
        <strong>Regexp 5:</strong> <code>(12345678|abcdefghijkl)</code>
        <p>
        Search for an alternation of two long words with no common sub-string.
        </p>
        <p>
        V8 performs superbly here (it does not use SIMD), with a
        Boyer-Moore-Horspool algorithm.
        As this performs so well, it is surprising that the same mechanism
        appears not to be used for single words (benchmarks 2 and 3).
        </p>
        <p>
        Rejit looks for both strings in parallel. The performance is about
        half the performance for regexp 2.
        </p>
      </div>
    </td>
  </tr>
  <tr>
    <td style="padding-bottom:50px;">
      <div>
        <div class="graph" id="plot_parallel_4_alternation_2_long" style="float:left;"> </div>
        <div style="float:left;"> <ul id="plot_parallel_4_alternation_2_long_choices" style="list-style: none;" class="flot_choices"> </ul> </div>
      </div>
      <script type="text/javascript">
        <!--
        $(function () {
          var data_4_alternation_2_long_re2_amortised = [[8,1.45455e+07],[16,2.58065e+07],[32,4.26667e+07],[64,6.33663e+07],[128,7.75758e+07],[256,9.73384e+07],[512,1.07563e+08],[1024,1.14413e+08],[2048,1.17634e+08],[4096,1.19977e+08],[8192,2.47492e+08],[16384,2.22912e+08],[32768,2.57549e+08],[65536,2.59765e+08],[131072,2.59626e+08],[262144,2.59783e+08],[524288,2.59198e+08],[1048576,2.59341e+08],[2097152,2.59646e+08],[4194304,2.59397e+08],[8388608,2.57808e+08],[16777216,2.57794e+08],];
          var data_4_alternation_2_long_re2_best = [[8,2e+07],[16,3.40426e+07],[32,5.2459e+07],[64,7.35632e+07],[128,8.53333e+07],[256,1.03226e+08],[512,1.11063e+08],[1024,1.16364e+08],[2048,1.18587e+08],[4096,1.20506e+08],[8192,2.48017e+08],[16384,2.23185e+08],[32768,2.57691e+08],[65536,2.59847e+08],[131072,2.59667e+08],[262144,2.59803e+08],[524288,2.59208e+08],[1048576,2.59346e+08],[2097152,2.59648e+08],[4194304,2.59398e+08],[8388608,2.57809e+08],[16777216,2.57795e+08],];
          var data_4_alternation_2_long_re2_worst = [[8,305810],[16,617046],[32,1.15523e+06],[64,2.23933e+06],[128,4.2953e+06],[256,8.16066e+06],[512,1.42857e+07],[1024,2.49756e+07],[2048,4.02042e+07],[4096,5.93623e+07],[8192,1.64432e+08],[16384,1.78261e+08],[32768,2.11666e+08],[65536,2.31863e+08],[131072,2.44647e+08],[262144,2.51259e+08],[524288,2.54939e+08],[1048576,2.57678e+08],[2097152,2.58589e+08],[4194304,2.59064e+08],[8388608,2.57488e+08],[16777216,2.57649e+08],];
          var data_4_alternation_2_long_rejit_amortised = [[8,1.86047e+07],[16,3.33333e+07],[32,5.51724e+07],[64,9.01408e+07],[128,1.85507e+08],[256,4.26667e+08],[512,7.75758e+08],[1024,1.1907e+09],[2048,1.46286e+09],[4096,2.1672e+09],[8192,2.56e+09],[16384,3.02847e+09],[32768,3.15684e+09],[65536,3.40093e+09],[131072,3.433e+09],[262144,6.52262e+09],[524288,6.61813e+09],[1048576,6.31634e+09],[2097152,6.28831e+09],[4194304,6.27054e+09],[8388608,5.77696e+09],[16777216,5.75252e+09],];
          var data_4_alternation_2_long_rejit_best = [[8,4.44444e+07],[16,6.95652e+07],[32,1e+08],[64,1.42222e+08],[128,2.84444e+08],[256,7.11111e+08],[512,1.21905e+09],[1024,1.73559e+09],[2048,1.81239e+09],[4096,2.49756e+09],[8192,2.77695e+09],[16384,3.18755e+09],[32768,3.23475e+09],[65536,3.44745e+09],[131072,3.45654e+09],[262144,6.54542e+09],[524288,6.629e+09],[1048576,6.32091e+09],[2097152,6.29076e+09],[4194304,6.27195e+09],[8388608,5.77803e+09],[16777216,5.75301e+09],];
          var data_4_alternation_2_long_rejit_worst = [[8,293902],[16,596570],[32,1.19181e+06],[64,2.37653e+06],[128,4.72325e+06],[256,9.32945e+06],[512,1.85979e+07],[1024,3.69275e+07],[2048,7.28826e+07],[4096,1.43117e+08],[8192,2.71528e+08],[16384,5.06774e+08],[32768,8.8015e+08],[65536,1.40004e+09],[131072,1.93893e+09],[262144,4.89806e+09],[524288,5.03107e+09],[1048576,5.66308e+09],[2097152,5.91013e+09],[4194304,5.97054e+09],[8388608,5.53258e+09],[16777216,5.59858e+09],];
          var data_4_alternation_2_long_v8_amortised = [[8,80000000],[16,160000000],[32,320000000],[64,640000000],[128,1280000000],[256,1706666666.6666665],[512,3413333333.333333],[1024,4096000000],[2048,4096000000],[4096,5120000000],[8192,5285161290.32258],[16384,5285161290.32258],[32768,3788208092.4855485],[65536,2590355731.2252965],[131072,2184533333.333333],[262144,2184533333.333333],[524288,2184533333.333333],[1048576,2139951020.4081633],[2097152,2162012371.134021],[4194304,2184533333.333333],[8388608,2118335353.5353537],[16777216,2112999496.2216623],];
            var datasets = [ {data: data_4_alternation_2_long_re2_amortised, label: "re2_amortised", color: "#DEBD00", },{data: data_4_alternation_2_long_re2_best, label: "re2_best", color: "#E0D48D", },{data: data_4_alternation_2_long_re2_worst, label: "re2_worst", color: "#E0D48D", },{data: data_4_alternation_2_long_rejit_amortised, label: "rejit_amortised", color: "#277AD9", },{data: data_4_alternation_2_long_rejit_best, label: "rejit_best", color: "#94B8E0", },{data: data_4_alternation_2_long_rejit_worst, label: "rejit_worst", color: "#94B8E0", },{data: data_4_alternation_2_long_v8_amortised, label: "v8_amortised", color: "#00940A", }, ];
            var choiceContainer = $("#plot_parallel_4_alternation_2_long_choices");
            $.each(datasets, function(key, val) {
               choiceContainer.append('<li style="list-style: none;"><input type="checkbox" name="' + key +
                                      '" checked="checked" id="id' + key + '">' +
                                      '<label for="id' + key + '">'
                                      + val.label + '</label></li>');
            });
            plot_according_to_choices("plot_parallel_4_alternation_2_long", datasets, choiceContainer);
            $("#plot_parallel_4_alternation_2_long").bind("plothover", plothover_func);
            function replot() { plot_according_to_choices("plot_parallel_4_alternation_2_long", datasets, choiceContainer); }
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
        <strong>Regexp 6:</strong> <code>(12345678|xyz)</code>
        <p>
        Search for an alternation of two words with no common sub-string,
        including one short word.
        </p>
        <p>
        v8 still does very well, but the short string does not allow for code
        as efficient as before.
        </p>
        <p>
        Rejit's brute force algorithm still enjoys the SIMD speed-up and shows 
        more stable performance.
        </p>
      </div>
    </td>
  </tr>
  <tr>
    <td style="padding-bottom:50px;">
      <div>
        <div class="graph" id="plot_parallel_5_alternation_2_short" style="float:left;"> </div>
        <div style="float:left;"> <ul id="plot_parallel_5_alternation_2_short_choices" style="list-style: none;" class="flot_choices"> </ul> </div>
      </div>
      <script type="text/javascript">
        <!--
        $(function () {
          var data_5_alternation_2_short_re2_amortised = [[8,1.53846e+07],[16,2.75862e+07],[32,4e+07],[64,6.46465e+07],[128,8.70748e+07],[256,9.92248e+07],[512,1.11547e+08],[1024,1.17701e+08],[2048,1.19556e+08],[4096,1.21687e+08],[8192,1.66877e+08],[16384,2.36524e+08],[32768,2.54647e+08],[65536,2.59775e+08],[131072,2.59775e+08],[262144,2.59788e+08],[524288,2.55849e+08],[1048576,2.59751e+08],[2097152,2.59413e+08],[4194304,2.59689e+08],[8388608,2.57857e+08],[16777216,2.57999e+08],];
          var data_5_alternation_2_short_re2_best = [[8,2e+07],[16,3.55556e+07],[32,5.33333e+07],[64,7.44186e+07],[128,9.55224e+07],[256,1.09871e+08],[512,1.14798e+08],[1024,1.19487e+08],[2048,1.20471e+08],[4096,1.22159e+08],[8192,1.6732e+08],[16384,2.36763e+08],[32768,2.54786e+08],[65536,2.59837e+08],[131072,2.59811e+08],[262144,2.59806e+08],[524288,2.55858e+08],[1048576,2.59756e+08],[2097152,2.59416e+08],[4194304,2.59691e+08],[8388608,2.57859e+08],[16777216,2.58e+08],];
          var data_5_alternation_2_short_re2_worst = [[8,365798],[16,724966],[32,1.39983e+06],[64,2.63591e+06],[128,5.20961e+06],[256,1.00708e+07],[512,1.83119e+07],[1024,3.14496e+07],[2048,5.00366e+07],[4096,7.11853e+07],[8192,8.99034e+07],[16384,2.0906e+08],[32768,2.22609e+08],[65536,2.38486e+08],[131072,2.49362e+08],[262144,2.5404e+08],[524288,2.53884e+08],[1048576,2.58837e+08],[2097152,2.58878e+08],[4194304,2.59338e+08],[8388608,2.57674e+08],[16777216,2.57897e+08],];
          var data_5_alternation_2_short_rejit_amortised = [[8,1.95122e+07],[16,3.40426e+07],[32,5.92593e+07],[64,9.27536e+07],[128,1.85507e+08],[256,3.50685e+08],[512,6.5641e+08],[1024,1.10108e+09],[2048,1.8789e+09],[4096,2.22609e+09],[8192,2.78639e+09],[16384,3.10303e+09],[32768,3.28008e+09],[65536,3.36082e+09],[131072,7.1624e+09],[262144,5.52464e+09],[524288,6.91035e+09],[1048576,6.4318e+09],[2097152,6.20661e+09],[4194304,6.18273e+09],[8388608,5.66079e+09],[16777216,5.63785e+09],];
          var data_5_alternation_2_short_rejit_best = [[8,4.70588e+07],[16,7.27273e+07],[32,1.10345e+08],[64,1.45455e+08],[128,2.90909e+08],[256,5.33333e+08],[512,9.48148e+08],[1024,1.55152e+09],[2048,2.46747e+09],[4096,2.56e+09],[8192,3.04535e+09],[16384,3.25726e+09],[32768,3.36427e+09],[65536,3.40447e+09],[131072,7.21365e+09],[262144,5.5445e+09],[524288,6.92221e+09],[1048576,6.43693e+09],[2097152,6.209e+09],[4194304,6.18446e+09],[8388608,5.66174e+09],[16777216,5.63834e+09],];
          var data_5_alternation_2_short_rejit_worst = [[8,307220],[16,604002],[32,1.21304e+06],[64,2.44929e+06],[128,4.74074e+06],[256,9.74496e+06],[512,1.91402e+07],[1024,3.79822e+07],[2048,7.57116e+07],[4096,1.48567e+08],[8192,2.819e+08],[16384,5.18809e+08],[32768,9.04943e+08],[65536,1.41761e+09],[131072,4.05796e+09],[262144,4.12371e+09],[524288,5.36191e+09],[1048576,5.79676e+09],[2097152,5.81799e+09],[4194304,5.89303e+09],[8388608,5.39606e+09],[16777216,5.47634e+09],];
          var data_5_alternation_2_short_v8_amortised = [[8,80000000],[16,320000000],[32,640000000],[64,426666666.6666666],[128,853333333.3333333],[256,1280000000],[512,1706666666.6666665],[1024,2275555555.5555553],[2048,2560000000],[4096,2642580645.16129],[8192,2445373134.328358],[16384,2357410071.942446],[32768,2114064516.1290321],[65536,1971007518.7969928],[131072,1872457142.857143],[262144,1872457142.857143],[524288,1807889655.1724136],[1048576,1839607017.5438595],[2097152,1823610434.7826085],[4194304,1831573799.1266375],[8388608,1788615778.251599],[16777216,1779132131.495228],];
            var datasets = [ {data: data_5_alternation_2_short_re2_amortised, label: "re2_amortised", color: "#DEBD00", },{data: data_5_alternation_2_short_re2_best, label: "re2_best", color: "#E0D48D", },{data: data_5_alternation_2_short_re2_worst, label: "re2_worst", color: "#E0D48D", },{data: data_5_alternation_2_short_rejit_amortised, label: "rejit_amortised", color: "#277AD9", },{data: data_5_alternation_2_short_rejit_best, label: "rejit_best", color: "#94B8E0", },{data: data_5_alternation_2_short_rejit_worst, label: "rejit_worst", color: "#94B8E0", },{data: data_5_alternation_2_short_v8_amortised, label: "v8_amortised", color: "#00940A", }, ];
            var choiceContainer = $("#plot_parallel_5_alternation_2_short_choices");
            $.each(datasets, function(key, val) {
               choiceContainer.append('<li style="list-style: none;"><input type="checkbox" name="' + key +
                                      '" checked="checked" id="id' + key + '">' +
                                      '<label for="id' + key + '">'
                                      + val.label + '</label></li>');
            });
            plot_according_to_choices("plot_parallel_5_alternation_2_short", datasets, choiceContainer);
            $("#plot_parallel_5_alternation_2_short").bind("plothover", plothover_func);
            function replot() { plot_according_to_choices("plot_parallel_5_alternation_2_short", datasets, choiceContainer); }
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
        <strong>Regexp 7:</strong> <code>(abcd---|abcd___)</code>
        <p>
        Search for an alternation of two words sharing a common prefix.
        </p>
        <p>
        Re2 spots the common prefix and uses the information to speed up the search,
        probably by interpreting it as <code>abcd(---|___)</code>.
        Rejit and (apparently) V8 do not do that yet. Such an optimisation would
        enable performance similar to searching for the single common prefix.
        <br />Rejit could highly benefit from a generalised version of this
        optimisation looking for common sub-regexp and applied to find a better
        dominant sub-regexp.
        </p>
      </div>
    </td>
  </tr>
  <tr>
    <td style="padding-bottom:50px;">
      <div>
        <div class="graph" id="plot_parallel_6_alternation_2_common_prefix" style="float:left;"> </div>
        <div style="float:left;"> <ul id="plot_parallel_6_alternation_2_common_prefix_choices" style="list-style: none;" class="flot_choices"> </ul> </div>
      </div>
      <script type="text/javascript">
        <!--
        $(function () {
          var data_6_alternation_2_common_prefix_re2_amortised = [[8,1.56863e+07],[16,3.2e+07],[32,6.4e+07],[64,1.20755e+08],[128,2.28571e+08],[256,4e+08],[512,5.88506e+08],[1024,8.12698e+08],[2048,1.01386e+09],[4096,1.13778e+09],[8192,1.27205e+09],[16384,1.29008e+09],[32768,1.02785e+09],[65536,1.07489e+09],[131072,1.51546e+09],[262144,1.51214e+09],[524288,1.45176e+09],[1048576,1.44637e+09],[2097152,1.4404e+09],[4194304,1.44584e+09],[8388608,1.41863e+09],[16777216,1.41723e+09],];
          var data_6_alternation_2_common_prefix_re2_best = [[8,2.28571e+07],[16,4.70588e+07],[32,9.41176e+07],[64,1.72973e+08],[128,3.2e+08],[256,5.33333e+08],[512,7.21127e+08],[1024,9.30909e+08],[2048,1.10108e+09],[4096,1.1907e+09],[8192,1.30238e+09],[16384,1.30654e+09],[32768,1.03336e+09],[65536,1.07772e+09],[131072,1.51721e+09],[262144,1.51292e+09],[524288,1.45212e+09],[1048576,1.44655e+09],[2097152,1.4405e+09],[4194304,1.4459e+09],[8388608,1.41868e+09],[16777216,1.41725e+09],];
          var data_6_alternation_2_common_prefix_re2_worst = [[8,313849],[16,637196],[32,1.2549e+06],[64,2.49513e+06],[128,4.98831e+06],[256,9.89181e+06],[512,1.9076e+07],[1024,3.66107e+07],[2048,6.93297e+07],[4096,1.30779e+08],[8192,2.38695e+08],[16384,3.9056e+08],[32768,5.17335e+08],[65536,5.87293e+08],[131072,1.20904e+09],[262144,1.31295e+09],[524288,1.34161e+09],[1048576,1.40652e+09],[2097152,1.41922e+09],[4194304,1.43175e+09],[8388608,1.41003e+09],[16777216,1.41206e+09],];
          var data_6_alternation_2_common_prefix_rejit_amortised = [[8,1.95122e+07],[16,3.55556e+07],[32,6.15385e+07],[64,1.03226e+08],[128,2.03175e+08],[256,4.33898e+08],[512,7.0137e+08],[1024,1.23373e+09],[2048,1.44225e+09],[4096,2.11134e+09],[8192,2.56e+09],[16384,3.03407e+09],[32768,3.2094e+09],[65536,3.38512e+09],[131072,3.61578e+09],[262144,6.67373e+09],[524288,6.98585e+09],[1048576,6.47309e+09],[2097152,6.27946e+09],[4194304,6.28078e+09],[8388608,5.70494e+09],[16777216,5.7284e+09],];
          var data_6_alternation_2_common_prefix_rejit_best = [[8,5e+07],[16,8e+07],[32,1.23077e+08],[64,1.68421e+08],[128,3.36842e+08],[256,7.75758e+08],[512,1.34737e+09],[1024,1.82857e+09],[2048,1.79649e+09],[4096,2.42367e+09],[8192,2.77695e+09],[16384,3.18136e+09],[32768,3.29658e+09],[65536,3.4312e+09],[131072,3.64089e+09],[262144,6.69931e+09],[524288,6.99797e+09],[1048576,6.47789e+09],[2097152,6.2821e+09],[4194304,6.28219e+09],[8388608,5.70592e+09],[16777216,5.72891e+09],];
          var data_6_alternation_2_common_prefix_rejit_worst = [[8,299625],[16,607211],[32,1.19225e+06],[64,2.40421e+06],[128,4.80661e+06],[256,9.44649e+06],[512,1.90405e+07],[1024,3.72499e+07],[2048,7.35104e+07],[4096,1.43669e+08],[8192,2.72431e+08],[16384,5.10723e+08],[32768,8.86101e+08],[65536,1.39054e+09],[131072,1.9444e+09],[262144,4.92104e+09],[524288,5.28623e+09],[1048576,5.78908e+09],[2097152,5.91297e+09],[4194304,6.04393e+09],[8388608,5.46251e+09],[16777216,5.57811e+09],];
          var data_6_alternation_2_common_prefix_v8_amortised = [[8,160000000],[16,160000000],[32,320000000],[64,640000000],[128,853333333.3333333],[256,1706666666.6666665],[512,2560000000],[1024,4096000000],[2048,4551111111.111111],[4096,4818823529.411764],[8192,5120000000],[16384,4890746268.656716],[32768,4519724137.931034],[65536,4057956656.3467493],[131072,3276800000],[262144,3744914285.714286],[524288,3744914285.714286],[1048576,3615779310.344827],[2097152,3615779310.344827],[4194304,3647220869.565217],[8388608,3452102057.6131687],[16777216,3437954098.3606563],];
            var datasets = [ {data: data_6_alternation_2_common_prefix_re2_amortised, label: "re2_amortised", color: "#DEBD00", },{data: data_6_alternation_2_common_prefix_re2_best, label: "re2_best", color: "#E0D48D", },{data: data_6_alternation_2_common_prefix_re2_worst, label: "re2_worst", color: "#E0D48D", },{data: data_6_alternation_2_common_prefix_rejit_amortised, label: "rejit_amortised", color: "#277AD9", },{data: data_6_alternation_2_common_prefix_rejit_best, label: "rejit_best", color: "#94B8E0", },{data: data_6_alternation_2_common_prefix_rejit_worst, label: "rejit_worst", color: "#94B8E0", },{data: data_6_alternation_2_common_prefix_v8_amortised, label: "v8_amortised", color: "#00940A", }, ];
            var choiceContainer = $("#plot_parallel_6_alternation_2_common_prefix_choices");
            $.each(datasets, function(key, val) {
               choiceContainer.append('<li style="list-style: none;"><input type="checkbox" name="' + key +
                                      '" checked="checked" id="id' + key + '">' +
                                      '<label for="id' + key + '">'
                                      + val.label + '</label></li>');
            });
            plot_according_to_choices("plot_parallel_6_alternation_2_common_prefix", datasets, choiceContainer);
            $("#plot_parallel_6_alternation_2_common_prefix").bind("plothover", plothover_func);
            function replot() { plot_according_to_choices("plot_parallel_6_alternation_2_common_prefix", datasets, choiceContainer); }
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
  [grep jrep benchs article]: /blog/
