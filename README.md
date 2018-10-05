# Singularity container recipes

This repository contains recipes for building singularity containers for various tasks, usually to run software on TSD that is difficult to install due to its being offline.

## Container descriptions
See subdirectories for the respective containers' descriptions.

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
