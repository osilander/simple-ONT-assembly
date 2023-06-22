# simple-ONT-assembly
Simple instructions for long read Oxford Nanopore assembly with some extras

## Package Management
We use a package manager to take care of dependencies. For speed we use `mamba` not `conda`. Note that the `mamba` [installation instructions](https://mamba.readthedocs.io/en/latest/installation.html) say to install directly from mambaforge. Check for the install package on your OS, or the link [here](https://github.com/conda-forge/miniforge/releases/latest/download/Mambaforge-Linux-x86_64.sh) (right-click and copy link) provides the Linux download.

To download and install the package, move to the terminal and use `wget`, and then run using `bash`. This will take you through the installation, in which you agree to the T&C and direct it to the installation location (default is usually fine):

```bash
wget https://github.com/conda-forge/miniforge/releases/latest/download/Mambaforge-Linux-x86_64.sh
bash Mambaforge-Linux-x86_64.sh

# remove once installed
rm Mambaforge-Linux-x86_64.sh
```
You will likely have to restart your terminal window after this to get `mamba` working.

## Required Tools
For read QC, trimming, visualisation, assembly, and annotation, we require a few tools. We can use `mamba` to install them. The syntax is realitvely simple. In this case we will install all into our base environment.

For general sequence handling and characterisation we will use [seqkit](https://bioinf.shenwei.me/seqkit/usage/).<br>
For read visualisation we will use [NanoPlot](https://github.com/wdecoster/NanoPlot).<br>
For read QC and trimming we will use [chopper](https://github.com/wdecoster/chopper).<br>
For optional read mapping we will use [bwa mem](https://github.com/lh3/bwa).<br>
For assembly we will use [raven](https://github.com/lbcb-sci/raven) although [flye](https://github.com/fenderglass/Flye) can also be used.<br>
For polishing we will use [medaka](https://github.com/nanoporetech/medaka).<br>
For annotation we will use [prokka](https://github.com/tseemann/prokka) although [bakta](https://github.com/oschwengers/bakta) can also be used.<br>
For assembly visualisation we could use [Bandage](https://rrwick.github.io/Bandage/). I will not cover that here.

```bash
# we will install all of these one-by-one to make sure it goes smoothly
mamba install -c bioconda seqkit

mamba install -c bioconda nanoplot

mamba install -c bioconda chopper

mamba install -c bioconda bwa

# note that raven is "raven-assembler"
mamba install -c bioconda raven-assembler

# for medaka I recommend a new environment
mamba create -n medaka -c conda-forge -c bioconda medaka

mamba install -c bioconda prokka

# this is an extra program to visualise directory structure
mamba install -c conda-forge tree
```

## General notes
Several of the steps below may take a few minutes. For any step that takes longer than 2-3 minutes I recommend `tmux`, a terminal multiplexer that will esnure your process deosn't stop once you have disconnected from the server.

#### Organisation
You should endeavour to keep your data organised. In general, you should have the original data stored elsewhere and always keep it untouched. You should operate only on copied data. The data that you are using in the steps below should exist in a personal directory (either on the server or on your computer). This directory should indicate the title or subject of your project, e.g. `soil-isolate-assemblies`. It is best in my experience to put the sequence data files in a directory caleld `data`. Any results from the analyses below you should store in a directory called `results`. To visualise your directory structure, I recommend [tree](https://www.tecmint.com/linux-tree-command-examples/) (above).

```bash
# the structure should look like this
# this visualise 3 *L*evels down
tree -L 3
```

```bash
# output from tree looks like this
├── soil-isolate-assemblies
    ├── data
    │   └── reads.fastq
    └── results
```

I will not go through here in detail into file naming conventions, etc. but it is something to study beforehand. [Here](https://johndecember.com/unix/tutor/filenames.html) is an example.

#### Code Editing and Terminal Use
For editing code and dealing with the terminal I recomend [`VSCode`](https://code.visualstudio.com/) although there are many other options out there.

#### Pipeline Management
To manage and automate the whole process I recommend [`snakemake`](https://snakemake.readthedocs.io/en/stable/) or `Nextflow`. I will not cover any of those details here.

## Important Note
If you have not basecalled your data using either the high accuracy "hac" or super high accuracy "sup" basecalling methods, then you _must_ rebasecall. "fast" basecalling is not accurate enough for good assemblies.

## Assembly Process

### Quick data summary
The first step is a quick examination of the data. Here I will assume you have a single `fastq` file containing all the reads for your strain. This file will be `reads.fastq`.

```bash
# here the -a option gets *a*ll the statistics
seqkit stats -a reads.fastq
```
The output in this case should be a table with statistics such as number of reads, average length, total length, etc. Pay attention to the total length (total number of basepairs) and the N50. The total bp should be ~50 - 60X higher than your genome length. For example, if your estimated genome length is 5Mbp, then you should have 250Mbp of data. Your N50 should be above 6Kbp, and ideally above 10Kbp. Anything beyond that is great but usually not needed except for highly repetitive genomes.<br>
If your total bp is less than 20X your genome size, the assembly is unlikely to be successful. If your N50 is less than 4Kbp (esp. before filtering) then your assembly is likely to be unsuccessful. You shouuld consider collecting additional data.

### Data visualisation
The next step will be some simple visualisation. Here we will use NanoPlot. The syntax here is relatively simple. We will again assume you only have a single `.fastq`.

```bash
# -t specifies the number of threads
NanoPlot -t 2 --fastq reads.fastq --maxlength 40000 --plots dot kde -o qc-plots
```

Take a look at the output (in the qc-plots directory), specifically the `html` report. You should see summary statistics like those with `seqkit` but also some plots. An important plot is the bottom-most plot which should show length and quality. You need to make sure you have some long and high-quality reads.

### Read QC
The next step is quality control of the reads. We need to make sure that most of the reads are high quality (for ONT data) and not short. This is done using `chopper`. Here, we will also specify a maximum length, as reads longer than that are often artifacts. Note that to determine the cutoffs here you should examine the Nanoplot output. I often do a head crop and tail crop (cut of the start and end of the reads) for safety's sake (e.g. if there are adaptors or greater inaccuracy in these areas.)

```bash
# note that you have to "feed" your data to chopper using cat
# the -q options specifies the average read quality
# the -l option specifies the minimum length
cat reads.fastq | chopper -q 10 -l 1000 --maxlength 100000 --headcrop 50 --tailcrop 50 > trim.reads.fastq
```
### Contamination
It is possible that there is lambda phage control DNA in your sample, although in most cases this will not be true consult the individual(s) who did the sequencing. If there is, one option is to map out your reads against the genome of lambda and take only those reads that don't map. You can also map out against other common contaminants, such as the human genome, or common bacteria or phages that are used in your lab setting. For such mapping I recommend `bwa mem`.<br>
However, a far easier approach is the `--contam` paramter in `chopper`, used as follows:

```bash
# ref.fasta is the reference, for example lambda or human
# this can be combined with the step above.
cat reads.fastq | chopper --contam ref.fasta > uncontam.reads.fastq
```

### Assembly
For assembly we will use `raven`. This is a great and fast assembler. On a laptop it might take ten minutes. On a reasonable server-scale computer (e.g. 40 threads, 200GB RAM), it should take less than two minutes. Be very careful, the assembly will output directly to standard out (the terminal window), so we have to redirect the output to a file. The default number of threads is quite small, so here we give it 16. Note the redirect arrow `>` and the output to a `.fasta` file. Note that in the syntax below I have returned to the `reads.fastq` naming rather than `uncontam.reads.fastq`.

```bash
raven -t 16 trim.reads.fastq > assembly.fasta
```

### Polishing
I recommend polishing using Oxford Nanopore's own tool, `medaka`. I have not benchmarked the polishing steps very well for any new ONT data (10.4.1). Note that above we have installed `medaka` in its own environment.<br>
Unfortunately we will also have to select a model. Assuming you have recent flow cells, chemistry, and basecalling, this is _very_ likely to be:<br>
`r104_e81_sup_variant_g610` for super high accuracy "sup" basecalling.<br>
`r104_e81_hac_variant_g610` for high accuracy "hac" basecalling.<br>

```bash
# first activate
mamba activate medaka

medaka_consensus -i trim.reads.fastq -d assembly.fasta -o polished-assembly -t 4 -m r104_e81_sup_variant_g610

# deactivate to move back into base env
mamba deactivate
```

Note that this will output the polished assembly into the named directory, but the polished verion will be `consensus.fasta`. You should rename this file to something sensible, perhaps `polished.assembly.fasta`.

### Assembly summary
For a quick summary of the assembly we will use `seqkit` again. This is relatively simple as it is the same syntax as previously.

```bash
seqkit stats -a polished.assembly.fasta
```

With luck, the summary will indicate a single (or small number) of contigs. Note that `raven` _does not_ output a graph file, so the assembly cannot be visualised with a program like `bandage`. If you want a graph file, I recommend `flye`.

### Annotation
The last step here is annotation. Here we will use prokka, which is relatively fast and simple to use. As mentioned above, `bakta` is also a possibility, and may even be faster. The result of `prokka` will be an output directory with approximately ten files. Perhaps the most important is the Genbank `.gbk` file, which is compatible with a number of other pieces of software.

```bash
# --cpus is the number of threads
# --outdir is the output directory
prokka --cpus 16 --outdir mystrain polished.assembly.fasta
```

### Assembly quality control
I will not cover methods for doing this now. There are three recommendations assuming you only have long reads. First, check the number and orientation of the rRNA operons. This is only possible for realtively common and well characterised bacterial species, and can be done using [`socru`](https://github.com/quadram-institute-bioscience/socru).<br>
Second, check the length of your open reading frames. This is best done with [`ideel`](https://github.com/phiweger/ideel).<br>
Finally, check the split mappings (supplementary malignment) of your original reads on your assembly. If there are locations at which there are many split mappings, it is very likely this is an assembly error. This would usually be done using the mapper above, [`bwa mem`](https://github.com/lh3/bwa) and then some data manipulation to find regions with lots of supplementary reads.<br>
There are also packages to do assembly QC. I am not well acquainted with them, and many require short reads. [`QUAST`](https://github.com/ablab/quast) is not very useful in this case, as the assemblies we are usually looking at are complete and single contig. [`ALE`](https://bioinformaticshome.com/tools/wga/descriptions/ALE-Assembly.html#gsc.tab=0) works better with short reads as far as I know

### Conclusions
I think this is - approximately - best practices for now. These instructions assume substantial familiarity with the command line and some ability to trubleshoot issues that arise. The primary method for troubleshooting these is to _look very carefully_ for the specific error message and Google it, or paste the message into ChatGPT and see if it can offer help.