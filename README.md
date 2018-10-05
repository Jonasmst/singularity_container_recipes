# Singularity container recipes

This repository contains recipes for building singularity containers for various tasks, usually to run software on TSD that is difficult to install due to its being offline.

## Container descriptions
### [python\_container\_with\_rseqc.txt](https://github.com/Jonasmst/singularity_container_recipes/blob/master/python_container_with_rseqc.txt)
This container contains a python2 runtime with RSeQC and all its dependencies, in addition to some frequently used libraries in data science, such as numpy, pandas and matplotlib. Check out the recipe for a complete list of packages.

### [r\_container\_with\_deseq2.txt](https://github.com/Jonasmst/singularity_container_recipes/blob/master/r_container_with_deseq2.txt)
This container contains an R 3.4.4 runtime with DESeq2 and all its dependencies. Check out the recipe for a complete list of packages installed.

_Note_: You may have to set the R library-path to avoid errors with R trying to read packages from the host file system. Use something like:

`singularity exec r_container.simg find / -iname "*deseq2*"`

to find where R-packages were installed inside the container. Then set the library-path to this directory in the beginning of R-scripts run within the container by:

`.libPaths("/usr/local/lib/R/site-library")`

Now, R should look for packages in `/usr/local/lib/R/site-library`, rather than whatever is the default path in the host system.

## Building containers
Building singularity containers requires 
1. A working singularity installation. I use a vagrant VM on macOS.
2. Sudo privileges.

Building is done with the `build` command, and requires a name of the container image you're building, and the path to a recipe. E.g.:

`sudo singularity build r_container.simg r_container_with_deseq2.txt`

It's also a good idea to reroute output messages from stdout to a log file. This makes it easier to find errors in the quite long stream of output, and can be done by piping the output to a file using `tee`:

`sudo singularity build r_container.simg r_container_with_deseq2.txt | tee build_log.txt`

## Running containers
To check if the built container works, e.g. that your R-packages have been installed correctly, you can run a shell in the container, open an R session and load the package:

`singularity shell r_container.simg`

then

`R`

and finally

`library(DESeq2)`

A successfull run should look something like this:

```
vagrant@vagrant:~/r_container$ singularity shell r_container.simg
Singularity: Invoking an interactive shell within container...

Singularity r_container.simg:~/r_container> R

R version 3.4.4 (2018-03-15) -- "Someone to Lean On"
Copyright (C) 2018 The R Foundation for Statistical Computing
Platform: x86_64-pc-linux-gnu (64-bit)

R is free software and comes with ABSOLUTELY NO WARRANTY.
You are welcome to redistribute it under certain conditions.
Type 'license()' or 'licence()' for distribution details.

  Natural language support but running in an English locale

R is a collaborative project with many contributors.
Type 'contributors()' for more information and
'citation()' on how to cite R or R packages in publications.

Type 'demo()' for some demos, 'help()' for on-line help, or
'help.start()' for an HTML browser interface to help.
Type 'q()' to quit R.

During startup - Warning message:
Setting LC_CTYPE failed, using "C"
> library(DESeq2)
Loading required package: S4Vectors
Loading required package: stats4
Loading required package: BiocGenerics
Loading required package: parallel

Attaching package: 'BiocGenerics'

The following objects are masked from 'package:parallel':

    clusterApply, clusterApplyLB, clusterCall, clusterEvalQ,
    clusterExport, clusterMap, parApply, parCapply, parLapply,
    parLapplyLB, parRapply, parSapply, parSapplyLB

The following objects are masked from 'package:stats':

    IQR, mad, sd, var, xtabs

The following objects are masked from 'package:base':

    anyDuplicated, append, as.data.frame, cbind, colMeans, colnames,
    colSums, do.call, duplicated, eval, evalq, Filter, Find, get, grep,
    grepl, intersect, is.unsorted, lapply, lengths, Map, mapply, match,
    mget, order, paste, pmax, pmax.int, pmin, pmin.int, Position, rank,
    rbind, Reduce, rowMeans, rownames, rowSums, sapply, setdiff, sort,
    table, tapply, union, unique, unsplit, which, which.max, which.min


Attaching package: 'S4Vectors'

The following object is masked from 'package:base':

    expand.grid

Loading required package: IRanges
Loading required package: GenomicRanges
Loading required package: GenomeInfoDb
Loading required package: SummarizedExperiment
Loading required package: Biobase
Welcome to Bioconductor

    Vignettes contain introductory material; view with
    'browseVignettes()'. To cite Bioconductor, see
    'citation("Biobase")', and for packages 'citation("pkgname")'.

Loading required package: DelayedArray
Loading required package: matrixStats

Attaching package: 'matrixStats'

The following objects are masked from 'package:Biobase':

    anyMissing, rowMedians


Attaching package: 'DelayedArray'

The following objects are masked from 'package:matrixStats':

    colMaxs, colMins, colRanges, rowMaxs, rowMins, rowRanges

The following object is masked from 'package:base':

    apply

>
```

