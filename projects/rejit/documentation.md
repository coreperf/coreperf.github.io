---
layout: project
title: CorePerf - Projects - Rejit - Documentation
project_name: Rejit
project_github_url: https://github.com/coreperf/rejit
---

* <a href="#building_and_usage">Building and usage</a>
* <a href="#sample_programs">Sample Programs</a>
* <a href="#running_the_benchmarks">Running the Benchmarks</a>
  * <a href="#rejit_benchmark_suite">Rejit benchmark suite</a>
  * <a href="#grep_benchmarks">Grep benchmarks</a>
  * <a href="#dna_matching_benchmarks">DNA matching benchmarks</a>
* <a href="#syntax">Syntax</a>
* <a href="#testing">Testing</a>
* <a href="#compilation_information">Compilation information</a>
  * <a href="#disassembly_of_the_generated_code">Disassembly of the generated code</a>

### Building and usage

All scripts and utilities should include a help message available via ```$ <script> --help```.

First build rejit.
{% highlight sh %}
$ cd <rejit>
$ scons
{% endhighlight %}

Then include ```<rejit>/include/rejit.h``` in your program.

{% highlight c %}
#include <rejit.h>
using namespace std;
using namespace rejit;

int main() {
  string text = "";
  vector<Match> matches;
  rejit::MatchAll("regexp", text, &matches);
  for (vector<Match>::iterator it = matches.begin(); it < matches.end(); it++) {
    // Do something.
  }
  return 0;
}
{% endhighlight %}

Finally, compile and link your program with the rejit library
{% highlight sh %}
$ g++ -o myprg myprg.cc -I<rejit>/include -L<rejit>/build/latest -lrejit
{% endhighlight %}

Documentation for the various functions offered by rejit are available as
comments in ```include/rejit.h```.  You can also find examples in the sample
programs provided in ```sample/```.
Compilation options are detailed in the help message from ```scons --help```.


### Sample programs

A few sample programs using rejit are included in the ```sample/``` folder.
It includes the ```regexdna``` and ```jrep``` samples. Compile them with:
{% highlight sh %}
$ scons sample/basic
$ scons sample/jrep
$ scons sample/regexdna
$ scons sample/regexdna-multithread
{% endhighlight %}

Use ```$ sample/<sample> --help``` for details.


### Running the benchmarks

#### Rejit benchmark suite
You can run the rejit benchmark suite with
{% highlight sh %}
$ <rejit>/tools/benchmarks/run.py
$ <browser> <rejit>/tools/benchmarks/benchmarks_results.html
{% endhighlight %}
As usual the ```--help``` switch will list the options available.

Benchmark engines can be built separately.
{% highlight sh %}
$ scons pcre_engine
$ scons re2_engine
$ scons rejit_engine
$ scons v8_engine
{% endhighlight %}
The build script will take care of cloning the appropriate repositories and
building everything. Files are located in ```tools/benchmarks/engines/<engine>```.

Note that compilation of re2 currently fails on OSX10.9 (at least).
[Here](http://code.google.com/p/re2/issues/detail?id=94) is a simple
fix.

#### Grep benchmarks
Grep benchmarks are currently stil run manually. The rejit-powered grep utility
is part of the sample programs.

You can benchmark with commands such as
{% highlight sh %}
$ CMD='jrep -R regexp linux-3.10.6/'; $CMD > /dev/null && time $CMD > /dev/null
{% endhighlight %}
Be very careful with shell special characters escaping. Also beware difference
of syntaxes between jrep and grep (using respectively the Extended and Basic
Regular Expression syntaxes), or use an option switch (```-E``` with ```GNU
grep```) to use ERE.

#### DNA matching benchmarks
This benchmark is detailed on its [original site][dna benchmark].
It is currently run manually. The rejit-powered implementations are part of the
sample programs. They are simply run with
{% highlight sh %}
$ ./sample/regexdna < input.file
{% endhighlight %}
See the ```--help``` option for information on how to generate the input files.


### Syntax

Rejit currently follows the POSIX Extended Regular Expression syntax([Wikipedia link][wikipedia ERE]).
Reintroducing the Basic Regular Expression syntax is part of the future tasks.

Rejit supports the following elements of the ERE:

- Parenthesis groups and alternations (```|```).
- Repetitions using brackets ```{min,max}```, ```*```, ```+```, ```?```.
- Anchors for start and end of lines (respectively ```^``` and ```$```).
- The ```.``` character, matching everything except newlines characters.
- Digits ```\d``` and non-digits ```\D```, equivalent respectively to
  ```[0-9]``` and ```[^0-9]``` for the ASCII encoding.
- Whitespace ```\s``` and non-whitespace ```\S```, equivalent respectively to ```[0-9]``` and ```[^0-9]``` for the ASCII
  encoding.
- Arbitrary hex encoding using the prefix ```\x```. (Eg. ```\x12``` matches the ASCII character encoded with the code ```0x12```.)

### Testing

Rejit includes a test suite that can be run with ```tools/tests/run.py```.


### Compilation Information

The `compinfo` tool can be used to show some information about the compilation
of a regexp.
{% highlight sh %}
$ scons compinfo
$ ./tools/analysis/compinfo --help
{% endhighlight %}

For example:

{% raw %}
```
$ ./tools/analysis/compinfo --print_ff_elements=1 "[0-9]abcd[0-9]"
Fast forward elements ----------------------{{{
MultipleChar [abcd] {1, 2}
}}}--------------- End of fast forward elements
```
{% endraw %}

#### Disassembly of the generated code

`compinfo` can also be used to dump the generated code so it can be
disassembled.
{% highlight sh %}
$ scons compinfo
$ ./tools/analysis/compinfo --dump_code=1 regexp
{% endhighlight %}
Then use ```ndisasm``` to disassemble the blob.
{% highlight sh %}
$ ndisasm -b 64 dump.1 > disasm.regexp
$ cat disasm.regexp
[...]
0000009D  66490F6F0E        movdqa xmm1,[r14]
000000A2  660F3A63C10C      pcmpistri xmm0,xmm1,0xc
000000A8  0F821C000000      jc qword 0xca
000000AE  66490F6F5610      movdqa xmm2,[r14+0x10]
000000B4  660F3A63C20C      pcmpistri xmm0,xmm2,0xc
000000BA  0F8206000000      jc qword 0xc6
000000C0  4983C620          add r14,byte +0x20
000000C4  EBCE              jmp short 0x94
[...]
{% endhighlight %}




  [dna benchmark]: http://benchmarksgame.alioth.debian.org/
  [wikipedia ERE]: http://en.wikipedia.org/wiki/Regular_expression#POSIX_extended
