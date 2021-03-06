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

```
# Save plot to the /mnt/ directory
pdf("/mnt/MA_plot.pdf")
plotMA(some_variable)
dev.off()
```
The `MA_plot.pdf` can now be found in the `my_directory` directory.

The shared directory can also be used to provide input files from the host system to the container, e.g. R-scripts that are to be
run within the container. Simply copy your scripts to the `my_directory` directory and access them from the `/mnt` directory:

`singularity exec -B my_directory:/mnt/ r_container.simg R --vanilla < /mnt/r_script.R`

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
singularity exec -B $shared_directory:/mnt $container R --vanilla < my_r_script.R
```
Here, `my_r_script.R` is located within the directory from which the job is submitted, i.e. in the `$shared_directory` directory.
The very last line of the SLURM-script runs the container and asks it to execute the R-script. The R-script could be anything, e.g.
a simple hello-world program, or the entire DESeq2 pipeline.

### Some observations related to mapping.
#### When specifying the input file to be run in the SLURM-file, we don't need to use mapped paths.
For instance, in the script above, we're executing `my_r_script.R`, which is the path to the script outside of the container. If we try
to read it from the mapped directory inside the container, i.e. `/mnt/`, we get an error saying there's no such file. E.g., we can't do

`singularity exec -B shared_directory/:/mnt/ r_container.simg R --vanilla < /mnt/my_r_script.R`

we must do

`singularity exec -B shared_directory/:/mnt/ r_container.simg R --vanilla < shared_directory/my_r_script.R`

#### Input and output files handled within the container, must reside in the mapped directory.
When reading input files or writing output files from processes within the container, e.g. in the `my_r_script.R`, script
(which is run inside the container), we need to use the path of the mapped directory (i.e. `/mnt/`). E.g. inside `my_r_script.R`,
we can't do

```
outfile <- file("shared_directory/output.txt")
writeLines(c("Hello", "World"), outfile)
close(outfile)
```

we must do

```
outfile <- file("/mnt/output.txt")
writeLines(c("Hello", "World"), outfile)
close(outfile)
```

#### Conclusion
It seems that arguments provided in the SLURM-script when executing the container (e.g. the `my_r_script.R` script) is interpreted in the scope
outside of the container, i.e. in the host file system, while everything within `my_r_script.R` is done in the scope inside the container. This
comes with the following caveat:

If you specify paths as input arguments to your R-script, e.g.:

`singularity exec -B shared_directory/:/mnt/ r_container.simg R --vanilla < shared_directory/my_r_script.R --args output_file.txt`

the `output_file.txt` path must be prepended with a path within the scope of the container. E.g.:

`singularity exec -B shared_directory/:/mnt/ r_container.simg R --vanilla < shared_directory/my_r_script.R --args /mnt/output_file.txt`

Specifying a path within the scope of the host system won't work. E.g.:

`singularity exec -B shared_directory/:/mnt/ r_container.simg R --vanilla < shared_directory/my_r_script.R --args shared_directory/output_file.txt`

and will give you an error saying that there's no such file or directory. The reason is that when the `my_r_script.R` reads the input argument
(the path to `output_file.txt`), the script is within the container, and can only relate to paths within the container's scope.

The same applies with input files. If you're passing the path to e.g. a BAM file to the R-script, the paths must be prepended by the `/mnt/` path,
and will then be able to read files from the `shared_directory/` directory.

Hope this makes sense.

