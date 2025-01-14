cnv-methyl manual: dynamic Somatic Copy Nunmber Alterations for
methylation array data.
================
Izar de Villasante
09 January 2023

<!-- README.md is generated from README.Rmd. Please edit that file -->

# Introduction

The `cnv.methyl`package implements an automatic and dynamic thresholding
technique for **c**opy **n**umver **v**ariation analysis using Illumina
450k or EPIC DNA **methyl**ation arrays. It enhances the whole pipline
of methylation array processing from raw data to cnv calls. The main
reasons to use it are:

1.  It is fast. Runs in parallel and it is prepared to be run on HPC
    environments seamlessly.
2.  It is dynamic. It provides cnv calls and copy number values for your
    genes of interest based on each array metrics (purity, mean, sd).
3.  It is flexible. Since it accepts both **450k** and **EPIC**
    methylation arrays, different controls and genome annotations and
    although it relies on `conumee` package to calculate segmentations
    and generate log2r intensities, it can also accept input from any
    other tools.

## Installation

You can install the development version of cnv.methyl from
[GitHub](https://github.com/) with:

``` r
# install.packages("devtools")
devtools::install_github("https://github.com/ijcBIT/cnv.methyl.git")
```

## Context

Although the primary purpose of methylation arrays is the detection of
genome-wide DNA methylation levels \[@bibikovahigh2011\], it can
additionally be used to extract useful information about copy-number
alterations, e.g. in clinical cancer samples. The approach was initially
described in Sturm et al., 2012 \[@sturmhotspot2012\]. Some tools have
been developed for this purpose, such as `conumee`, `ChAMP` and
`CopyNumber450k` \[@conumee;@champ;@cnv450k\].

Nevertheless, all this tools require a certain level of human
interaction and interpretation in order to obtain meaningful information
from the data. Some of these tasks are automatically resolved by the
package such as providing the right annotation for the genes of interest
or setting a threshold for each cna.

Setting the threshold, may be the most decisive and challenging step. So
far, the main approach is to visually inspect the log2ratios plot in
order to get some insight of the genomic alterations. Nevertheless this
threshold may vary between arrays depending on purity of the samples and
noise. SNPs based arrays tools, such as `ASCAT` or `PURPLE`, are aware
of this problems and correct it in order to provide a copy number value.
Nevertheless, this level of precision had not been yet accomplished by
the available tools for methylation arrays. Until now.

This CNV analysis pipline can be broken into 3 main blocks:
pre-processing, segmentation & cnv calling:

# Pre-processing:

This pipeline uses an enhanced parallel version of `read.metharray.exp`
function from `minfi` package @minfi in order to load the idats and a
precalculated matrix from `RFpurify` @RFpurify for imputing the
purities.

The minimum requirements to run this pipline are the sample sheet and
its corresponding idat files. Both of them can be found within the
package as example data. Let’s have a look:

``` r
sample_sheet<-data.table::fread(system.file("extdata", "Sample_sheet_example.csv",package="cnv.methyl"))
str(sample_sheet)
```

This format has to be respected in order to make everything work fine.
`Sample_Name` contains the sample ids and `filenames` contain their
current path. The other parameters are required by minfi in order to
read the arrays. Basename contains the working directory where minfi
will looks for files and it depends on the folder parameter.

``` r
library(cnv.methyl)
folder="analysis/intermediate/IDATS/"
sample_sheet$Basename<-paste0(folder, basename(sample_sheet$filenames))
myLoad <- pre_process.myLoad(
  targets=sample_sheet,folder=folder,arraytype="450K"
  )
```

``` r
myLoad
```

Once data is loaded tumor purity in the sample is calculated with
`purify()` :

``` r
library(cnv.methyl)
purity<-purify(myLoad=myLoad)
myLoad@colData$Purity_Impute_RFPurify<-purity
```

This function uses the pre-computed matrix of absolute purities
`RFpurify_ABSOLUTE` provided in RFpurify @RFpurify and also performs
imputation of missing values with `impute.knn`.

Then data is normalized with `queryfy`:

``` r
query <- queryfy(myLoad)
```

This function perforrms the following steps:

1.  Normalization by `minfi::reprocessQuantile`

2.  Filtering:

    - `minfi::detectionP()` Remove arrays: Minimum detection threshold
      with p-value \< 0.01 per probe and a maximum of 10% failed probes
      per sample.

    - `minfi::dropLociWithSnps()` Removes snps: probes that were located
      +/- 10 bases away from known SNPs were also filtered out.

    - `maxprobes::dropXreactiveLoci()` Remove cross reactive probes.

You can use `pre_process()` function in order to run all these steps at
once:

``` r
library(cnv.methyl)
sample_sheet<-data.table::fread(system.file("extdata", "Sample_sheet_example.csv",package="cnv.methyl"))
out="analysis/intermediate/"
pre_process(targets = sample_sheet, out = out, RGset = F )
```

``` elixir
[//]: (It is important to know the folder structure. 
out is for working directory, 
subf is the subfolder inside out where minfi reads the idats. 
Folder overwrites this parameters.  )
```

As you can see it is pretty easy to substitute any part of the analysis
to your convenience. The most interesting part here is that all packages
and data needed for the analysis are bundled together and the speedup of
`minfi` loading idats in `parallel` .

# Segmentation

Segmentation is performed using a two-step approach as described in
`conumee`. First, the combined intensity values of both ‘methylated’ and
‘unmethylated’ channel of each CpG are normalized using a set of normal
controls (i.e. with a flat genome not showing any copy-number
alterations).

By default a set of 96 Whole Blood Samples are used unless the user
specifies something else:

``` r
 cnv.methyl:::control
```

Also annotation for **EPIC** , **450K** , or an overlap of both arrays
is also built-in:

``` r
cnv.methyl:::anno_epic
cnv.methyl:::anno_450K
cnv.methyl:::anno_overlap
```

The right annotation will be chosen in each case according to the
arraytype. If not specified it defaults to `anno_overlap`.

If you are using the built-in controls and epic arrays as input you
should change arraytype to overlap. The intensities can be given in the
following formats:

a genomics Ratio set or path to file. Accepted formats are: .fst, .rds,
.txt or other text formats readable by fread.

The output of `pre_process` function above with intensities from 20
samples is used as example dataset:

``` r
# 
# intensity<-readRDS(system.file("extdata", "intensities.RDS",package="cnv.methyl"))
# run_conumee(intensities=intensity,arraytype="450K")
# run_conumee(intensities = intensity,anno_file=anno, ctrl_file=ctrl_file,
#               Sample_Name=Sample_Name,seg.folder = seg.folder,
#               log2r.folder = log2r.folder,arraytype=arraytype,
#               conumee.folder=conumee.folder, probeid=probeid)
```

This step is required for correcting for probe and sample bias (e.g.
caused by GC-content, type I/II differences or technical variability).
Secondly, neighboring probes are combined in a hybrid approach,
resulting in bins of a minimum size and a minimum number of probes. This
step is required to reduce remaining technical variability and enable
meaningful segmentation results.

Plotting is not yet implemented by the pipeline. If you are interested
in this functionality, please make a request.

# cnv calling:

In order to remove biological and technical noise of each sample, the
relationship between log2ratio signal and noise \[tumor purity & signal
standard deviation\] has been calculated for each copy number
alteration, as described in the paper \[Blecua, P et al.\]@Blecua

### Predict Kcn values:

A precalculated constant Kcn is used in order to adjust each cpg site
log2r intensity value in a given array taking into account the purity of
the sample (biological noise) and the overall standard deviation of the
sample (technical noise).

There are currently 2 different sets of Constants available. One
calculated with the cancers proportion described in the paper
(“curated”) & one with balanced proportion of different cancers
(“balanced)

``` r
Kc_get(ID="sample_name",ss=sample_sheet,
       Kc_method="curated")
```

If you have your segments and log2ratio intensity files somewhere
different than the default folder you should also specify where.

The output is a genomic ranges object with 3 extra columns:

- cna: The name of the predicted cn category.
- cn: number of predicted copies for the gene.
- gene.name: USCS gene names for the genes that are found in that
  region.

You can also calculate a new constant with your own pool of reference
samples. These reference samples must be treated in the same way as the
targets with unknown cnv state with the difference that the sample sheet
must contain one column for each of the real copy number state for each
gene. Default values are:

${Amp10,Amp,Gains,Diploid,HetLoss,HomDel}$

This information is later used by the Kc_make() function to calculate
the Kcn constants.

### Generate Kcn values:

Once the log2r ratios from the array are calculated by CONUMEE, and the
Segments file is created, the Kcn for each cn state can be retrieved as
follows:

For each of cn state, we perform a linear regression:

$p * sd(log2[Ra]) ~ 0 + (log2[Rcn] – mean(log2[Ra]))$

$y = 0 + bx , then: B = y/x$

$Kcn = 1/b → x/y$

Therefore:

$Kcn =(log2[Rcn] – mean(log2[Ra])) / p * sd(log2[Ra])$

- mean(log2\[Ra\]) = the log2ratio of the whole array obtained by
  `conumee::CNV.fit()`
- log2\[Rcn\] = the mean log2 value for each gene of interest and cn
  estate.
- p = purity calculated with Rfpurify from methylation arrays using
  `pre_process`
- sd(log2\[Ra\]) = standard deviation of the array.

Cn are the different categories of copy number a segment can have. If
you want to change the defaults just specify these values on the cncols
variable and generate one column in your sample sheet for each of your
predefined categories.

``` r
Kc_make(ss=ss_train,ID=ss_train$Sample_Name,cncols=c("Amp10","Amp","Gains","HetLoss","HomDel"))
```

For more details about the package’s theorical basis please refer to the
paper from [Blecua, P. et
al](https://app.dimensions.ai/details/publication/pub.1147118663)

When using `cnv.methyl` in your work, please cite as:

``` r
citation("cnv.methyl")
```
