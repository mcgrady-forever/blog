---
title: jvm04
date: 2017-10-11 23:43:26
tags:
---

jstat -gcutil 25354 500 3
  S0     S1     E      O      P     YGC     YGCT    FGC    FGCT     GCT   
 96.48   0.00  29.45  57.62  89.07  17974  409.657    18    7.916  417.572
 96.48   0.00  30.32  57.62  89.07  17974  409.657    18    7.916  417.572
 96.48   0.00  31.14  57.62  89.07  17974  409.657    18    7.916  417.572

after restart:
 3733 work      20   0 6525m 1.4g  18m S  9.3  2.3   1:45.08 java 

 jstat -gcutil 3733 500 3
  S0     S1     E      O      P     YGC     YGCT    FGC    FGCT     GCT   
  0.00  21.84  13.14  29.65  97.65     19    0.403     2    0.162    0.565
  0.00  21.84  14.02  29.65  97.65     19    0.403     2    0.162    0.565
  0.00  21.84  14.42  29.65  97.65     19    0.403     2    0.162    0.565