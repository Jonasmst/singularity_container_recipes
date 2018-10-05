# Singularity python container recipes

This repository contains recipes for building singularity containers for various tasks, usually to run software on TSD that is difficult to install due to its being offline.

## python\_container\_with\_rseqc.txt
This is the recipe for a singularity container running Ubuntu 16.04.5 LTS (Xenial Xerus). It enables running RSeQC, a quality control
software for RNA-seq data, on TSD, an offline HPC cluster for sensitive data. However, it includes some other tools as well, like numpy,
pandas and matplotlib. See the recipe for a complete list of tools. Most tools are available in a directory called

`/usr/local/bin`

within the container.

## Running RSeQC from container
To run this container, the Singularity software must be loaded as a module and run on Colossus. To load Singularity, simply include the line `module load singularity` in your SLURM-script. Here's a minimum example running the `bam_stat.py` python script on a BAM-file:
```
#!/bin/bash
# Job name
#SBATCH --job-name=rseqc_example
#
# Project
#SBATCH --account=p19
#
# Wall clock limit
#SBATCH --time=00:30:00
#
# Max memory usage
#SBATCH --mem-per-cpu=1G
#
# Set number of nodes
#SBATCH --nodes=1
#
# Set number of CPUs
#SBATCH --cpus-per-task=1


# Setup job environment
source /cluster/bin/jobsetup

# Purge previously loaded modules (incase they interfere with the current job)
module purge

# Load singularity
module load singularity

# Path to singularity container
container="/cluster/projects/p19/Software/RSeQC/rseqc_container.simg"

# Path to directory containing BAM-file
bam_directory="/cluster/projects/p19/Software/RSeQC/example/input_files"

# Name of BAM-file
bam_filename="1.sorted.bam"

# Path to bam_stat.py within the container
program_path="/usr/local/bin/bam_stat.py"

# Run bam_stat.py in container
# Note: -B maps the $bam_directory on TSD to the /mnt directory within the container file system
singularity run -B $bam_directory:/mnt $container python $program_path -i /mnt/$bam_filename
```

### Handling input and output from outside the container
A singularity container is a self-contained operating system, with its own file-system, separate from the TSD filesystem.
This means that programs running inside the container cannot see the files on the outside, i.e. on TSD. In order to make files accessible,
we need to bind paths between the two file systems. Essentially, this means that you can link one directory on TSD to another directory
within the container. This is controlled by the `-B` flag, which expects two directory paths delimited by `:`.
In our example we're mapping the `$bam_directory` directory in TSD to the `/mnt` directory within the container, so that every file
residing within the `$bam_directory` is accessible from the `/mnt` directory in the container. This is why we're providing
`/mnt/$bam_filename`, instead of `$bam_container/$bam_filename` as the input file path for our python script. 
It is possible to map multiple directories, for instance a separate output directory, but it's probably easier to just
write output data to the `/mnt/` directory within the container.
