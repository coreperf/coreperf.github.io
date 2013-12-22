---
layout: project
title: CorePerf - Projects - Rejit - Documentation
project_name: Rejit
project_github_url: https://github.com/coreperf/rejit
---


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
