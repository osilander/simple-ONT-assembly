# simple-ONT-assembly
Simple instructions for long read Oxford Nanopore assembly with some extras

## Package Management
We use a package manager to take care of dependencies. For speed we use `mamba` not `conda`. Note that the `mamba` [installation instructions](https://mamba.readthedocs.io/en/latest/installation.html) say to install directly from mambaforge. Check for the install package on your OS, or the link [here](https://github.com/conda-forge/miniforge/releases/latest/download/Mambaforge-Linux-x86_64.sh) (right-click and copy link) provides the download.

To download and install the package, move to the terminal and use `wget`, and then run using `bash`. This will take you through the installation, in which you agree to the T&C and direct it to the installation location (default is usually fine):

```bash
wget https://github.com/conda-forge/miniforge/releases/latest/download/Mambaforge-Linux-x86_64.sh
bash Mambaforge-Linux-x86_64.sh

# remove once installed
rm Mambaforge-Linux-x86_64.sh
```
You will likely have to restart your terminal window after this.

## Required Tools
For read QC, trimming, visualisation, assembly, and annotation, we require a few tools. We can use `mamba` to install them. The syntax is realitvely simple. In this case we will install all into our base environment.

For general sequence handling and characterisation we will use [seqkit](https://bioinf.shenwei.me/seqkit/usage/).<br>
For read visualisation we will use [NanoPlot](https://github.com/wdecoster/NanoPlot).<br>
For read QC and trimming we will use [chopper](https://github.com/wdecoster/chopper).<br>
For assembly we will use [raven](https://github.com/lbcb-sci/raven). [Flye](https://github.com/fenderglass/Flye) can also be used.<br>
For annotation we will use [prokka](https://github.com/tseemann/prokka). [Bakta](https://github.com/oschwengers/bakta) can also be used.
For assembly visualisation we can use [Bandage](https://rrwick.github.io/Bandage/). I will not cover that here.

```bash
# we will install all of these one-by-one to make sure it goes smoothly
mamba install -c bioconda seqkit

mamba install -c bioconda nanoplot

mamba install -c bioconda chopper

# note that raven is "raven-assembler"
mamba install -c bioconda raven-assembler

mamba install -c bioconda prokka
```

### Quick data summary
The first step is a quick examination of the data. Here I will assume you have a single `fastq` file containing all the reads for your strain. This file will be `reads.fastq`.

```bash
# here the -a option gets All the statistics
seqkit stats -a reads.fastq
```
The output in this case should be a table with statistics such as number of reads, average length, total length, etc. Pay attention to the total length (total number of basepairs) and the N50. The total bp should be ~50-60X higher than your genome length. For example, if your estimated genome length is 5Mbp, then you should have 250Mbp of data. Your N50 should be above 6Kbp, and ideally above 10Kbp. Anything beyond that is great but usually not needed except for highly repetitive genomes.<br>
If your total bp is less than 20X your genome size, the assembly is unlikely to be successful. If your N50 is less than 4Kbp (esp. before filtering) then your assembly is likely to be unsuccessful. You shouuld consider collecting additional data.

### Data visualisation
The next 