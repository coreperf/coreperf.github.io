---
layout: post
title:  "More grep vs rejit benchmarks"
date:   2013-08-28
author: Alexandre
categories: [update,rejit]
---


I realised that the [article introducing rejit][rejit article] contained only one benchmark
comparing rejit to GNU grep.
<br />
So below are shortly commented benchmark results (grepping through the Linux kernel
sources) for a wider range of regular expressions.

The commands used to benchmarks follow the form:

    $ REGEXP="re"; CMD="jrep -R $REGEXP linux-3.10.6 "; \
    $CMD > /dev/null && time $CMD > /tmp/res

    $ REGEXP="re"; CMD="grep -R $REGEXP linux-3.10.6 "; \
    $CMD > /dev/null && time $CMD > /dev/null

For concision, the REGEXP variable is reminded only in the extended regular
expression (ERE) format. (Grep uses BRE.)

Benchmarks were run with the engines versions below:

    GNU grep version 2.14, commit: 599f2b15bc152cdf19022c7803f9a5f336c25e65
    Rejit commit: 6cd1ca642783ac9968085682b1a82b6961ad9109

Jrep is a grep-like utility, a simple wrapper around the rejit library,
provided as a sample program in the repository.

<hr>

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
      <br />time for grep / time for jrep
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
      <strong>Regexp</strong>: <code>"regexp|yoyo"</code>
    </td>
  </tr>
  <tr>
    <td>
      <pre style="width:80%"><code>real  0m2.097s
user  0m1.812s
sys   0m0.275s</code></pre>
    </td>
    <td>
      <pre style="width:80%"><code>real  0m0.410s
user  0m0.137s
sys   0m0.268s</code></pre>
    </td>
    <td>
      <span style="color:green">5.11</span>
    </td>
  </tr>
  <tr><td colspan="3">
    Rejit uses simple brute-force SIMD code checking for all words
    parallelly.
  </td></tr>
  <tr>
    <td colspan="3">
      <strong>Regexp</strong>: <code>"regexp|yoyo|unsigned"</code>
    </td>
  </tr>
  <tr>
    <td>
      <pre style="width:80%"><code>real  0m2.125s
user  0m1.839s
sys   0m0.275s</code></pre>
    </td>
    <td>
      <pre style="width:80%"><code>real  0m1.319s
user  0m0.916s
sys   0m0.393s</code></pre>
    </td>
    <td>
      <span style="color:green">1.61</span>
    </td>
  </tr>
  <tr><td colspan="3">
    The performance advantage of SIMD code decreases with the number of
    alternations. Grep shows stable performance with its (Boyer-Moore-like
    I suppose) algorithm.
  </td></tr>
  <tr>
    <td colspan="3">
      <strong>Regexp</strong>: <code>"regexp|yoyo|unsigned|struct|union"</code>
    </td>
  </tr>
  <tr>
    <td>
      <pre style="width:80%"><code>real  0m2.214s
user  0m1.907s
sys   0m0.298s</code></pre>
    </td>
    <td>
      <pre style="width:80%"><code>real  0m2.082s
user  0m1.590s
sys   0m0.479s</code></pre>
    </td>
    <td>
      <span>1.06</span>
    </td>
  </tr>
  <tr>
    <td colspan="3">
      <strong>Regexp</strong>: <code>"regexp|yoyo|unsigned|struct|union|static"</code>
    </td>
  </tr>
  <tr>
    <td>
      <pre style="width:80%"><code>real  0m2.184s
user  0m1.903s
sys   0m0.272s</code></pre>
    </td>
    <td>
      <pre style="width:80%"><code>real  0m2.239s
user  0m1.730s
sys   0m0.497s</code></pre>
    </td>
    <td>
      <span>0.98</span>
    </td>
  </tr>
  <tr><td colspan="3">
    The SIMD brute-force starts being slower than grep's algorithm.
    With this many alternations, it would be time to fall back to a smarter
    algorithm or implement faster SIMD code.
  </td></tr>
  <tr>
    <td colspan="3">
      <strong>Regexp</strong>: <code>"regexp|yoyo|unsigned|struct|union|static|void"</code>
    </td>
  </tr>
  <tr>
    <td>
      <pre style="width:80%"><code>real  0m2.199s
user  0m1.925s
sys   0m0.263s</code></pre>
    </td>
    <td>
      <pre style="width:80%"><code>real  0m2.530s
user  0m2.009s
sys   0m0.508s</code></pre>
    </td>
    <td>
      <span style="color:red">0.86 (1 / 1.15) </span>
    </td>
  </tr>
  <tr>
    <td colspan="3">
      <strong>Regexp</strong>: <code>"regexp|yoyo|unsigned|struct|union|static|void|NULL"</code>
    </td>
  </tr>
  <tr>
    <td>
      <pre style="width:80%"><code>real  0m2.202s
user  0m1.915s
sys   0m0.278s</code></pre>
    </td>
    <td>
      <pre style="width:80%"><code>real  0m5.694s
user  0m5.163s
sys   0m0.511s</code></pre>
    </td>
    <td>
      <span style="color:red">0.39 (1 / 2.59)</span>
    </td>
  </tr>
  <tr><td colspan="3">
    At this point there are too many alternations for rejit to use SSE4.2.
    Rejit doesn't do any register allocation but simply assigns one XMM
    register to each word. 
    When not enough XMM registers are available, it falls back to (unluckily
    still brute-force) non-SIMD code.
    Performance drops significantly.
  </td></tr>
  <tr>
    <td colspan="3">
      <strong>Regexp</strong>: <code>"\s+void.*\(\)"</code>
    </td>
  </tr>
  <tr>
    <td>
      <pre style="width:80%"><code>real  0m0.836s
user  0m0.576s
sys   0m0.255s</code></pre>
    </td>
    <td>
      <pre style="width:80%"><code>real  0m0.371s
user  0m0.101s
sys   0m0.265s</code></pre>
    </td>
    <td>
      <span style="color:green">2.25</span>
    </td>
  </tr>
  <tr><td colspan="3">
    The first of a few measurements for complicated regular expressions which
    may be delicate to handle.
    <br />It looks like both grep and rejit know well enough to search for the
    parts of the regular expression that are easy to match.
    <br />With few alternations, rejit has the advantage.
    It would start losing the race if we started increasing the number of
    alternations. (As in the alternation tests above.)
  </td></tr>
  <tr>
    <td colspan="3">
      <strong>Regexp</strong>: <code>"(\s+void.*\(\))|(union.*\{)"</code>
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
  <tr>
    <td colspan="3">
      <strong>Regexp</strong>: <code>"(\s+void.*\(\))|(struct.*\{)"</code>
    </td>
  </tr>
  <tr>
    <td>
      <pre style="width:80%"><code>real  0m2.141s
user  0m1.866s
sys   0m0.264s</code></pre>
    </td>
    <td>
      <pre style="width:80%"><code>real  0m2.102s
user  0m1.653s
sys   0m0.438s</code></pre>
    </td>
    <td>
      <span>1.02</span>
    </td>
  </tr>
  <tr><td colspan="3">
    <p>
    Interesting difference with above!<br />
    I initially thought that the the performance gap was caused differences in
    frequencies for the first or last letters in <code>struct</code> and
    <code>union</code>.
    <br />Further testing seems to indicate that this is caused by the much
    higher number of matches: 119183 against 5484 lines matching.
    Duplicating the last letter to look for <code>structs</code> drops the
    number of matches and restores performance.
    </p>
  </td></tr>
  <tr>
    <td colspan="3">
      <strong>Regexp</strong>: <code>"struct"</code>
    </td>
  </tr>
  <tr>
    <td>
      <pre style="width:80%"><code>real  0m1.075s
user  0m0.769s
sys   0m0.300s</code></pre>
    </td>
    <td>
      <pre style="width:80%"><code>real 0m1.988s
user  0m1.498s
sys   0m0.478s</code></pre>
    </td>
    <td>
      <span style="color:red">0.54</span>
    </td>
  </tr>
  <tr><td colspan="3">
    <p>
    Let's benchmark again for a single word with many matches (1038237 lines
    matching, 10x more). Grep still has a big advantage. For comparison,
    testing with <code>int</code> (1013108 lines matching) gives equivalent
    results.
    </p>
  </td></tr>
  <tr>
    <td colspan="3">
      <strong>Regexp</strong>: <code>"structs"</code>
    </td>
  </tr>
  <tr>
    <td>
      <pre style="width:80%"><code>real 0m0.813s
user  0m0.533s
sys   0m0.278s</code></pre>
    </td>
    <td>
      <pre style="width:80%"><code>real 0m0.424s
user  0m0.148s
sys   0m0.270s</code></pre>
    </td>
    <td>
      <span style="color:green">1.92</span>
    </td>
  </tr>
  <tr><td colspan="3">
    <p>
    Now we have few matches (897 lines), but a word with a similar profile.
    Jrep gains its performance back!
    </p>
  </td></tr>
</table>

<hr>

The tests indicate that Rejit would benefit from some cleverer
algorithms and jrep from better memory management, but globally it performs
very well. Grep's stability is impressive.

<br />
Updated on 02/09/2013:
* Updated comments for the <code>"(\s+void.*\(\))|(struct.*\{)"</code> benchmark.
* Add benchmarks for <code>struct</code> and <code>structs</code>

[rejit article]: /projects/rejit/
