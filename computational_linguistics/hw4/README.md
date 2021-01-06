## Introduction

Since I [already programmed](https://github.com/zouharvi/mff-student/blob/master/1920-2/statistical_machine_translation/ibm1.py) the IBM Model 1 for a course on Statistical Machine Translation, the submitted code is just a polished and commented version of it.
I am also attaching a draft of the paper _Leveraging Neural MT Output for Word Alignment_, which you may find interesting. Please don't share outside of your team as it is currently in a review process. It covers three topics: (1) using an NMT to get word alignments, (2) various hard alignment extractor methods and (3) combination of all in small neural network. The extractor methods are in brackets. Hard alignment extractors are reused from this projects and this document describes them very superficially and without mathematical notation. The original version was based on Kohen's pseudocode, which he recently [revised](http://mt-class.org/jhu/slides/lecture-ibm-model1.pdf). 
For the graphics I also used [Slow Align](https://vilda.net/s/slowalign/), a small tool for displaying word alignment (needs major reworking).

## Baseline

Baseline observed.

```
Precision = 0.243110
Recall    = 0.544379
AER       = 0.681684
```

### Custom Models

### IBM1 (A0)

Even though the code uses numpy whenever possible, it still takes considerable amount of time to run on all 100k examples (~10 minutes for 5 steps). This version simply takes the maximum aligned token. This makes a strong assumption that every token is aligned to exactly one other token.

Results for `A0` with 5 EM steps.

```
Precision = 0.615811
Recall    = 0.801775
AER       = 0.324835
```

### Threshold (A2)

Results for  `A2(0.35)` with 7 EM steps.

```
Precision = 0.685567
Recall    = 0.778107
AER       = 0.280435
```


### Proportion of best (A3)

This extraction method examines all target tokens and selects the best one. All other alignments which have scores of at least `alpha` times the maximum one are taken. Here `alpha` is a parameter. In the following example consider `alpha=0.4`, then _menu_ would align to _Popupmenü_ and _Dialogfeld_ (mistakenly) and not to others. 

<img src='img/align_a3.svg' width='1000px' height='250px'>

Results for `A3(0.98)` with 7 EM steps.

```
Precision = 0.537281
Recall    = 0.878698
AER       = 0.370400
```

### Proportion of reverse best (A4)

The same approach can be done from the target token perspective. Suprisingly, this works a bit better than A3. In the example for `alpha=0.5`, _Popupmenü_ would align to _pop-up_, _menu_ and _dialog_ (mistakenly).

<img src='img/align_a4.svg' width='1000px' height='250px'>


Results for `A4(0.98)` with 7 EM steps.

```
Precision = 0.590789
Recall    = 0.846154
AER       = 0.330601
```

### Intersection (A2*A3*A4)

As described by Koehn, intersection in alignment directions increases precision. The intersection of different alignment methods also improves the performance. Intersecting the previous two examples leads to _Popupmenü_ being aligned to _pop-up_ and _menu_.

<img src='img/align_int.svg' width='1000px' height='250px'>

The main disadvantage of both A3 and A4 is that even with `alpha=1`, they insist that every token is aligned. Thresholding the alignments improves the performance. Results for `A2(0.25) \cap A3(0.98) \cap A4(0.98)` with 7 EM steps.

```
Precision = 0.758483
Recall    = 0.784024
AER       = 0.231228
```

### Intersection And Reverse (A2'*A3'*A4' * A2*A3*A4)

The final step combines the previous approach by computing the alignment from the other direction and intersecting it as well. In this this version only alignment which are within 6 places of each other are considered. This promotes diagonal alignment. (`src/intersect_reverse.sh`)

Results for `A2(0.25) \cap A3(0.9) \cap A4(0.9) \cap A2'(0.25) \cap A3'(0.9) \cap A4'(0.9)` with 5 EM steps.

```
Precision = 0.850633
Recall    = 0.704142
AER       = 0.216917
```

The third sentence of the dataset:

```
 Alignment 2  KEY: ( ) = guessed, * = sure, ? = possible
  ---------------------------------------------------------------------------
 |(*)                                                                         | ils
 |   (*)            ( )                                                       | sont
 |      (*)            ( )                                                    | restreints
 |         (*)                                                                | par
 |                                                                            | certaines
 |            (*)                                                             | limites
 |               (*)                                                          | qui
 |                ?  ?  ?                                                     | ont
 |                ?  ?  ?                                                     | été
 |                ?  ?  ?                                                     | fixées
 |                         ?  ? (?)                                           | pour
 |                                 (*)                                        | garantir
 |                                    (*)                                     | que
 |                                        *                                   | la
 |                                          (*)                               | liberté
 |                                              *                             | de
 |                                                 *                          | une
 |                                                   (*)                      | personne
 |                                                      ( ) *                 | ne
 |                                                       ?     ?              | empiète
 |                                                         (*)                | pas
 |                                                       ?     ?              | sur
 |                                                                *           | celle
 |                                                                   *        | de
 |                                                                      *     | une
 |                                                                     (*)    | autre
 |                                                                        (*) | .
  ---------------------------------------------------------------------------
   t  a  c  b  l  w  a  i  i  o  t  e  t  t  f  o  o  p  d  n  v  t  o  a  . 
   h  r  o  y  i  h  r  m  n  r  o  n  h  h  r  f  n  e  o  o  i  h  f  n    
   e  e  n     m  i  e  p     d     s  a  e  e     e  r  e  t  o  a     o    
   y     s     i  c     o     e     u  t     e        s  s     l  t     t    
         t     t  h     s     r     r        d        o        a        h    
         r     s        e           e        o        n        t        e    
         a              d                    m                 e        r    
         i                                                                   
         n                                                                   
         e                                                                   
         d                                                                   
```

## fast_align

The script `src/fastalign.sh` runs fast_align with best practice arguments (`-v` Dirichlet prior and `-do` close to diagonal). EM steps are default 5.

```
Precision = 0.788122
Recall    = 0.831361
AER       = 0.196670
```

Alignment of the third sentence in the data by fast_align. Neither the current implementation nor fast_align guessed the _ont été fixées_ phrase correctly, but fast_align performed slightly better.

```
  Alignment 2  KEY: ( ) = guessed, * = sure, ? = possible
  ---------------------------------------------------------------------------
 |(*)                                                                         | ils
 |   (*)            ( )                                                       | sont
 |      (*)                                                                   | restreints
 |         (*)         ( )                                                    | par
 |                                                                            | certaines
 |            (*)                                                             | limites
 |               (*)                                                          | qui
 |                ?  ?  ?                                                     | ont
 |                ?  ?  ?                                                     | été
 |                ?  ?  ?                                                     | fixées
 |                         ? (?)(?)                                           | pour
 |                                 (*)                                        | garantir
 |                                    (*)                                     | que
 |                                       (*)                                  | la
 |                                          (*)                               | liberté
 |                                             (*)                            | de
 |                                                 *                          | une
 |                                                ( )(*)                      | personne
 |                                                      ( ) *                 | ne
 |                                                       ?     ?              | empiète
 |                                                         (*)                | pas
 |                                                       ?     ?              | sur
 |                                                               (*)          | celle
 |                                                            ( )   (*)       | de
 |                                                                      *     | une
 |                                                                     (*)    | autre
 |                                                                        (*) | .
  ---------------------------------------------------------------------------
   t  a  c  b  l  w  a  i  i  o  t  e  t  t  f  o  o  p  d  n  v  t  o  a  . 
   h  r  o  y  i  h  r  m  n  r  o  n  h  h  r  f  n  e  o  o  i  h  f  n    
   e  e  n     m  i  e  p     d     s  a  e  e     e  r  e  t  o  a     o    
   y     s     i  c     o     e     u  t     e        s  s     l  t     t    
         t     t  h     s     r     r        d        o        a        h    
         r     s        e           e        o        n        t        e    
         a              d                    m                 e        r    
         i                                                                   
         n                                                                   
         e                                                                   
         d                                                                   
```

Without `-d` the results are:

```
Precision = 0.645161
Recall    = 0.766272
AER       = 0.313448
```