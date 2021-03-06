## SAMSA2 - A complete metatranscriptome analysis pipeline

*****
* Multithreading added for PEAR, Trimmomatic, SortMeRNA
* The script automatically creates a `checkpoint` file in `input_files`. Once a step is finished, it writes the name of that specific step in `checkpoint` and that step is skipped on a rerun of the master_script.
* New version of the master script called `master_script_preserving_unmerged.sh`.  In this script, in the merging step, unmerged reads are concatenated and added to a single file. The forward read and the reverse (complement) read are concatenated with a string of 20 Ns in the middle: This is done through a new R script entitled: `combining_umerged.R`
* Extra care is taken to remove unnecessary files once a step is performed to keep disk usage at a minimum.
* All options, read & program location are to be specified in the first section of the script.
* The script is formated to be run on a HPC using a SLURM job scheduler, but this can be easily changed / removed.
*****
### Usage protocol and pipeline

* Step 1: Install `samsa2` into `/hpc/tools/samsa2/`, be careful, it will be working folder in future analysis.
* Step 2: run `./setup_and_test/package_installation.bash` to install required packages
* Step 3: download all reference with `./setup_and_test/full_database_download.bash`, it is very large
* Step 4: `mkdir ~/hpc/tools/samsa2/input_files` and copy fastq files into this folder
* Step 5: `mkdir ~/hpc/tools/samsa2/result_files` and the results will be set here
* Step 6: before you run any R script, copy all `annot_function.tsv` and `annot_organism.tsv` to a folder
* Step 7: and set this folder as the working_directory for the further R script
* Step 6: pie chart plot: `Rscript Subsystems_pie_charts.R -I ./result/ -O xx -L 5`
* Step 7:`Rscript ./R_scripts/diversity_graphs.R`
* Step 8:`Rscript ./R_scripts/diversity_stats.R`
* Step 9:`Rscript ./R_scripts/get_normalized_counts_table.R`
* Step 10:`Rscript ./R_scripts/get_raw_counts_table.R`
* Step 11:`Rscript ./R_scripts/make_combined_graphs.R`
* Step 12:`Rscript ./R_scripts/make_DESeq_heatmap.R`
* Step 13:`Rscript ./R_scripts/make_DESeq_PCA.R`
* Step 14:`Rscript ./R_scripts/run_DESeq_stats.R`
* Step 15:`Rscript ./R_scripts/Subsystems_pie_charts.R`

### New in version 2:
* Option to annotate against custom databases (`fasta files with stains you want to show`).

### Dependencies
SAMSA2 requires Python2 for aggregation scripts.  Currently, this pipeline works mostly with Python3, although there may be some errors not yet caught.

The following programs can be downloaded OR can be installed from the binaries provided in the programs/ folder.

1. DIAMOND, version 0.8.3: https://github.com/bbuchfink/diamond

2. Trimmomatic, a flexible read cleaner: http://www.usadellab.org/cms/?page=trimmomatic
    ```
    wget http://www.usadellab.org/cms/uploads/supplementary/Trimmomatic/Trimmomatic-Src-0.39.zip
    cd TrimGalore-0.4.5
    ```

3. PEAR, if using paired-end data (recommended): https://sco.h-its.org/exelixis/web/software/pear/

4. SortMeRNA: http://bioinfo.lifl.fr/RNA/sortmerna/
    ```
    wget https://github.com/biocore/sortmerna/archive/2.1.tar.gz
    tar xzvf 2.1.tar.gz
    cd sortmerna-2.1
    ./configure
    make
    ```
## Quick start

1. Download SAMSA2:`git clone https://github.com/transcript/samsa2.git`
2. Either install the dependencies from the links above, or use the setup_and_test/package_installation.bash script provided with SAMSA2 for installing from the included binaries.
3. Access the full databases by downloading them using the full\_database\_download.bash script, located in the setup\_and\_test folder.  This downloads the full RefSeq bacteria database, for organism and specific functional results, and the SEED Subsystems database, which is used for hierarchical functional ontology.
4. Make changes to the master_script.bash, which performs the first 3 of 4 steps in the SAMSA2 pipeline (preprocessing, annotation, aggregation)
5. If not using master_script, use DIAMOND to annotate your reads against a database of your choosing (note that database must be local and DIAMOND-indexed).  See "example\_DIAMOND\_annotation\_script.bash" for more details.
6. If not using master_script, use "DIAMOND\_analysis\_counter.py" to create a ranked abundance summary of the DIAMOND results from each metatransciptome file: `python ~/hpc/tools/samsa2/python_scripts/DIAMOND_analysis_counter.py -I 42_S3_L001 -D ~/hpc/db/nr -O`
7. Import these abundance summaries into R and use "run\_DESeq\_stats.R" to determine the most significantly differing features between either individual metatranscriptomes, or control vs. experimental groups.

## SAMSA: Simple Analysis of Metatranscriptomes by Sequence Annotation
Metatranscriptome, RNA-seq data from multiple members of a microbial community, offers incredibly powerful insights into the workings of a complex ecosystem.  RNA sequences are able to not only identify the individual members of a community down to the strain level, but can also provide information on the activity of these microbes at the time of sample collection - something that cannot be determined through other meta- (metagenome, 16S rRNA sequencing) method.  

However, working with metatranscriptome data often proves challenging, given its high complexity and large size.  SAMSA is one of the first bioinformatics pipelines designed with metatranscriptome data specifically in mind.  It accepts raw sequence data in FASTQ form as its input, and performs four phases:

**Preprocessing:** If the sequencing was paired-end, PEAR is used to merge mate pairs.  Trimmomatic is used for the removal of adaptor contamination and low-quality reads.  SortMeRNA removes ribosomal sequences, as these don't contribute to the mRNA functional profile of the metatranscriptome.

**Annotation:** Annotation is completed using [DIAMOND, an accelerated BLAST-like sequence aligner.](https://github.com/bbuchfink/diamond)  (Why DIAMOND?  At a standard rate of 10 annotations per second, a standard BLAST approach would take several months to finish - just for a single file!)

*Note: DIAMOND is set by default to run in fast mode, where it prioritizes speed.  If you are especially concerned about accuracy, you can change DIAMOND to sensitive mode by editing lines 162 and 199 in the master_script, replacing "--fast" in the DIAMOND command with "--sensitive".  This may significantly increase memory requirements.*

**Aggregation:** DIAMOND returns results on a per-read basis, a bit like a ticker tape or a line item receipt.  In the aggregation step, Python scripts condense these line-by-line results to create summary tables.

**Analysis:** R scripts use DESeq to compute most significantly different features between control vs. experimental samples.  These R scripts generate a tabular output with assigned p-values and log2FoldChange scores for each feature.  These 'features' can be either organisms or specific functions.  R can also create graphs showing visual representation of the metatranscriptome(s).

### Individual programs in SAMSA2 and their functions
For more information, please consult the manual, which goes into more detail on each step in the SAMSA pipeline.

**Preprocessing:** The following program steps can either be run through master_script.bash or individually:

* PEAR - merges two paired-end FASTQ sequencing files together to make a single file of extended fragments.
* Trimmomatic - removes low-quality sequences and/or adaptor contamination.
* SortMeRNA - removes ribosomal reads, as these don't contribute to the mRNA profile of the metatranscriptome.

**Annotation:** This step can be performed through master_script.bash, or individually.

* DIAMOND_example\_script.bash - a guide to structuring commmands for DIAMOND to perform annotations or for building dictionaries, including customizable paths to system locations and reference dictionaries.

**Aggregation:** This step can be performed through master_script.bash, or individually.

* DIAMOND_analysis\_counter.py - creates a summary file from DIAMOND results, condensing identical matches and providing counts.  Can be enabled to return either organism or functional results.
* DIAMOND_subsystems_analysis_counter.py - handles Subsystems hierarchical annotations, keeping all hierarchy levels in the output.
* DIAMOND_specific_organism_counter.py - allows for subsetting of results to obtain only results matching a specific genus, species, or function.

**Analysis:**    
Note: these programs are located in the "R_scripts" folder.  They all require two sets of DIAMOND summarized results from the aggregation step; an experimental set and a control set.

* diversity_stats.R - this program computes Shannon and Simpson diversity scores.
* make_DESeq\_PCA.R - this program creates a PCA plot of the supplied results (either organism or functional category).
* make_DESeq\_heatmap.R - generates a heatmap contrasting control vs. experimental sets.
* make_combined\_graphs.R - generates two stacked bar graphs: a relative version (y axis stretched to 100%) and an absolute version (y axis is raw counts).
* run_DESeq\_stats.R - computes p-values and most significant differences between summarized results sets.

