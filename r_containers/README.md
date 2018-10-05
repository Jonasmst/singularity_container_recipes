# Singularity R container recipes

This repository contains recipes for building singularity containers for various tasks, usually to run software on TSD that is difficult to install due to its being offline.
This directory contains recipes for the R programming language.

## r\_container\_with\_deseq2.txt 
This container contains an R 3.4.4 runtime with DESeq2 and all its dependencies. Check out the recipe for a complete list of packages installed.

### Setting library paths to avoid errors when loading R packages.
_Note_: You may have to set the R library-path to avoid errors with R trying to read packages from the host file system. Use something like:

`singularity exec r_container.simg find / -iname "*deseq2*"`

to find where R-packages were installed inside the container. Then set the library-path to this directory in the beginning of R-scripts run within the container by:

`.libPaths("/usr/local/lib/R/site-library")`

Now, R should look for packages in `/usr/local/lib/R/site-library`, rather than whatever is the default path in the host system.


### Mapping file systems - Handling input and output files from outside the container
Singularity containers by design do not share a file system with its host system. This means that output files created when running a container stays within the container, inaccessible to the host system from which the container is run.
In order to access files created within the container, we can bind paths from outside the container to the inside of the container, essentially specifying
a directory that links the outside to the inside, similar to how services like Dropbox works, where the same directory can be accessed from two different
computers. We need two paths:
1) Path to the directory in the host system, i.e. "outside" of the container.
2) Path to the directory in the container.

When running the container, we can then save files to the directory specified in `2)`, and later access them from the host system
in the directory specified in `1)`. To do this, we use the `-B` flag when running our container:

`singularity exec -B my_directory:/mnt r_container.simg`

Here, `:` delimits two paths, and `my_directory` is the directory in the host system (`1)`, the one we have access to outside of the container),
and `/mnt` is the directory within the container (`2)`). When we run scripts inside the container, we can write output files to the `/mnt` directory,
and then access these files later in the `my_directory` directory. E.g. saving a plot from an R-script:

The shared directory can also be used to provide input files from the host system to the container, e.g. R-scripts that are to be
run within the container. Simply copy your scripts to the `my_directory` directory and access them from the `/mnt` directory:
`singularity exec -B my_directory:/mnt/ r_container.simg R --vanilla < /mnt/r_script.R`

```
# Save plot to the /mnt/ directory
pdf("/mnt/MA_plot.pdf")
plotMA(some_variable)
dev.off()
```
The `MA_plot.pdf` can now be found in the `my_directory` directory.

### Running the container
On [TSD](https://www.uio.no/english/services/it/research/sensitive-data/index.html), we're not allowed to run singularity outside of SLURM. If you try, you'll see an error message along the lines of:
```
ERROR   : Singularity is not running with appropriate privileges!
ERROR   : Check installation path is not mounted with 'nosuid', and/or consult manual.
ABORT   : Retval = 255
```

So we'll need to create a SLURM-script in which we specify how to run the container, and then submit it as a job to the cluster. Here's an example
of what a SLURM-script could look like:
```
#!/bin/bash
# Job name
#SBATCH --job-name=deseq2
#
# Project
#SBATCH --account=p19
#
# Wall clock limit
#SBATCH --time=01:00:00
#
# Max memory usage
#SBATCH --mem-per-cpu=1G
#
# Set number of nodes
#SBATCH --nodes=1
#
# Set number of CPUs
#SBATCH --cpus-per-task=1
#

## Setup job environment, this is required for all jobs on TSD
source /cluster/bin/jobsetup

## Purge currently loaded modules to avoid conflicts
module purge

## Load singularity
module load singularity

## Path to singularity container
container="/cluster/projects/p19/Projects/CRC-RNA-SEQ/deseq2/r_container.simg"

## Path to shared directory, in this case, we use the directory from which we run the SLURM-script
## so that the container can read input files from this directory, and write output files to this
## directory
shared_directory="/cluster/projects/p19/Projects/CRC-RNA-SEQ/deseq2"

## Run container
echo "Running container"
singularity exec -B $shared_directory:/mnt $container R --vanilla < /mnt/my_r_script.R
```
Here, `my_r_script.R` is located within the directory from which the job is submitted, i.e. in the `$shared_directory` directory. So it is mapped
from outside the container to the inside of the container (as done with the `-B` flag, see above). The very last line of the SLURM-script
runs the container and asks it to execute the R-script. The R-script could be anything, e.g. a simple hello-world program, or the entire DESeq2 pipeline.
