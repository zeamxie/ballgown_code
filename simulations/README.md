## code for differential expression simulations

This code executes the simulations presented in the [ballgown manuscript](http://biorxiv.org/content/early/2014/09/05/003665.full-text.pdf+html). Specifically, `sim_results.R` produces these results:

* Supplementary Figure 6 (all panels)
* Supplementary Figure 7a-b

### two simulations
Two separate simulation scenarios are presented in the manuscript. 

1. The first scenario (described in the main manuscript) involves setting each transcript's expression level using FPKM, and accordingly, adding differential expression at the FPKM level. In other words, transcripts differentially expressed at 6x fold change had an average FPKM that was 6x greater in the overexpressed group. The number of reads to generate from each transcript was then calculated by multiplying the FPKM by the transcript length (in kilobases) and by a library size factor (in millions of reads). Relevant code is in the `FPKM` folder.

2. The second scenario (described in the manuscript supplement) involves drawing the number of reads to generate from each transcript from a negative binomial distribution. The transcript's length has no effect on the number of reads simulated from it. Scripts for this scenario are in the `NB` folder.

Specific details about parameters chosen for these simulations are available in the manuscript supplement (Supplementary Note 5).

### how to use this code

#### (0) get dependencies
This code depends on R and the Rscript command line utility, the Biostrings, Ballgown, and Polyester R packages, and Python >=2.5. I also used my custom `usefulstuff` R package, available from GitHub. We ran all code on Linux.

To download Biostrings, Ballgown, and Polyester: in R, run:
```S
source("http://bioconductor.org/biocLite.R")
biocLite("Biostrings")
biocLite("ballgown")
biocLite("polyester")
```

You'll need devtools to install usefulstuff from GitHub:
```R
install.packages('devtools') #if needed
devtools::install_github('alyssafrazee/usefulstuff')
```

Additionally, we relied heavily on the Sun Grid Engine (SGE) scheduling system when running this pipeline, since this is what our department uses to schedule batch cluster jobs. In particular, the shell scripts in this folder contain `qsub` commands, indicating that a script is being submitted to the cluster to be run, so these lines will have to be modified if you want to run this code without using SGE. 

Finally, you will need [TopHat](http://tophat.cbcb.umd.edu/) (for the paper, we used version 2.0.11), [Cufflinks](http://cufflinks.cbcb.umd.edu/manual.html) (we used version 2.2.1), and [Tablemaker](https://github.com/alyssafrazee/tablemaker). These scripts assume that the executables "tophat", "cufflinks", "cuffmerge", "cuffdiff", and "tablemaker" are in your path. 

#### (1) get transcripts to simulate from
We simulated reads from human chromosome 22, Ensembl version 74. All transcripts can be downloaded from `ftp://ftp.ensembl.org/pub/release-74/fasta/homo_sapiens/cdna/Homo_sapiens.GRCh37.74.cdna.all.fa.gz`. Download this file, un-tar, and un-zip it, then run `get_chr22.R` to subset to chromsome 22. This produces `ensembl_chr22.fa`.

#### (2) get annotation files
We used Illumina's iGenomes annotation files, available at [this link](http://tophat.cbcb.umd.edu/igenomes.shtml). Specifically, we used the Ensembl annotation (first link on the page). The `ANNOTATIONPATH` variable in the shell scripts points to the folder containing the `Homo_Sapiens` directory that comes with the iGenomes index download. This is used in several places in these scripts.

To create the `genes-clean.gtf` file (used in `sim_results.R`), the `genes.gtf` file (located in the `Annotation/Genes` subfolder) was processed with the `clean_genes.R` script. This `genes-clean.gtf` file contains only chromosomes 1-22, X, and Y (the clean_genes script removes all others). 

`genes-clean.gtf` is available [here](https://www.dropbox.com/s/89iaagrkwlu0tbs/genes-clean.gtf). You should put it in the top-level directory (i.e., in *this* directory).


#### (3) pre-build a transcriptome for TopHat
During TopHat runs in the simulations, we aligned first to the transcriptome, then to the genome (i.e., we used TopHat's `-G` option). We pre-built a transcriptome index and used that build for all TopHat runs to avoid re-building every time. A script plus small dummy reads used to build the transcriptome index are in the `tophat_transcriptome` subfolder. The `ANNOTATIONPATH` environment variable should be the same as it is in the shell scripts in the main (`simulations`) directory.

#### (4) run the shell script starting with `run_sim`
i.e., run `run_sim_directFPKM_geuvadis.sh` or `run_sim_p00.sh`. 

You might need to edit some environment variables at the beginning of these scripts:  
* `$SOFTWAREPATH` should contain a folder called `cufflinks-2.1.1.Linux_x86_64` (containing `cufflinks`, `cuffmerge`, and `cuffdiff`)
* `$SOFTWAREPATH` should also contain the `tablemaker` binary
* `$ANNOTATIONPATH` should contain a folder called `Homo_Sapiens`, which comes with the Ensembl [iGenomes download](http://tophat.cbcb.umd.edu/igenomes.shtml)
* `$MAINDIR` only exists to reference `$FOLDERNAME`. All output from this pipeline will be written to `$FOLDERNAME`. **Make sure `$FOLDERNAME` is empty when you begin running the script.**
* `$PYTHON` should point to your python executable
* `$Q` is the SGE queue to run the jobs on. Also note that line 2 of the script, the one beginning with `#$`, contains arguments to pass to SGE. Change these as needed for your system.
* in `run_sim_directFPKM_geuvadis.sh`, `$GEUVADISBG` should point to the GEUVADIS ballgown object (`geuvadisbg.rda`), which you can download [here](https://www.dropbox.com/s/kp5th9hgkq8ckom/geuvadisbg.rda) or create yourself (see the "GEUVADIS_preprocessing" folder in this repo).


These scripts:  
* simulate RNA-seq reads with the code in `FPKM` (scenario #1) and `NB` (scenario #2) folders. The shell script should be directly run (it calls the R script)
* run TopHat on each simulated sample
* Assemble transcripts with Cufflinks for each sample
* merge sample-specific assemblies with Cuffmerge
* run Tablemaker (preprocessing step for Ballgown) on each sample
* run Cuffdiff on the experiment

The python scripts in this repo are also called by the shell scripts. All the python scripts do is look for output: they check to make sure all the TopHat jobs are done before moving on to Cufflinks, check to make sure all the Cufflinks jobs are done before doing Cuffmerge, and check to make sure all the tablemaker jobs have finished before re-organizing output files and moving on to the analysis step.

#### (5) analyze output
All output will be organized in the folder specified in the `FOLDERNAME` variable at the beginning of the shell scripts (default is `./FPKM_out` and `./NB_out`). TopHat output will be in the `alignments` subfolder, Cufflinks in the `assemblies` folder, Cuffdiff in the `cuffdiff` folder, etc. Ballgown `.ctab` files will be in subfolders of the `ballgown` directory, so a ballgown object can be created in R as follows:

```S
# assuming current working directory is FPKM or NB:
library(ballgown)
bgresults = ballgown(dataDir='ballgown', samplePattern='sample')
```

Running `sim_results.R` gives all the figures/results for the manuscript.
 
