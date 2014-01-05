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

### Building and usage

All scripts and utilities should include a help message available via ```$ <script> --help```

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

Documentation for the various functions offered by rejit are available as comments in ```include/rejit.h```.
You can also find examples in the sample programs provided in ```sample/```.


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

#### Grep benchmarks
Grep benchmarks are currently stil run manually. The rejit-powered grep utility
is part of the sample programs.

You can benchmark with commands such as
{% highlight sh %}
$ CMD='jrep -R regexp linux-3.10.6/'; $CMD > /dev/null && time $CMD > /dev/null
{% endhighlight %}
Be very careful with shell special characters escaping, and the difference of
syntaxes between jrep and grep (using respectively the Extended and Basic Regular Expression syntaxes).

#### DNA matching benchmarks
This benchmark is detailed on its [original site][dna benchmark].
It is currently run manually. The rejit-powered implementations are part of the
sample programs. They are simply run with
{% highlight sh %}
$ ./sample/regexdna < input.file
{% endhighlight %}
See the ```--help``` option for information on how to generate the input files.


  [dna benchmark]: http://benchmarksgame.alioth.debian.org/
