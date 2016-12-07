# Computational methods used in Wu et al.

## Table of Contents
* [EM: Estimating parameters using expectation maximization (EM)](#em-estimating-parameters-using-expectation-maximization-em)
  * [Requirement](#requirement)
  * [Scripts and usages](#scripts-and-usages)
  * [Example](#example)
  * [Note](#note)
* [SNP: Calling heterozygous sites](#snp-calling-heterozygous-sites)
  * [Requirement](#requirement-1)
  * [Scripts and usages](#scripts-and-usages-1)
  * [Example](#example-1)
* [R: R scripts](#r-r-scripts)

## `EM`: Estimating parameters using expectation maximization (EM)
Use expectation maximization (EM) to estimate parameters for
mixture of dirichlet-multinomials (MDM) models from counts-by-ref file generated by the python script.

### Requirement
- [R](https://www.r-project.org/)
- [mc2d: Tools for Two-Dimensional Monte-Carlo Simulations](https://cran.r-project.org/web/packages/mc2d/index.html)

### Scripts and usages
#### `search_em_mdm.R`: EM for trio data
- 3 parameters required.

  ```
  Rscript search_em_mdm.R numTrial numComponent countsByRefFile
  ```
- `numTrail`: The number of attemps to estimate the parameters
- `numComponent`: The number of components in the MDM model. A suffix 'p' or "P" can be appended to the number, which will use an alternative filtering process (See **Note** below).
- `countsByRefFile`: Counts-by-ref file generated from the python script `get_base_counts_trio.py`
- The default read count filter is 10 and 150. This can be change by
  ```R
  upperLimit <- 150
  lowerLimit <- 10
  ```

#### `search_em_mdm_single.R`: EM for single data
- 3 parameters required.

  ```
  Rscript em_mdm_single.R numTrial numComponent countsByRefFileSingle
  ```
- `numTrail`: The number of attemp to estimate the parameters. Same as above
- `numComponent`: The number of components in the MDM model.
- `countsByRefFileSingle`: Counts-by-ref file generated from the python script `get_base_counts_single.py`


#### `mdm.R`: Core functions
- Usually don't use this directly.


### Example
Run EM on 2 components MDM model with alternative filter 5 times on `counts_example_byref.txt`
```bash
Rscript search_em_mdm.R 5 2P counts_example_byref.txt
```
The expected output should be something like the following and you should have different `ll` values. The `max ll` we are able to obtain for the example dataset is `-2593.181911539119`
```
Searching for maximum likelihood of 2-component model....
Using potential heterozygous data with 10x to 150x coverage....
  Trial 1: ll = -2593.377037122174 (max ll = -2593.377037122174)
  Trial 2: ll = -2593.424801477642 (max ll = -2593.377037122174)
  Trial 3: ll = -2593.18827510774 (max ll = -2593.18827510774)
  Trial 4: ll = -2593.199191810304 (max ll = -2593.18827510774)
  Trial 5: ll = -2593.560652839873 (max ll = -2593.18827510774)
Saving results to em_mdm_example_2P_results_10275.RData ...

```


### Note
There are two different filtering processes.
The default filter used both read counts (default values: between 10 and 150) and previous SNPs calls result.
In our case, we are using the results from 1000 Genomes Project.
This is used to generate the True Heterozygote (TH) dataset in our paper.


An alternative filter only filtering on read counts.
This is used to generate  the Potential Heterozygote (PH) dataset in our paper.
When the users specify the number of components, if a suffix "p" or "P" is appended (e.g. 3p or 4P),
then the alternative filter is applied.

<!--
((dat$snp == 1 & dat$snpdif == 0) | dirty_data),
snp found as variable in the 1000 genomes data - what we're using as our standard for true snps
#         self.altalt = altalt        #(1) a different alterate allele was found in the 1000 genomes data
 it's a SNP and dif==0?
 -->






## `SNP`: Calling heterozygous sites
Call heterozygous sites using `Samtools` and `BCFtools`. Filter the results and generate base counts-by-ref files for expectation maximization analyses.

### Requirement
- [Samtools](http://www.htslib.org/) version 1.2 or later
- [BCFtools](http://www.htslib.org/) version 1.2 or later
- [Python 2.7](https://www.python.org/)
- **Recommendation:** [GNU Parallel](https://www.gnu.org/software/parallel/)


### Scripts and usages
#### `pipelineCore.sh`: Main scriipt to run the analyses
- Calls other scripts in the folder
- The following parameters are required.

  ```bash
  datasetName="CEU13_example" #Can be any name
  chromosome=21 #Which chromosome would you like to perforem the analyses on
  chromosomePosition=":9411180-9421180"  #Perfrom analyses on a specific region e.g. chromosomePosition=":10000-20000"
  #chromosomePosition="" #Blank: perform analyses on the whole chromosame.

  workingDir="./data/${datasetName}_C${chromosome}/"  #Location of the output files
  scriptDir="./"  #Location of the scripts

  variableSiteFile="./data/snp_chr21_v3_partial.vcf.gz" #Location of known snp file
  refGenome="./data/human_g1k_v37_c21_partial.fasta" #Location of the reference file
  dataDir="./data/"   #Location of the datafile
  dataFiles=('CEU13_NA12878.bam' 'CEU13_NA12892.bam' 'CEU13_NA12891.bam') #File names of each individual

  parallelCount=3 #Number of parallel jobs
  numSplitFiles=5 #Number of split files
  isTrio=1 #1:trio 0:single caller
  isOriginalCaller=1 #[1|0] # BCFtools call:classic mode -c, --consensus-caller  or new mode -m, --multiallelic-caller
  exome=0 #1: turn on exome analyses
  exome_file=0 #Location of the exome files
  ```


#### `get_base_counts_trio.py`: Python script to parse the output for both trio and single callers
  - 6 parameters required.

  ```bash
  python2 get_base_counts_trio.py chromosome datasetName variableSiteFile workingDir pileupFile.gz pileupExomeFile

  # chromosome: chromasome number (21)
  # datasetName: Any name (CEU13_example)
  # variableSiteFile: Location of known snp file (/data/snp_chr21_v3_partial.vcf.gz)
  # workingDir: Location of the output files (/data/CEU13_example_C21/original/)
  # pileupFile: pileupFile created by Samtools, MUST be is gzip format. (chr21_CEU13_example.pileups.gz)
  # pileupExomeFile: pileupFile for exame. 0 if no exame. (0)
  ```

#### `get_base_counts_single.py`: Python script to parse the output for single caller
 - Same usage as above


#### `checkSnps.sh`: Calling SNPs using trio and single caller
  - Called by `pipelineCore.sh`, usually don't call this directly.
  - Trio: 9 parameters required.

  ```bash
  checkSnps.sh chromosome isOriginalCaller outfileDir refGenome infilePrefix infileChild infileParent1 infileParent2 "startPos ndPos"
  ```
  - Single: 6 parameters required.

  ```bash
  checkSnps.sh chromosome isOriginalCaller outfileDir refGenome infile "startPos endPos"
  ```


#### `setupParameters.sh`: Setup parameters for the pipeline
 - This should not be call directly.


#### Authors' script
- These two scripts are used by authors to perform the analyese. They record all the dataset, sequence files, refenece genome ... etc used in the analyese.
- The directories and location of these files need to be reconfigured if users
- `runCEU1X_ChromY.sh`: Script for the CEU dataset.
  - Usage: `runCEU1X_ChromY.sh ceuDataset chromosomeNumber`
  - Example: `runCEU1X_ChromY.sh 13 21 #Analyese on CEU13 chromosome 21`
- `runCHM1_ChromY.sh`:  Script for the CHM1 dataset.
  - Usage: `runCHM1_ChromY.sh chromosomeNumber`
  - Example: `runCHM1_ChromY.sh 21 #Analyese on CHM1 chromosome 21`

### Example
```bash
#setup example files
cd WuEtAl2016/SNP/example
tar -xzf example.tar.gz
setupCEU13.sh
#Run example dataset
cd WuEtAl2016/SNP/
pipelineCore.sh
```
The expected output should be someting like the following
```
===== Setup other parameters =====
===== Samtools found. samtools --version-only: ...
...
...
===== Setup =====
DatasetName: CEU13_example
WorkingDir: ./data/CEU13_example_C21//original/
scriptDir: ./
dataDir: ./data/
VariableSiteFile: ./data/snp_chr21_v3_partial.vcf.gz
ReferenceGenome: ./data/human_g1k_v37_c21_partial.fasta
...
...
===== Parse output files =====
Parse calls from trio
Parse calls from single
Get variable sites
Process homos_ref
Process homos_alt
Process hets
```


## `R`: R scripts
- Various R scripts for generating summary tables and figures for the manuscripts.
- Use with cation! Some sections of the scripts might be hard coded or design to run interactively.
- `summarySetup.R`: Setup folders and filenames.
- `summaryFunctions.R`: Lots of functions for various analyses.
- `summaryModel.R`: Analyese results from CEU.
- `summaryModelChm1.R`: Analyese results from CHM1.
- `summaryQqplotExp.R`: QQ plots for CEU.
- `summaryQqplotExpChm1.R`: QQ plots for CHM1.
- `summaryModelPlots.R`: Phi plot.
- `summaryCompareMajMinCnvLcr.R`: Analyses for major vs minor compoents.
