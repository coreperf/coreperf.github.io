---
layout: post
title:  "Side by side benchmarks of pcre, re2, rejit, and v8."
date:   2013-09-01
author: Alexandre
categories: [update,rejit]
---

After having a look at the benchmark results for rejit, v8, and re2, a few
people noted that I should also benchmark PCRE, so here are the results.

The benchmark suite can be run as usually with:

    $ git clone git@github.com:coreperf/rejit.git && cd rejit
    $ ./tools/benchmarks/run.py
    $ <browser> html/rejit.html

The results are for the following engines versions

    Rejit commit: 953557acd1614dfba5bff37bd12c2674a2e22eb7
    V8 version 3.20.9, commit: 455bb9c2ab8af080aa15e0fbf4838731f45241e8
    Re2 commit: aa957b5e3374
    PCRE revision: 1358

on a machine with a quad-core Intel Core
i5-2400 CPU @ 3.10GHz with 4GiB RAM on Fedora 17 (3.8.13-100.fc17.x86\_64).

The PCRE benchmark engine was adapted from the repository example. I found it
much harder to put in place than I thought it should be, and the results don't
look as good as I expected from the comments that lead me to add it. Suggestions
to improve the PCRE benchmark engine implementation are welcome.



<script language="javascript" type="text/javascript" src="/resources/js/flot/jquery.min.js"> </script>
<script language="javascript" type="text/javascript" src="/resources/js/flot/jquery.flot.min.js"> </script>
<script language="javascript" type="text/javascript" src="/resources/js/flot/jquery.flot.time.min.js"> </script>
<script language="javascript" type="text/javascript" src="/resources/js/flot_utils.js"> </script>

<table>
  <tr>
    <td>
      <div style="padding-left: 5em;"> <p>
(0) <strong>Regexp:</strong> <code>abcdefg</code>
<br />
Search for a simple concatenation of characters.
<br />
Starting character 'a' is <strong>not</strong> present in the text searched.
</p>
 </div>
    </td>
  </tr>

  <tr>
    <td>
      <div>
        <div id="plot_parallel_0_chars_null_density" style="float:left;width:600px;height:400px"></div>
        <div style="float:left;"> <ul id="plot_parallel_0_chars_null_density_choices" style="list-style: none;" class="flot_choices"> </ul> </div>
      </div>
      <script language="javascript" type="text/javascript">
        <!--
        $(function () {
          var data_0_chars_null_density_pcre_amortised = [[8,6.15385e+07],[16,1.06667e+08],[32,1.88235e+08],[64,2.90909e+08],[128,4.26667e+08],[256,4.83019e+08],[512,6.5641e+08],[1024,7.21127e+08],[2048,7.3405e+08],[4096,7.40687e+08],[8192,7.82426e+08],[16384,7.8959e+08],[32768,7.91306e+08],[65536,1.66928e+09],[131072,1.63411e+09],[262144,1.69158e+09],[524288,1.68842e+09],[1048576,1.69112e+09],[2097152,1.68385e+09],[4194304,1.67974e+09],[8388608,1.64125e+09],[16777216,1.63868e+09],];
var data_0_chars_null_density_pcre_best = [[8,7.27273e+07],[16,1.23077e+08],[32,2e+08],[64,3.2e+08],[128,4.41379e+08],[256,4.92308e+08],[512,6.73684e+08],[1024,7.31429e+08],[2048,7.3935e+08],[4096,7.42029e+08],[8192,7.83923e+08],[16384,7.89971e+08],[32768,7.91498e+08],[65536,1.66971e+09],[131072,1.63431e+09],[262144,1.69169e+09],[524288,1.68847e+09],[1048576,1.69114e+09],[2097152,1.68388e+09],[4194304,1.67974e+09],[8388608,1.64126e+09],[16777216,1.63868e+09],];
var data_0_chars_null_density_pcre_worst = [[8,6.72269e+06],[16,1.46789e+07],[32,2.83186e+07],[64,5.51724e+07],[128,9.55224e+07],[256,1.8156e+08],[512,2.95954e+08],[1024,4.32068e+08],[2048,5.61096e+08],[4096,6.49128e+08],[8192,7.09264e+08],[16384,7.52941e+08],[32768,7.69925e+08],[65536,8.41716e+08],[131072,1.5384e+09],[262144,1.61081e+09],[524288,1.65078e+09],[1048576,1.68741e+09],[2097152,1.68199e+09],[4194304,1.67898e+09],[8388608,1.64033e+09],[16777216,1.63781e+09],];
var data_0_chars_null_density_re2_amortised = [[8,1.86047e+07],[16,3.72093e+07],[32,7.44186e+07],[64,1.45455e+08],[128,2.90909e+08],[256,5.68889e+08],[512,1.08936e+09],[1024,2.0898e+09],[2048,3.79259e+09],[4096,6.30154e+09],[8192,9.63765e+09],[16384,1.37681e+10],[32768,1.46942e+10],[65536,1.57161e+10],[131072,1.53301e+10],[262144,1.47272e+10],[524288,2.66001e+10],[1048576,2.64058e+10],[2097152,2.64892e+10],[4194304,2.58796e+10],[8388608,1.08456e+10],[16777216,9.19028e+09],];
var data_0_chars_null_density_re2_best = [[8,2.42424e+07],[16,4.84848e+07],[32,9.69697e+07],[64,1.88235e+08],[128,3.65714e+08],[256,7.31429e+08],[512,1.38378e+09],[1024,2.62564e+09],[2048,4.55111e+09],[4096,7.44727e+09],[8192,1.10703e+10],[16384,1.50312e+10],[32768,1.5384e+10],[65536,1.60627e+10],[131072,1.55115e+10],[262144,1.48188e+10],[524288,2.66678e+10],[1048576,2.64391e+10],[2097152,2.6506e+10],[4194304,2.58908e+10],[8388608,1.08472e+10],[16777216,9.19093e+09],];
var data_0_chars_null_density_re2_worst = [[8,430802],[16,869565],[32,1.74387e+06],[64,3.45759e+06],[128,6.93767e+06],[256,1.39055e+07],[512,2.79171e+07],[1024,5.52916e+07],[2048,1.11003e+08],[4096,2.20215e+08],[8192,4.39249e+08],[16384,8.56456e+08],[32768,1.6e+09],[65536,3.25241e+09],[131072,4.83304e+09],[262144,6.74759e+09],[524288,1.54977e+10],[1048576,2.10769e+10],[2097152,2.34817e+10],[4194304,2.35834e+10],[8388608,1.03933e+10],[16777216,9.03322e+09],];
var data_0_chars_null_density_rejit_amortised = [[8,1.95122e+07],[16,3.80952e+07],[32,6.95652e+07],[64,1.3913e+08],[128,2.61224e+08],[256,4.92308e+08],[512,9.30909e+08],[1024,1.76552e+09],[2048,2.84444e+09],[4096,4.35745e+09],[8192,5.76901e+09],[16384,6.88403e+09],[32768,7.5156e+09],[65536,7.98246e+09],[131072,8.23317e+09],[262144,1.71112e+10],[524288,1.60431e+10],[1048576,1.75523e+10],[2097152,1.75025e+10],[4194304,1.73089e+10],[8388608,1.03952e+10],[16777216,9.00147e+09],];
var data_0_chars_null_density_rejit_best = [[8,3.80952e+07],[16,7.27273e+07],[32,1.23077e+08],[64,2.66667e+08],[128,5.12e+08],[256,9.48148e+08],[512,1.70667e+09],[1024,2.84444e+09],[2048,4.17959e+09],[4096,5.53514e+09],[8192,6.71475e+09],[16384,7.5156e+09],[32768,7.87692e+09],[65536,8.192e+09],[131072,8.34854e+09],[262144,1.7235e+10],[524288,1.61022e+10],[1048576,1.75847e+10],[2097152,1.75186e+10],[4194304,1.73168e+10],[8388608,1.03979e+10],[16777216,9.00254e+09],];
var data_0_chars_null_density_rejit_worst = [[8,362483],[16,723327],[32,1.45059e+06],[64,2.91439e+06],[128,5.81554e+06],[256,1.17055e+07],[512,2.28776e+07],[1024,4.60018e+07],[2048,9.11843e+07],[4096,1.82857e+08],[8192,3.56484e+08],[16384,6.77585e+08],[32768,1.24688e+09],[65536,2.15863e+09],[131072,3.28419e+09],[262144,5.08919e+09],[524288,1.13164e+10],[1048576,1.36462e+10],[2097152,1.56586e+10],[4194304,1.608e+10],[8388608,9.73416e+09],[16777216,8.76159e+09],];
var data_0_chars_null_density_v8_amortised = [[8,80000000],[16,160000000],[32,320000000],[64,426666666.6666666],[128,853333333.3333333],[256,1706666666.6666665],[512,5120000000],[1024,6826666666.666666],[2048,13653333333.333332],[4096,16384000000],[8192,27306666666.666664],[16384,32768000000],[32768,38550588235.29411],[65536,35424864864.86487],[131072,13107200000],[262144,26214400000],[524288,17476266666.666664],[1048576,26214400000],[2097152,23301688888.888885],[4194304,26214400000],[8388608,10485760000],[16777216,9218250549.45055],];

          var datasets = [ {data: data_0_chars_null_density_pcre_amortised, label: "pcre_amortised",
color: "#DEBD00",
},{data: data_0_chars_null_density_pcre_best, label: "pcre_best",
color: "#E0D48D",
},{data: data_0_chars_null_density_pcre_worst, label: "pcre_worst",
color: "#E0D48D",
},{data: data_0_chars_null_density_re2_amortised, label: "re2_amortised",
color: "#277AD9",
},{data: data_0_chars_null_density_re2_best, label: "re2_best",
color: "#94B8E0",
},{data: data_0_chars_null_density_re2_worst, label: "re2_worst",
color: "#94B8E0",
},{data: data_0_chars_null_density_rejit_amortised, label: "rejit_amortised",
color: "#00940A",
},{data: data_0_chars_null_density_rejit_best, label: "rejit_best",
color: "#72B377",
},{data: data_0_chars_null_density_rejit_worst, label: "rejit_worst",
color: "#72B377",
},{data: data_0_chars_null_density_v8_amortised, label: "v8_amortised",
color: "#A22EBF",
}, ];

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
      <div style="padding-left: 5em;"> <p>
(1) <strong>Regexp:</strong> <code>abcdefg</code>
<br />
Same as before, except that the text to match is composed of random (equally
probable) characters in the range [0-z].
</p>
 </div>
    </td>
  </tr>

  <tr>
    <td>
      <div>
        <div id="plot_parallel_1_chars_average_density" style="float:left;width:600px;height:400px"></div>
        <div style="float:left;"> <ul id="plot_parallel_1_chars_average_density_choices" style="list-style: none;" class="flot_choices"> </ul> </div>
      </div>
      <script language="javascript" type="text/javascript">
        <!--
        $(function () {
          var data_1_chars_average_density_pcre_amortised = [[8,6.15385e+07],[16,1.14286e+08],[32,1.77778e+08],[64,2.90909e+08],[128,2.78261e+08],[256,2.61224e+08],[512,2.27556e+08],[1024,2.26049e+08],[2048,2.48242e+08],[4096,2.41083e+08],[8192,2.51365e+08],[16384,3.23156e+08],[32768,4.89586e+08],[65536,5.40548e+08],[131072,5.44839e+08],[262144,5.42337e+08],[524288,5.37522e+08],[1048576,5.40049e+08],[2097152,5.39477e+08],[4194304,5.38776e+08],[8388608,5.37003e+08],[16777216,5.36722e+08],];
var data_1_chars_average_density_pcre_best = [[8,7.27273e+07],[16,1.33333e+08],[32,1.88235e+08],[64,3.04762e+08],[128,2.90909e+08],[256,2.66667e+08],[512,2.29596e+08],[1024,2.27051e+08],[2048,2.48846e+08],[4096,2.41367e+08],[8192,2.51443e+08],[16384,3.23283e+08],[32768,4.89659e+08],[65536,5.40592e+08],[131072,5.44862e+08],[262144,5.42348e+08],[524288,5.37527e+08],[1048576,5.40052e+08],[2097152,5.39478e+08],[4194304,5.38777e+08],[8388608,5.37004e+08],[16777216,5.36723e+08],];
var data_1_chars_average_density_pcre_worst = [[8,7.33945e+06],[16,1.45455e+07],[32,2.80702e+07],[64,5e+07],[128,8.82759e+07],[256,1.30612e+08],[512,1.57538e+08],[1024,1.85507e+08],[2048,2.2069e+08],[4096,2.26298e+08],[8192,2.41154e+08],[16384,2.49035e+08],[32768,5.11042e+08],[65536,4.78819e+08],[131072,5.26119e+08],[262144,5.29402e+08],[524288,5.34328e+08],[1048576,5.39838e+08],[2097152,5.38992e+08],[4194304,5.38688e+08],[8388608,5.36858e+08],[16777216,5.36675e+08],];
var data_1_chars_average_density_re2_amortised = [[8,1.86047e+07],[16,3.72093e+07],[32,7.44186e+07],[64,1.3617e+08],[128,2.61224e+08],[256,4.41379e+08],[512,6.32099e+08],[1024,8.60504e+08],[2048,1.05026e+09],[4096,1.13463e+09],[8192,1.26615e+09],[16384,1.2911e+09],[32768,1.04158e+09],[65536,1.73238e+09],[131072,1.55945e+09],[262144,1.50796e+09],[524288,1.45035e+09],[1048576,1.44478e+09],[2097152,1.44281e+09],[4194304,1.44573e+09],[8388608,1.4155e+09],[16777216,1.41473e+09],];
var data_1_chars_average_density_re2_best = [[8,2.35294e+07],[16,4.84848e+07],[32,9.69697e+07],[64,1.72973e+08],[128,3.28205e+08],[256,5.33333e+08],[512,7.11111e+08],[1024,9.3945e+08],[2048,1.10703e+09],[4096,1.16364e+09],[8192,1.28603e+09],[16384,1.30135e+09],[32768,1.0449e+09],[65536,1.73467e+09],[131072,1.56224e+09],[262144,1.5084e+09],[524288,1.45059e+09],[1048576,1.4449e+09],[2097152,1.44287e+09],[4194304,1.44577e+09],[8388608,1.41553e+09],[16777216,1.41474e+09],];
var data_1_chars_average_density_re2_worst = [[8,428954],[16,855158],[32,1.73536e+06],[64,3.40426e+06],[128,6.77249e+06],[256,1.32987e+07],[512,2.43693e+07],[1024,4.65455e+07],[2048,8.75588e+07],[4096,1.59813e+08],[8192,2.81512e+08],[16384,4.37373e+08],[32768,5.73068e+08],[65536,6.65138e+08],[131072,1.22853e+09],[262144,1.29607e+09],[524288,1.36395e+09],[1048576,1.40991e+09],[2097152,1.423e+09],[4194304,1.43394e+09],[8388608,1.40857e+09],[16777216,1.41165e+09],];
var data_1_chars_average_density_rejit_amortised = [[8,1.95122e+07],[16,3.72093e+07],[32,7.11111e+07],[64,1.30612e+08],[128,2.56e+08],[256,4.83019e+08],[512,9.14286e+08],[1024,1.6e+09],[2048,2.4381e+09],[4096,3.53103e+09],[8192,4.65455e+09],[16384,5.59181e+09],[32768,6.01248e+09],[65536,6.3076e+09],[131072,6.27739e+09],[262144,1.31006e+10],[524288,1.18349e+10],[1048576,1.18243e+10],[2097152,1.15565e+10],[4194304,1.14573e+10],[8388608,9.30764e+09],[16777216,8.74501e+09],];
var data_1_chars_average_density_rejit_best = [[8,3.80952e+07],[16,7.27273e+07],[32,1.28e+08],[64,2.56e+08],[128,5.12e+08],[256,9.14286e+08],[512,1.65161e+09],[1024,2.56e+09],[2048,3.30323e+09],[4096,4.31158e+09],[8192,5.28516e+09],[16384,6.00147e+09],[32768,6.24152e+09],[65536,6.44405e+09],[131072,6.34117e+09],[262144,1.31664e+10],[524288,1.18671e+10],[1048576,1.1839e+10],[2097152,1.15635e+10],[4194304,1.14614e+10],[8388608,9.31012e+09],[16777216,8.74601e+09],];
var data_1_chars_average_density_rejit_worst = [[8,361337],[16,728597],[32,1.45322e+06],[64,2.88029e+06],[128,5.79186e+06],[256,1.16948e+07],[512,2.32094e+07],[1024,4.62094e+07],[2048,9.13063e+07],[4096,1.79886e+08],[8192,3.50535e+08],[16384,6.59316e+08],[32768,1.18083e+09],[65536,1.98174e+09],[131072,2.90819e+09],[262144,4.52362e+09],[524288,8.7425e+09],[1048576,1.01724e+10],[2097152,1.06905e+10],[4194304,1.08e+10],[8388608,8.78692e+09],[16777216,8.46227e+09],];
var data_1_chars_average_density_v8_amortised = [[8,53333333.33333333],[16,160000000],[32,320000000],[64,640000000],[128,853333333.3333333],[256,1280000000],[512,1462857142.857143],[1024,2048000000],[2048,1638400000],[4096,1638400000],[8192,1531214953.271028],[16384,1575384615.3846157],[32768,1524093023.255814],[65536,1510046082.9493086],[131072,1456355555.5555553],[262144,1456355555.5555553],[524288,1456355555.5555553],[1048576,1476867605.633803],[2097152,1466539860.1398602],[4194304,1466539860.1398602],[8388608,1436405479.4520547],[16777216,1436405479.4520547],];

          var datasets = [ {data: data_1_chars_average_density_pcre_amortised, label: "pcre_amortised",
color: "#DEBD00",
},{data: data_1_chars_average_density_pcre_best, label: "pcre_best",
color: "#E0D48D",
},{data: data_1_chars_average_density_pcre_worst, label: "pcre_worst",
color: "#E0D48D",
},{data: data_1_chars_average_density_re2_amortised, label: "re2_amortised",
color: "#277AD9",
},{data: data_1_chars_average_density_re2_best, label: "re2_best",
color: "#94B8E0",
},{data: data_1_chars_average_density_re2_worst, label: "re2_worst",
color: "#94B8E0",
},{data: data_1_chars_average_density_rejit_amortised, label: "rejit_amortised",
color: "#00940A",
},{data: data_1_chars_average_density_rejit_best, label: "rejit_best",
color: "#72B377",
},{data: data_1_chars_average_density_rejit_worst, label: "rejit_worst",
color: "#72B377",
},{data: data_1_chars_average_density_v8_amortised, label: "v8_amortised",
color: "#A22EBF",
}, ];

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
      <div style="padding-left: 5em;"> <p>
(2) <strong>Regexp:</strong> <code>abcdefg</code>
<br />
Same as before, but this time the character range is [a-j].
</p>
 </div>
    </td>
  </tr>

  <tr>
    <td>
      <div>
        <div id="plot_parallel_2_chars_high_density" style="float:left;width:600px;height:400px"></div>
        <div style="float:left;"> <ul id="plot_parallel_2_chars_high_density_choices" style="list-style: none;" class="flot_choices"> </ul> </div>
      </div>
      <script language="javascript" type="text/javascript">
        <!--
        $(function () {
          var data_2_chars_high_density_pcre_amortised = [[8,4.70588e+07],[16,8.42105e+07],[32,1.23077e+08],[64,1.6e+08],[128,1.14286e+08],[256,1.33333e+08],[512,1.52836e+08],[1024,1.31959e+08],[2048,1.39795e+08],[4096,1.29415e+08],[8192,1.22378e+08],[16384,2.38764e+08],[32768,2.56682e+08],[65536,2.44958e+08],[131072,2.57534e+08],[262144,2.55588e+08],[524288,2.56214e+08],[1048576,2.49735e+08],[2097152,2.56048e+08],[4194304,2.54673e+08],[8388608,2.54938e+08],[16777216,2.5453e+08],];
var data_2_chars_high_density_pcre_best = [[8,5e+07],[16,9.41176e+07],[32,1.33333e+08],[64,1.64103e+08],[128,1.16364e+08],[256,1.34737e+08],[512,1.53754e+08],[1024,1.323e+08],[2048,1.39986e+08],[4096,1.29497e+08],[8192,1.22397e+08],[16384,2.38799e+08],[32768,2.56702e+08],[65536,2.44967e+08],[131072,2.57539e+08],[262144,2.55591e+08],[524288,2.56215e+08],[1048576,2.49736e+08],[2097152,2.56048e+08],[4194304,2.54673e+08],[8388608,2.54939e+08],[16777216,2.5453e+08],];
var data_2_chars_high_density_pcre_worst = [[8,6.77966e+06],[16,1.40351e+07],[32,2.62295e+07],[64,4.60432e+07],[128,6.1244e+07],[256,8.82759e+07],[512,1.19626e+08],[1024,1.16629e+08],[2048,1.29293e+08],[4096,1.2465e+08],[8192,1.20029e+08],[16384,2.49794e+08],[32768,2.16964e+08],[65536,2.41875e+08],[131072,2.52776e+08],[262144,2.53088e+08],[524288,2.55754e+08],[1048576,2.49624e+08],[2097152,2.56007e+08],[4194304,2.5552e+08],[8388608,2.5504e+08],[16777216,2.54602e+08],];
var data_2_chars_high_density_re2_amortised = [[8,1.73913e+07],[16,3.55556e+07],[32,6.66667e+07],[64,1.12281e+08],[128,1.70667e+08],[256,2.78261e+08],[512,4.0315e+08],[1024,4.30252e+08],[2048,5.15869e+08],[4096,4.97691e+08],[8192,4.53348e+08],[16384,4.01569e+08],[32768,7.22877e+08],[65536,7.00695e+08],[131072,6.88801e+08],[262144,6.77707e+08],[524288,6.74768e+08],[1048576,6.71144e+08],[2097152,6.66734e+08],[4194304,6.73366e+08],[8388608,6.63077e+08],[16777216,6.61266e+08],];
var data_2_chars_high_density_re2_best = [[8,2.22222e+07],[16,4.44444e+07],[32,8.42105e+07],[64,1.3617e+08],[128,1.96923e+08],[256,3.12195e+08],[512,4.41379e+08],[1024,4.49123e+08],[2048,5.29199e+08],[4096,5.04433e+08],[8192,4.55871e+08],[16384,4.02654e+08],[32768,7.23675e+08],[65536,7.00995e+08],[131072,6.88982e+08],[262144,6.77795e+08],[524288,6.7482e+08],[1048576,6.71166e+08],[2097152,6.66747e+08],[4194304,6.73373e+08],[8388608,6.63083e+08],[16777216,6.6127e+08],];
var data_2_chars_high_density_re2_worst = [[8,425306],[16,853789],[32,1.71582e+06],[64,3.17618e+06],[128,5.75281e+06],[256,1.08705e+07],[512,2.09065e+07],[1024,3.73996e+07],[2048,7.00171e+07],[4096,1.19278e+08],[8192,1.83102e+08],[16384,2.3243e+08],[32768,2.61266e+08],[65536,5.60041e+08],[131072,6.03907e+08],[262144,6.36766e+08],[524288,6.52952e+08],[1048576,6.60753e+08],[2097152,6.63696e+08],[4194304,6.70294e+08],[8388608,6.61289e+08],[16777216,6.61959e+08],];
var data_2_chars_high_density_rejit_amortised = [[8,1.86047e+07],[16,3.55556e+07],[32,6.80851e+07],[64,1.33333e+08],[128,2.46154e+08],[256,4.26667e+08],[512,8e+08],[1024,1.34737e+09],[2048,2.17872e+09],[4096,2.84444e+09],[8192,3.47119e+09],[16384,4.08579e+09],[32768,4.34589e+09],[65536,4.45823e+09],[131072,4.41617e+09],[262144,7.56767e+09],[524288,7.02234e+09],[1048576,6.77637e+09],[2097152,6.73805e+09],[4194304,6.68895e+09],[8388608,6.11187e+09],[16777216,6.11256e+09],];
var data_2_chars_high_density_rejit_best = [[8,4e+07],[16,6.95652e+07],[32,1.28e+08],[64,2.56e+08],[128,4.74074e+08],[256,7.31429e+08],[512,1.28e+09],[1024,1.93208e+09],[2048,2.88451e+09],[4096,3.33008e+09],[8192,3.81023e+09],[16384,4.31158e+09],[32768,4.4704e+09],[65536,4.52597e+09],[131072,4.44915e+09],[262144,7.59618e+09],[524288,7.0327e+09],[1048576,6.78119e+09],[2097152,6.74065e+09],[4194304,6.69033e+09],[8388608,6.11285e+09],[16777216,6.11305e+09],];
var data_2_chars_high_density_rejit_worst = [[8,365965],[16,726942],[32,1.44731e+06],[64,2.91838e+06],[128,5.81554e+06],[256,1.14798e+07],[512,2.29494e+07],[1024,4.61469e+07],[2048,9.0901e+07],[4096,1.76096e+08],[8192,3.42189e+08],[16384,6.31368e+08],[32768,1.10628e+09],[65536,1.76838e+09],[131072,2.40235e+09],[262144,5.63024e+09],[524288,5.22668e+09],[1048576,5.99529e+09],[2097152,6.3383e+09],[4194304,6.47699e+09],[8388608,5.89825e+09],[16777216,5.96165e+09],];
var data_2_chars_high_density_v8_amortised = [[8,80000000],[16,106666666.66666666],[32,320000000],[64,426666666.6666666],[128,853333333.3333333],[256,731428571.4285715],[512,853333333.3333333],[1024,1077894736.8421052],[2048,1170285714.2857141],[4096,1153802816.9014084],[8192,1153802816.9014084],[16384,1141742160.2787457],[32768,1085033112.5827816],[65536,978149253.7313434],[131072,936228571.4285715],[262144,936228571.4285715],[524288,919803508.7719297],[1048576,927943362.8318584],[2097152,932067555.5555556],[4194304,936228571.4285715],[8388608,924874090.4079384],[16777216,1827583442.2657955],];

          var datasets = [ {data: data_2_chars_high_density_pcre_amortised, label: "pcre_amortised",
color: "#DEBD00",
},{data: data_2_chars_high_density_pcre_best, label: "pcre_best",
color: "#E0D48D",
},{data: data_2_chars_high_density_pcre_worst, label: "pcre_worst",
color: "#E0D48D",
},{data: data_2_chars_high_density_re2_amortised, label: "re2_amortised",
color: "#277AD9",
},{data: data_2_chars_high_density_re2_best, label: "re2_best",
color: "#94B8E0",
},{data: data_2_chars_high_density_re2_worst, label: "re2_worst",
color: "#94B8E0",
},{data: data_2_chars_high_density_rejit_amortised, label: "rejit_amortised",
color: "#00940A",
},{data: data_2_chars_high_density_rejit_best, label: "rejit_best",
color: "#72B377",
},{data: data_2_chars_high_density_rejit_worst, label: "rejit_worst",
color: "#72B377",
},{data: data_2_chars_high_density_v8_amortised, label: "v8_amortised",
color: "#A22EBF",
}, ];

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
      <div style="padding-left: 5em;"> <p>
(3) <strong>Regexp:</strong> <code>([complex]|(regexp)){2,7}abcdefgh(at|the|[e-nd]as well)</code>
<br />
This illustrate the "fast forward" mechanism used in rejit.
</p>
 </div>
    </td>
  </tr>

  <tr>
    <td>
      <div>
        <div id="plot_parallel_3_complex_simple_ff" style="float:left;width:600px;height:400px"></div>
        <div style="float:left;"> <ul id="plot_parallel_3_complex_simple_ff_choices" style="list-style: none;" class="flot_choices"> </ul> </div>
      </div>
      <script language="javascript" type="text/javascript">
        <!--
        $(function () {
          var data_3_complex_simple_ff_pcre_amortised = [[8,4.70588e+07],[16,8.88889e+07],[32,5.91497e+06],[64,1.18299e+07],[128,7.9257e+06],[256,6.52229e+06],[512,7.469e+06],[1024,1.19445e+07],[2048,1.31173e+07],[4096,1.307e+07],[8192,1.31696e+07],[16384,1.31804e+07],[32768,1.31986e+07],[65536,1.32332e+07],[131072,1.32431e+07],[262144,1.32005e+07],[524288,1.31787e+07],[1048576,1.31552e+07],[2097152,1.31751e+07],[4194304,1.30257e+07],[8388608,1.31508e+07],[16777216,1.31706e+07],];
var data_3_complex_simple_ff_pcre_best = [[8,7.27273e+07],[16,1.33333e+08],[32,5.98131e+06],[64,1.19626e+07],[128,7.9602e+06],[256,6.53228e+06],[512,7.47554e+06],[1024,1.19501e+07],[2048,1.31198e+07],[4096,1.30712e+07],[8192,1.31727e+07],[16384,1.31807e+07],[32768,1.31988e+07],[65536,1.32333e+07],[131072,1.32431e+07],[262144,1.32005e+07],[524288,1.31787e+07],[1048576,1.31552e+07],[2097152,1.31751e+07],[4194304,1.30257e+07],[8388608,1.31508e+07],[16777216,1.31706e+07],];
var data_3_complex_simple_ff_pcre_worst = [[8,1.79372e+06],[16,3.61174e+06],[32,3.18725e+06],[64,6.39361e+06],[128,6.17463e+06],[256,5.84742e+06],[512,5.78793e+06],[1024,1.19473e+07],[2048,1.21948e+07],[4096,1.26085e+07],[8192,1.2888e+07],[16384,1.30453e+07],[32768,1.3129e+07],[65536,1.32037e+07],[131072,1.32261e+07],[262144,1.31923e+07],[524288,1.31689e+07],[1048576,1.31563e+07],[2097152,1.31881e+07],[4194304,1.30205e+07],[8388608,1.31399e+07],[16777216,1.3167e+07],];
var data_3_complex_simple_ff_re2_amortised = [[8,7.69231e+06],[16,2.85714e+07],[32,2.99065e+07],[64,4.26667e+07],[128,1.03226e+08],[256,1.13778e+08],[512,9.96109e+07],[1024,1.51479e+08],[2048,1.5295e+08],[4096,1.5876e+08],[8192,1.60754e+08],[16384,1.60643e+08],[32768,1.64341e+08],[65536,1.64725e+08],[131072,1.65889e+08],[262144,1.65612e+08],[524288,1.6621e+08],[1048576,1.6617e+08],[2097152,1.66357e+08],[4194304,1.66333e+08],[8388608,1.66182e+08],[16777216,1.66131e+08],];
var data_3_complex_simple_ff_re2_best = [[8,1.48148e+07],[16,5e+07],[32,4.57143e+07],[64,5.87156e+07],[128,1.26733e+08],[256,1.30612e+08],[512,1.06889e+08],[1024,1.57296e+08],[2048,1.55979e+08],[4096,1.60376e+08],[8192,1.61578e+08],[16384,1.61117e+08],[32768,1.64572e+08],[65536,1.64858e+08],[131072,1.65952e+08],[262144,1.65644e+08],[524288,1.66226e+08],[1048576,1.66178e+08],[2097152,1.66361e+08],[4194304,1.66335e+08],[8388608,1.66184e+08],[16777216,1.66132e+08],];
var data_3_complex_simple_ff_re2_worst = [[8,105778],[16,405988],[32,453386],[64,701139],[128,1.90448e+06],[256,2.85842e+06],[512,4.20258e+06],[1024,6.66667e+06],[2048,1.29236e+07],[4096,2.20085e+07],[8192,3.60849e+07],[16384,5.17335e+07],[32768,7.33606e+07],[65536,9.63963e+07],[131072,1.18759e+08],[262144,1.34601e+08],[524288,1.47904e+08],[1048576,1.55419e+08],[2097152,1.61088e+08],[4194304,1.63357e+08],[8388608,1.6435e+08],[16777216,1.65045e+08],];
var data_3_complex_simple_ff_rejit_amortised = [[8,9.63855e+06],[16,1.97531e+07],[32,3.01887e+07],[64,5.12e+07],[128,9.27536e+07],[256,1.53293e+08],[512,2.86034e+08],[1024,1.13778e+09],[2048,2.0898e+09],[4096,2.51288e+09],[8192,3.69009e+09],[16384,4.66781e+09],[32768,5.48878e+09],[65536,1.19591e+10],[131072,1.15178e+10],[262144,1.09684e+10],[524288,1.22212e+10],[1048576,1.17567e+10],[2097152,1.14324e+10],[4194304,1.1531e+10],[8388608,9.25476e+09],[16777216,8.6555e+09],];
var data_3_complex_simple_ff_rejit_best = [[8,2.28571e+07],[16,4.57143e+07],[32,6.80851e+07],[64,1.16364e+08],[128,2.09836e+08],[256,3.24051e+08],[512,6.2439e+08],[1024,2.3814e+09],[2048,3.86415e+09],[4096,4.22268e+09],[8192,5.49799e+09],[16384,6.04576e+09],[32768,6.43772e+09],[65536,1.3081e+10],[131072,1.20581e+10],[262144,1.12412e+10],[524288,1.23711e+10],[1048576,1.18189e+10],[2097152,1.14624e+10],[4194304,1.15476e+10],[8388608,9.2614e+09],[16777216,8.65854e+09],];
var data_3_complex_simple_ff_rejit_worst = [[8,78771.2],[16,327936],[32,521258],[64,880209],[128,1.73465e+06],[256,3.08657e+06],[512,5.29966e+06],[1024,1.04886e+07],[2048,4.19586e+07],[4096,6.11343e+07],[8192,1.11214e+08],[16384,2.1431e+08],[32768,3.67354e+08],[65536,6.58322e+08],[131072,2.14415e+09],[262144,3.12933e+09],[524288,5.20488e+09],[1048576,7.412e+09],[2097152,8.84912e+09],[4194304,9.89153e+09],[8388608,8.27189e+09],[16777216,8.20153e+09],];
var data_3_complex_simple_ff_v8_amortised = [[8,40000000],[16,106666666.66666666],[32,213333333.3333333],[64,182857142.85714287],[128,256000000],[256,301176470.58823526],[512,284444444.4444444],[1024,256000000],[2048,210051282.05128208],[4096,182857142.85714287],[8192,168041025.64102563],[16384,161498275.01232135],[32768,155409058.57244486],[65536,155501245.69937122],[131072,154202352.94117644],[262144,154202352.94117644],[524288,154202352.94117644],[1048576,154202352.94117644],[2097152,154315820.45621783],[4194304,154942888.8067972],[8388608,154032464.19390377],[16777216,154728543.76095176],];

          var datasets = [ {data: data_3_complex_simple_ff_pcre_amortised, label: "pcre_amortised",
color: "#DEBD00",
},{data: data_3_complex_simple_ff_pcre_best, label: "pcre_best",
color: "#E0D48D",
},{data: data_3_complex_simple_ff_pcre_worst, label: "pcre_worst",
color: "#E0D48D",
},{data: data_3_complex_simple_ff_re2_amortised, label: "re2_amortised",
color: "#277AD9",
},{data: data_3_complex_simple_ff_re2_best, label: "re2_best",
color: "#94B8E0",
},{data: data_3_complex_simple_ff_re2_worst, label: "re2_worst",
color: "#94B8E0",
},{data: data_3_complex_simple_ff_rejit_amortised, label: "rejit_amortised",
color: "#00940A",
},{data: data_3_complex_simple_ff_rejit_best, label: "rejit_best",
color: "#72B377",
},{data: data_3_complex_simple_ff_rejit_worst, label: "rejit_worst",
color: "#72B377",
},{data: data_3_complex_simple_ff_v8_amortised, label: "v8_amortised",
color: "#A22EBF",
}, ];

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
      <div style="padding-left: 5em;"> <p>
(4) <strong>Regexp:</strong> <code>(12345678|abcdefghijkl)</code>
<br />
Search for an alternation of two long words with no common sub-string.
</p>
 </div>
    </td>
  </tr>

  <tr>
    <td>
      <div>
        <div id="plot_parallel_4_alternation_2_long" style="float:left;width:600px;height:400px"></div>
        <div style="float:left;"> <ul id="plot_parallel_4_alternation_2_long_choices" style="list-style: none;" class="flot_choices"> </ul> </div>
      </div>
      <script language="javascript" type="text/javascript">
        <!--
        $(function () {
          var data_4_alternation_2_long_pcre_amortised = [[8,9.1954e+06],[16,1.0596e+07],[32,1.08475e+07],[64,1.17002e+07],[128,1.19292e+07],[256,1.20869e+07],[512,1.19181e+07],[1024,1.50877e+07],[2048,2.36763e+07],[4096,2.58227e+07],[8192,2.56497e+07],[16384,2.52773e+07],[32768,2.55994e+07],[65536,2.56284e+07],[131072,2.5474e+07],[262144,2.59357e+07],[524288,2.58257e+07],[1048576,2.55508e+07],[2097152,2.57993e+07],[4194304,2.58587e+07],[8388608,2.54609e+07],[16777216,2.55235e+07],];
var data_4_alternation_2_long_pcre_best = [[8,9.52381e+06],[16,1.08108e+07],[32,1.09966e+07],[64,1.17647e+07],[128,1.19626e+07],[256,1.21097e+07],[512,1.19292e+07],[1024,1.50943e+07],[2048,2.36818e+07],[4096,2.5826e+07],[8192,2.56505e+07],[16384,2.52781e+07],[32768,2.55996e+07],[65536,2.56286e+07],[131072,2.54741e+07],[262144,2.59357e+07],[524288,2.58258e+07],[1048576,2.55508e+07],[2097152,2.57993e+07],[4194304,2.58587e+07],[8388608,2.54609e+07],[16777216,2.55235e+07],];
var data_4_alternation_2_long_pcre_worst = [[8,2.52366e+06],[16,4.25532e+06],[32,6.26223e+06],[64,8.24742e+06],[128,9.93018e+06],[256,1.09448e+07],[512,1.13074e+07],[1024,1.18642e+07],[2048,2.4208e+07],[4096,2.42095e+07],[8192,2.49353e+07],[16384,2.47803e+07],[32768,2.53412e+07],[65536,2.55003e+07],[131072,2.53919e+07],[262144,2.59207e+07],[524288,2.5839e+07],[1048576,2.55653e+07],[2097152,2.58136e+07],[4194304,2.58256e+07],[8388608,2.54563e+07],[16777216,2.5514e+07],];
var data_4_alternation_2_long_re2_amortised = [[8,1.48148e+07],[16,2.58065e+07],[32,4.26667e+07],[64,6.27451e+07],[128,8.47682e+07],[256,1e+08],[512,1.07563e+08],[1024,1.14413e+08],[2048,1.17972e+08],[4096,1.19906e+08],[8192,1.21561e+08],[16384,2.35911e+08],[32768,2.54529e+08],[65536,2.58372e+08],[131072,2.58816e+08],[262144,2.58351e+08],[524288,2.58036e+08],[1048576,2.58539e+08],[2097152,2.57568e+08],[4194304,2.5852e+08],[8388608,2.57381e+08],[16777216,2.56969e+08],];
var data_4_alternation_2_long_re2_best = [[8,2e+07],[16,3.40426e+07],[32,5.33333e+07],[64,7.35632e+07],[128,9.34307e+07],[256,1.06224e+08],[512,1.11063e+08],[1024,1.16364e+08],[2048,1.1907e+08],[4096,1.20435e+08],[8192,1.21832e+08],[16384,2.36183e+08],[32768,2.54687e+08],[65536,2.58453e+08],[131072,2.58851e+08],[262144,2.58372e+08],[524288,2.58046e+08],[1048576,2.58543e+08],[2097152,2.57571e+08],[4194304,2.58521e+08],[8388608,2.57382e+08],[16777216,2.56969e+08],];
var data_4_alternation_2_long_re2_worst = [[8,310198],[16,616333],[32,1.15691e+06],[64,2.25114e+06],[128,4.29963e+06],[256,8.22094e+06],[512,1.42897e+07],[1024,2.48182e+07],[2048,4.00391e+07],[4096,5.95003e+07],[8192,7.93952e+07],[16384,1.94469e+08],[32768,2.06947e+08],[65536,2.30217e+08],[131072,2.47506e+08],[262144,2.51247e+08],[524288,2.53819e+08],[1048576,2.56896e+08],[2097152,2.57384e+08],[4194304,2.5799e+08],[8388608,2.56681e+08],[16777216,2.56739e+08],];
var data_4_alternation_2_long_rejit_amortised = [[8,1.81818e+07],[16,3.33333e+07],[32,5.71429e+07],[64,9.01408e+07],[128,1.85507e+08],[256,4.19672e+08],[512,7.64179e+08],[1024,1.1907e+09],[2048,1.45248e+09],[4096,2.1672e+09],[8192,2.56e+09],[16384,2.98978e+09],[32768,3.19376e+09],[65536,3.39213e+09],[131072,7.13511e+09],[262144,6.53889e+09],[524288,6.60146e+09],[1048576,6.29284e+09],[2097152,6.2609e+09],[4194304,6.29407e+09],[8388608,5.74381e+09],[16777216,5.72719e+09],];
var data_4_alternation_2_long_rejit_best = [[8,4.21053e+07],[16,6.95652e+07],[32,1e+08],[64,1.42222e+08],[128,2.90909e+08],[256,7.11111e+08],[512,1.21905e+09],[1024,1.73559e+09],[2048,1.81239e+09],[4096,2.51288e+09],[8192,2.78639e+09],[16384,3.1387e+09],[32768,3.27353e+09],[65536,3.43841e+09],[131072,7.18596e+09],[262144,6.56016e+09],[524288,6.61228e+09],[1048576,6.29738e+09],[2097152,6.26333e+09],[4194304,6.29539e+09],[8388608,5.74476e+09],[16777216,5.72769e+09],];
var data_4_alternation_2_long_rejit_worst = [[8,295967],[16,603774],[32,1.2021e+06],[64,2.35814e+06],[128,4.7513e+06],[256,9.40831e+06],[512,1.85239e+07],[1024,3.74269e+07],[2048,7.36956e+07],[4096,1.43317e+08],[8192,2.74255e+08],[16384,5.1248e+08],[32768,8.83235e+08],[65536,1.40786e+09],[131072,2.07787e+09],[262144,4.81175e+09],[524288,5.1522e+09],[1048576,5.66308e+09],[2097152,5.88724e+09],[4194304,6.05055e+09],[8388608,5.50264e+09],[16777216,5.57168e+09],];
var data_4_alternation_2_long_v8_amortised = [[8,160000000],[16,106666666.66666666],[32,320000000],[64,640000000],[128,1280000000],[256,1706666666.6666665],[512,3413333333.333333],[1024,3413333333.333333],[2048,4096000000],[4096,5120000000],[8192,5120000000],[16384,5285161290.32258],[32768,3744914285.714286],[65536,2590355731.2252965],[131072,2184533333.333333],[262144,2184533333.333333],[524288,2184533333.333333],[1048576,2139951020.4081633],[2097152,2184533333.333333],[4194304,2184533333.333333],[8388608,2107690452.2613068],[16777216,2107690452.2613068],];

          var datasets = [ {data: data_4_alternation_2_long_pcre_amortised, label: "pcre_amortised",
color: "#DEBD00",
},{data: data_4_alternation_2_long_pcre_best, label: "pcre_best",
color: "#E0D48D",
},{data: data_4_alternation_2_long_pcre_worst, label: "pcre_worst",
color: "#E0D48D",
},{data: data_4_alternation_2_long_re2_amortised, label: "re2_amortised",
color: "#277AD9",
},{data: data_4_alternation_2_long_re2_best, label: "re2_best",
color: "#94B8E0",
},{data: data_4_alternation_2_long_re2_worst, label: "re2_worst",
color: "#94B8E0",
},{data: data_4_alternation_2_long_rejit_amortised, label: "rejit_amortised",
color: "#00940A",
},{data: data_4_alternation_2_long_rejit_best, label: "rejit_best",
color: "#72B377",
},{data: data_4_alternation_2_long_rejit_worst, label: "rejit_worst",
color: "#72B377",
},{data: data_4_alternation_2_long_v8_amortised, label: "v8_amortised",
color: "#A22EBF",
}, ];

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
      <div style="padding-left: 5em;"> <p>
(5) <strong>Regexp:</strong> <code>(12345678|xyz)</code>
<br />
Search for an alternation of two words with no common sub-string,
including one short word.
</p>
 </div>
    </td>
  </tr>

  <tr>
    <td>
      <div>
        <div id="plot_parallel_5_alternation_2_short" style="float:left;width:600px;height:400px"></div>
        <div style="float:left;"> <ul id="plot_parallel_5_alternation_2_short_choices" style="list-style: none;" class="flot_choices"> </ul> </div>
      </div>
      <script language="javascript" type="text/javascript">
        <!--
        $(function () {
          var data_5_alternation_2_short_pcre_amortised = [[8,9.09091e+06],[16,1.04575e+07],[32,1.09966e+07],[64,1.12676e+07],[128,1.17539e+07],[256,1.19514e+07],[512,1.19375e+07],[1024,1.95048e+07],[2048,2.41795e+07],[4096,2.59273e+07],[8192,2.55322e+07],[16384,2.55668e+07],[32768,2.55316e+07],[65536,2.58842e+07],[131072,2.57129e+07],[262144,2.56034e+07],[524288,2.58432e+07],[1048576,2.55813e+07],[2097152,2.55407e+07],[4194304,2.5599e+07],[8388608,2.5511e+07],[16777216,2.58418e+07],];
var data_5_alternation_2_short_pcre_best = [[8,9.41176e+06],[16,1.06667e+07],[32,1.11111e+07],[64,1.13274e+07],[128,1.17864e+07],[256,1.19682e+07],[512,1.19459e+07],[1024,1.95159e+07],[2048,2.41852e+07],[4096,2.5929e+07],[8192,2.5533e+07],[16384,2.55676e+07],[32768,2.5532e+07],[65536,2.58843e+07],[131072,2.5713e+07],[262144,2.56034e+07],[524288,2.58432e+07],[1048576,2.55813e+07],[2097152,2.55407e+07],[4194304,2.5599e+07],[8388608,2.55111e+07],[16777216,2.58418e+07],];
var data_5_alternation_2_short_pcre_worst = [[8,3.01887e+06],[16,4.90798e+06],[32,7.00219e+06],[64,8.57909e+06],[128,1.02236e+07],[256,1.09824e+07],[512,1.14082e+07],[1024,1.19208e+07],[2048,2.39196e+07],[4096,2.42453e+07],[8192,2.52178e+07],[16384,2.49985e+07],[32768,2.544e+07],[65536,2.58532e+07],[131072,2.56677e+07],[262144,2.55804e+07],[524288,2.58238e+07],[1048576,2.55775e+07],[2097152,2.55305e+07],[4194304,2.55985e+07],[8388608,2.54999e+07],[16777216,2.5855e+07],];
var data_5_alternation_2_short_re2_amortised = [[8,1.53846e+07],[16,2.71186e+07],[32,4.38356e+07],[64,6.46465e+07],[128,8.70748e+07],[256,1.04065e+08],[512,1.12775e+08],[1024,1.17566e+08],[2048,1.19836e+08],[4096,1.21905e+08],[8192,2.01129e+08],[16384,2.37277e+08],[32768,2.55282e+08],[65536,2.55073e+08],[131072,2.58453e+08],[262144,2.58601e+08],[524288,2.58234e+08],[1048576,2.58686e+08],[2097152,2.58386e+08],[4194304,2.58056e+08],[8388608,2.56837e+08],[16777216,2.56049e+08],];
var data_5_alternation_2_short_re2_best = [[8,2.05128e+07],[16,3.47826e+07],[32,5.2459e+07],[64,7.44186e+07],[128,9.55224e+07],[256,1.09871e+08],[512,1.15837e+08],[1024,1.19347e+08],[2048,1.20684e+08],[4096,1.22378e+08],[8192,2.01724e+08],[16384,2.37484e+08],[32768,2.55401e+08],[65536,2.55133e+08],[131072,2.58489e+08],[262144,2.58619e+08],[524288,2.58243e+08],[1048576,2.5869e+08],[2097152,2.58388e+08],[4194304,2.58057e+08],[8388608,2.56838e+08],[16777216,2.5605e+08],];
var data_5_alternation_2_short_re2_worst = [[8,363636],[16,715244],[32,1.40598e+06],[64,2.6534e+06],[128,5.15505e+06],[256,1.0055e+07],[512,1.84904e+07],[1024,3.1411e+07],[2048,5.01961e+07],[4096,7.13465e+07],[8192,8.97262e+07],[16384,2.0914e+08],[32768,2.22156e+08],[65536,2.34669e+08],[131072,2.50995e+08],[262144,2.53362e+08],[524288,2.56069e+08],[1048576,2.57502e+08],[2097152,2.57904e+08],[4194304,2.58411e+08],[8388608,2.57026e+08],[16777216,2.56406e+08],];
var data_5_alternation_2_short_rejit_amortised = [[8,1.86047e+07],[16,3.47826e+07],[32,6.03774e+07],[64,9.01408e+07],[128,1.85507e+08],[256,3.55556e+08],[512,6.48101e+08],[1024,1.10108e+09],[2048,1.86182e+09],[4096,2.32727e+09],[8192,2.67712e+09],[16384,3.09716e+09],[32768,3.30323e+09],[65536,3.35394e+09],[131072,3.42403e+09],[262144,7.14484e+09],[524288,6.88223e+09],[1048576,6.45357e+09],[2097152,6.25381e+09],[4194304,6.21609e+09],[8388608,5.69495e+09],[16777216,5.66871e+09],];
var data_5_alternation_2_short_rejit_best = [[8,4.70588e+07],[16,7.27273e+07],[32,1.10345e+08],[64,1.45455e+08],[128,2.90909e+08],[256,5.33333e+08],[512,9.48148e+08],[1024,1.55152e+09],[2048,2.46747e+09],[4096,2.69474e+09],[8192,2.9153e+09],[16384,3.25079e+09],[32768,3.38862e+09],[65536,3.39741e+09],[131072,3.44654e+09],[262144,7.16828e+09],[524288,6.89308e+09],[1048576,6.45874e+09],[2097152,6.25623e+09],[4194304,6.21756e+09],[8388608,5.69596e+09],[16777216,5.6692e+09],];
var data_5_alternation_2_short_rejit_worst = [[8,305344],[16,614439],[32,1.22841e+06],[64,2.44461e+06],[128,4.72499e+06],[256,9.71168e+06],[512,1.92989e+07],[1024,3.81236e+07],[2048,7.56278e+07],[4096,1.47817e+08],[8192,2.87237e+08],[16384,5.20954e+08],[32768,9.09969e+08],[65536,1.4115e+09],[131072,1.96304e+09],[262144,3.35137e+09],[524288,5.46133e+09],[1048576,5.76394e+09],[2097152,5.88377e+09],[4194304,5.9322e+09],[8388608,5.44428e+09],[16777216,5.51367e+09],];
var data_5_alternation_2_short_v8_amortised = [[8,80000000],[16,160000000],[32,320000000],[64,640000000],[128,853333333.3333333],[256,1280000000],[512,2048000000],[1024,2048000000],[2048,2560000000],[4096,2642580645.16129],[8192,2520615384.6153846],[16384,2374492753.6231885],[32768,2114064516.1290321],[65536,1982934947.0499246],[131072,1872457142.857143],[262144,1872457142.857143],[524288,1807889655.1724136],[1048576,1807889655.1724136],[2097152,1823610434.7826085],[4194304,1839607017.5438595],[8388608,1792437606.8376071],[16777216,1788615778.251599],];

          var datasets = [ {data: data_5_alternation_2_short_pcre_amortised, label: "pcre_amortised",
color: "#DEBD00",
},{data: data_5_alternation_2_short_pcre_best, label: "pcre_best",
color: "#E0D48D",
},{data: data_5_alternation_2_short_pcre_worst, label: "pcre_worst",
color: "#E0D48D",
},{data: data_5_alternation_2_short_re2_amortised, label: "re2_amortised",
color: "#277AD9",
},{data: data_5_alternation_2_short_re2_best, label: "re2_best",
color: "#94B8E0",
},{data: data_5_alternation_2_short_re2_worst, label: "re2_worst",
color: "#94B8E0",
},{data: data_5_alternation_2_short_rejit_amortised, label: "rejit_amortised",
color: "#00940A",
},{data: data_5_alternation_2_short_rejit_best, label: "rejit_best",
color: "#72B377",
},{data: data_5_alternation_2_short_rejit_worst, label: "rejit_worst",
color: "#72B377",
},{data: data_5_alternation_2_short_v8_amortised, label: "v8_amortised",
color: "#A22EBF",
}, ];

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
      <div style="padding-left: 5em;"> <p>
(6) <strong>Regexp:</strong> <code>(abcd--|abcd____)</code>
<br />
Search for an alternation of two words sharing a common prefix.
</p>
 </div>
    </td>
  </tr>

  <tr>
    <td>
      <div>
        <div id="plot_parallel_6_alternation_2_common_prefix" style="float:left;width:600px;height:400px"></div>
        <div style="float:left;"> <ul id="plot_parallel_6_alternation_2_common_prefix_choices" style="list-style: none;" class="flot_choices"> </ul> </div>
      </div>
      <script language="javascript" type="text/javascript">
        <!--
        $(function () {
          var data_6_alternation_2_common_prefix_pcre_amortised = [[8,3.80952e+07],[16,6.66667e+07],[32,1.14286e+08],[64,1.56098e+08],[128,2.06452e+08],[256,1.88235e+08],[512,1.68421e+08],[1024,1.64366e+08],[2048,1.62025e+08],[4096,1.59813e+08],[8192,1.67047e+08],[16384,2.45416e+08],[32768,3.47082e+08],[65536,3.60147e+08],[131072,3.6296e+08],[262144,3.61299e+08],[524288,3.56889e+08],[1048576,3.5861e+08],[2097152,3.59346e+08],[4194304,3.60165e+08],[8388608,3.59473e+08],[16777216,3.59185e+08],];
var data_6_alternation_2_common_prefix_pcre_best = [[8,4.21053e+07],[16,7.61905e+07],[32,1.28e+08],[64,1.68421e+08],[128,2.16949e+08],[256,1.92481e+08],[512,1.701e+08],[1024,1.65161e+08],[2048,1.62411e+08],[4096,1.59938e+08],[8192,1.67115e+08],[16384,2.45527e+08],[32768,3.47119e+08],[65536,3.60167e+08],[131072,3.6298e+08],[262144,3.61304e+08],[524288,3.56894e+08],[1048576,3.58611e+08],[2097152,3.59347e+08],[4194304,3.60166e+08],[8388608,3.59474e+08],[16777216,3.59185e+08],];
var data_6_alternation_2_common_prefix_pcre_worst = [[8,3.8835e+06],[16,7.9602e+06],[32,1.54589e+07],[64,2.93578e+07],[128,5.26749e+07],[256,7.75758e+07],[512,1.04065e+08],[1024,1.26108e+08],[2048,1.42124e+08],[4096,1.4938e+08],[8192,1.61038e+08],[16384,1.66741e+08],[32768,3.31861e+08],[65536,3.42403e+08],[131072,3.54306e+08],[262144,3.56901e+08],[524288,3.56051e+08],[1048576,3.58611e+08],[2097152,3.5916e+08],[4194304,3.6003e+08],[8388608,3.58198e+08],[16777216,3.58441e+08],];
var data_6_alternation_2_common_prefix_re2_amortised = [[8,1.6e+07],[16,3.2e+07],[32,6.27451e+07],[64,1.20755e+08],[128,2.24561e+08],[256,3.93846e+08],[512,5.17172e+08],[1024,8.192e+08],[2048,1.01386e+09],[4096,1.13463e+09],[8192,1.25644e+09],[16384,1.282e+09],[32768,1.03992e+09],[65536,1.45023e+09],[131072,1.57029e+09],[262144,1.50157e+09],[524288,1.44174e+09],[1048576,1.43637e+09],[2097152,1.42806e+09],[4194304,1.43311e+09],[8388608,1.41028e+09],[16777216,1.40882e+09],];
var data_6_alternation_2_common_prefix_re2_best = [[8,2.35294e+07],[16,4.70588e+07],[32,9.14286e+07],[64,1.68421e+08],[128,3.2e+08],[256,5.22449e+08],[512,6.16867e+08],[1024,9.3945e+08],[2048,1.10108e+09],[4096,1.1907e+09],[8192,1.28805e+09],[16384,1.29826e+09],[32768,1.04523e+09],[65536,1.45313e+09],[131072,1.57198e+09],[262144,1.50234e+09],[524288,1.44213e+09],[1048576,1.43656e+09],[2097152,1.42816e+09],[4194304,1.43316e+09],[8388608,1.41033e+09],[16777216,1.40885e+09],];
var data_6_alternation_2_common_prefix_re2_worst = [[8,310078],[16,631413],[32,1.25638e+06],[64,2.51276e+06],[128,4.97861e+06],[256,9.88799e+06],[512,1.90974e+07],[1024,3.69009e+07],[2048,6.92594e+07],[4096,1.31408e+08],[8192,2.37656e+08],[16384,3.93846e+08],[32768,5.18563e+08],[65536,1.21116e+09],[131072,1.0007e+09],[262144,1.28464e+09],[524288,1.3536e+09],[1048576,1.39238e+09],[2097152,1.40758e+09],[4194304,1.42022e+09],[8388608,1.39993e+09],[16777216,1.40425e+09],];
var data_6_alternation_2_common_prefix_rejit_amortised = [[8,1.95122e+07],[16,3.47826e+07],[32,6.15385e+07],[64,1.01587e+08],[128,2.06452e+08],[256,3.87879e+08],[512,8.12698e+08],[1024,1.16364e+09],[2048,1.45248e+09],[4096,2.19037e+09],[8192,2.61725e+09],[16384,3.02847e+09],[32768,3.2157e+09],[65536,3.39213e+09],[131072,3.45563e+09],[262144,7.28785e+09],[524288,6.95988e+09],[1048576,6.50199e+09],[2097152,6.27815e+09],[4194304,6.28106e+09],[8388608,5.74704e+09],[16777216,5.72902e+09],];
var data_6_alternation_2_common_prefix_rejit_best = [[8,5e+07],[16,8e+07],[32,1.14286e+08],[64,1.68421e+08],[128,3.36842e+08],[256,7.75758e+08],[512,1.34737e+09],[1024,1.65161e+09],[2048,1.79649e+09],[4096,2.5284e+09],[8192,2.84444e+09],[16384,3.17519e+09],[32768,3.29327e+09],[65536,3.4366e+09],[131072,3.4804e+09],[262144,7.31225e+09],[524288,6.97191e+09],[1048576,6.50724e+09],[2097152,6.28078e+09],[4194304,6.28247e+09],[8388608,5.74806e+09],[16777216,5.72955e+09],];
var data_6_alternation_2_common_prefix_rejit_worst = [[8,303605],[16,602864],[32,1.21951e+06],[64,2.42149e+06],[128,4.80661e+06],[256,9.58443e+06],[512,1.88097e+07],[1024,3.78418e+07],[2048,7.44186e+07],[4096,1.44276e+08],[8192,2.77413e+08],[16384,5.14735e+08],[32768,8.88744e+08],[65536,1.41394e+09],[131072,1.94787e+09],[262144,3.52771e+09],[524288,5.56451e+09],[1048576,5.78876e+09],[2097152,5.87059e+09],[4194304,6.01343e+09],[8388608,5.51139e+09],[16777216,5.5849e+09],];
var data_6_alternation_2_common_prefix_v8_amortised = [[8,160000000],[16,160000000],[32,320000000],[64,640000000],[128,1280000000],[256,1706666666.6666665],[512,3413333333.333333],[1024,4096000000],[2048,4551111111.111111],[4096,5120000000],[8192,5120000000],[16384,4964848484.848485],[32768,4551111111.111111],[65536,4057956656.3467493],[131072,3276800000],[262144,3744914285.714286],[524288,3744914285.714286],[1048576,3615779310.344827],[2097152,3554494915.2542377],[4194304,3647220869.565217],[8388608,3452102057.6131687],[16777216,3437954098.3606563],];

          var datasets = [ {data: data_6_alternation_2_common_prefix_pcre_amortised, label: "pcre_amortised",
color: "#DEBD00",
},{data: data_6_alternation_2_common_prefix_pcre_best, label: "pcre_best",
color: "#E0D48D",
},{data: data_6_alternation_2_common_prefix_pcre_worst, label: "pcre_worst",
color: "#E0D48D",
},{data: data_6_alternation_2_common_prefix_re2_amortised, label: "re2_amortised",
color: "#277AD9",
},{data: data_6_alternation_2_common_prefix_re2_best, label: "re2_best",
color: "#94B8E0",
},{data: data_6_alternation_2_common_prefix_re2_worst, label: "re2_worst",
color: "#94B8E0",
},{data: data_6_alternation_2_common_prefix_rejit_amortised, label: "rejit_amortised",
color: "#00940A",
},{data: data_6_alternation_2_common_prefix_rejit_best, label: "rejit_best",
color: "#72B377",
},{data: data_6_alternation_2_common_prefix_rejit_worst, label: "rejit_worst",
color: "#72B377",
},{data: data_6_alternation_2_common_prefix_v8_amortised, label: "v8_amortised",
color: "#A22EBF",
}, ];

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
</tr></table>
