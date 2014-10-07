

Normalisation of CAGE data by sub-sampling
=========================================

The number of different genes detected when sequencing a transcriptome library
increases slower and slower as reads are added: by definition the first read is
always new; the second has high chances to be different from the first, etc.,
but after millions of them each read will have increasing chances to be
identical to another read already sequenced.  The consequence of this is that
analysing the same library sequenced _deeply_, for instance on HiSeq, or
_shallowly_, for instance on MiSeq, will not yield the same result in terms of
gene discovery, etc.

When comparing libraries it can be useful to normalise them to the same
_depth_, that is, to use the same number of reads for each library.  There are
mainly two ways: either input a fixed number of reads in the alignment
pipeline, or sub-sample a fixed number of alignments after using all the reads.
This tutorial shows how do the second solution for CAGE data using `R`.  It is
related to the analysis presented in the supplementary note number 4 of the
FANTOM 5 paper ([Forrest _et al._, 2014][F5])

[F5]: http://dx.doi.org/10.1038/nature13182 "Forrest et al., 2014"

See the main [README](../README.md) for general recommendations on how or what
to prepare before running this tutorial.

Busy people familiar with CAGE and R can skip the tutorial and read the manual
of the `rrarefy` command of the `vegan` package, which does all the work.

<a id="data-prep">Data download and preparation</a>
---------------------------------------------------

### Information and download

The data downloaded here is the count of CAGE tags in the FANTOM5 CAGE peaks
for all the _Phase 1_ libraries of the FANTOM 5 project.  See the [README][F5 README]
file for more information on that file

[F5 README]: http://fantom.gsc.riken.jp/5/datafiles/latest/extra/CAGE_peaks/00_readme.txt


```bash
wget --quiet --timestamping http://fantom.gsc.riken.jp/5/datafiles/latest/extra/CAGE_peaks/hg19.cage_peak_counts.osc.txt.gz
echo "6be129d01ce86f3d9fdb3cf26ea3e5ac  hg19.cage_peak_counts.osc.txt.gz" | md5sum -c
```

```
## hg19.cage_peak_counts.osc.txt.gz: OK
```

### Data loading and preparation in R

The table is in [Order-Switchable Columns][OSC] format.  The `read.table`
command in R will automatically discard its comment lines, set up column names
with the `header=TRUE` option and row names with the `row.names=1` option.

[OSC]: http://sourceforge.net/projects/osctf/

The table is large: so loading the table will take time…


```r
osc <- read.table('hg19.cage_peak_counts.osc.txt.gz', row.names=1, header=TRUE)
dim(osc)
```

```
## [1] 184828    889
```

The name of the libraries are long because they contain a plain English
description of their contents.  We will shorten them to their identifier.  For
example, `counts.Adipocyte%20-%20breast%2c%20donor1.CNhs11051.11376-118A8`
becomes `CNhs11051`.  The association can be re-made using [FANTOM5 SDRF
files](../FANTOM5_SDRF_files/).


```r
colnames(osc) <- regmatches(colnames(osc), regexpr('CNhs.....', colnames(osc)))
```

The first line, `01STAT:MAPPED`, is special and contains the total number of
tags for each library.  The sum of tags in each peak is lower than this,
because some tags are not in peaks.  We will change the value of
`01STAT:MAPPED` so that the sum of the columns in our table is the total number
of reads for each library.


```r
summary(t(osc["01STAT:MAPPED",]))
```

```
##  01STAT:MAPPED     
##  Min.   :  515670  
##  1st Qu.: 2514503  
##  Median : 4275027  
##  Mean   : 4803336  
##  3rd Qu.: 6532804  
##  Max.   :16059002
```

```r
osc['01STAT:MAPPED',] <- osc['01STAT:MAPPED',] - colSums(osc[-grep('01STAT:MAPPED', rownames(osc)),])
summary(colSums(osc), digits=10)
```

```
##     Min.  1st Qu.   Median     Mean  3rd Qu.     Max. 
##   515670  2514503  4275027  4803336  6532804 16059002
```

### Number of peaks detected and total number of tags, part 1

Let's see now how many CAGE peaks are detected per library.  The command `osc >
0` produce a data frame containing `TRUE` where a tag count was higher than zero,
and `FALSE` otherwise.  In `R`, since `TRUE` equals 1 and `FALSE` equals 0, the
`colSums` command will then count the detected peaks.


```r
summary(colSums(osc > 0), digits=10)
```

```
##    Min. 1st Qu.  Median    Mean 3rd Qu.    Max. 
##   25325   47212   55771   55880   63096  114665
```

There are large variations in the number of peaks detected.  Let's take the example
of subcutaneous adipocytes (libraries [CNhs12494][], [CNhs11371][] and [CNhs12017][]).

[CNhs12494]: http://fantom.gsc.riken.jp/5/sstar/FF:11259-116F8 "Adipocyte - subcutaneous, donor1"
[CNhs11371]: http://fantom.gsc.riken.jp/5/sstar/FF:11336-117F4 "Adipocyte - subcutaneous, donor2"
[CNhs12017]: http://fantom.gsc.riken.jp/5/sstar/FF:11408-118E4 "Adipocyte - subcutaneous, donor3"


```r
numberOfPeaks <- colSums(osc > 0)
numberOfTags <- colSums(osc)
adipocytes <- c('CNhs12494', 'CNhs11371', 'CNhs12017')
numberOfPeaks[adipocytes]
```

```
## CNhs12494 CNhs11371 CNhs12017 
##     26353     38773     50976
```

```r
numberOfTags[adipocytes]
```

```
## CNhs12494 CNhs11371 CNhs12017 
##    535250   1384056   4224299
```

Do we see more peaks just because there were more tags ?

### Sub-sampling

Here, we will remove tags from the data until each library has the same number of tags,
that is, we will normalise the _sequencing depth_ on the most _shallow_ library.

We will use the `rrarefy` command of the [vegan](http://vegan.r-forge.r-project.org/) package.
Unlike most R commands that work on data frames, this command uses rows by default, so we will
transpose the table, sub-sample it, and transpose it again.

The sub-sampling removes tags randomly, so each time it is run it will never
produce exactly the same result.  However, highly expressed peaks will stay
highly expressed, etc.  For this tutorial, the computation is made identical
across runs by setting the random number seed with the `set.seed` command (and
resetting it with `rm(.Random.seed)`).  _Note: do not use `set.seed()` in your
project if you do not understand well the consequences._


```r
library(vegan)
```

```
## Loading required package: permute
## Loading required package: lattice
## This is vegan 2.0-10
```

```r
set.seed(1)
osc.sub <- t(rrarefy(t(osc), min(numberOfTags)))
rm(.Random.seed)
summary(colSums(osc.sub), digits=10)
```

```
##    Min. 1st Qu.  Median    Mean 3rd Qu.    Max. 
##  515670  515670  515670  515670  515670  515670
```

That is all !  Now all the libraries contain the same number of tags.

### Number of peaks detected and total number of tags, part 2


```r
colSums(osc.sub > 0)[adipocytes]
```

```
## CNhs12494 CNhs11371 CNhs12017 
##     25955     27756     26558
```

```r
colSums(osc.sub)[adipocytes]
```

```
## CNhs12494 CNhs11371 CNhs12017 
##    515670    515670    515670
```

The number of detected peaks is now very similar between the three biological replicates !

### Second part n construction…

The second part with sub-sampling both dimensions of expression tables will be
added later.