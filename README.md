# Processing Metagenomes and Resolving Genome-Assembled Genomes

## Description
In this hands-on workshop, we will explore bioinformatics tools for processing metagenomes, focusing on the essential steps to generate and analyze Metagenome-Assembled Genomes (MAGs). By the end of this session, participants will understand how to:

- Perform quality trimming of raw sequencing reads.
- Assemble metagenomes into contigs.
- Conduct genome binning to extract MAGs.
- Assess the quality and completeness of the generated MAGs.
- Assign taxonomy to MAGs using reference databases.

This workflow is widely used in environmental microbiology, allowing researchers to reconstruct microbial genomes from complex communities.

## Genome-Resolved Metagenomics Workflow

### 0. Download the Data
First, let's download the dataset used in this session:
```bash
wget 'https://disc-genomics.uibk.ac.at/data/came24.tar.gz' -O - | tar -zx
cd SLMU
```

### 1. Estimating Sequencing Coverage
We use **Nonpareil** to estimate sequencing redundancy and coverage:
```bash
mkdir -p 00_nonpareil

nonpareil -T kmer -s SLMU.1.fastq.gz -f fastq -b 00_nonpareil/SLMU

NonpareilCurves.R --pdf 00_nonpareil/SLMU.pdf --no-observed 00_nonpareil/SLMU.npo

open 00_nonpareil/SLMU.pdf
```
- **Nonpareil** estimates how much of the microbial diversity has been sequenced.
- The resulting **SLMU.pdf** shows a rarefaction curve; a steep curve indicates high diversity.

### 2. Assembly
Assembly is performed using **SPAdes**, a metagenomic assembler:
```bash
spades.py --meta -1 SLMU.1.fastq.gz -2 SLMU.2.fastq.gz -o 01_assembly
```
- This step can take a long time and require high RAM usage.
- If it's too slow, use the precomputed scaffold file:
```bash
rm -rf 01_assembly

mkdir -p 01_assembly

cp SLMU.scaffolds.fasta 01_assembly/scaffolds.fasta
```

### 3. Mapping Reads to the Assembly
We map sequencing reads back to the assembled scaffolds using **Bowtie2**:
```bash
mkdir -p 02_mapping

bowtie2-build 01_assembly/scaffolds.fasta 02_mapping/SLMU.idx

bowtie2 -1 SLMU.1.fastq.gz -2 SLMU.2.fastq.gz -S 02_mapping/SLMU.sam -x 02_mapping/SLMU.idx --no-unal

samtools view -b 02_mapping/SLMU.sam | samtools sort -o 02_mapping/SLMU.bam -

ls 02_mapping
```
- Mapping allows us to determine which contigs are well-supported by sequencing reads.

### 4. Binning
Binning is performed using **MetaBAT2**, grouping contigs into draft genomes:
```bash
mkdir -p 03_binning

jgi_summarize_bam_contig_depths --outputDepth 03_binning/SLMU.abund 02_mapping/SLMU.bam

metabat2 -i 01_assembly/scaffolds.fasta -a 03_binning/SLMU.abund -o 03_binning/SLMU_bin

ls 03_binning/SLMU_bin.*.fa
```
- **MetaBAT2** uses sequence composition and coverage data to cluster contigs into MAGs.
- The final output consists of individual genome bins (**SLMU_bin.*.fa** files).

### 5. Mapping Reads to MAGs
To further validate the recovered MAGs, we map reads back to them:
```bash
mkdir -p 05_genome_mapping

bowtie2-build 03_binning/SLMU_bin.5.fa 05_genome_mapping/SLMU_bin5.idx

bowtie2 -1 SLMU.1.fastq.gz -2 SLMU.2.fastq.gz -S 05_genome_mapping/SLMU_bin5.sam -x 05_genome_mapping/SLMU_bin5.idx --no-unal
```

### 6. Recruitment Plots
Recruitment plots visualize genome coverage across samples:
```bash
mkdir -p 06_recplot

rpe build -d 06_recplot/SLMU_5.db -r 05_genome_mapping/SLMU_bin5.sam -g 03_binning/SLMU_bin.5.fa --mag

rpe plot -d 06_recplot/SLMU_5.db

mv recruitment_plots 06_recplot/
```
- The resulting **SLMU_bin_recruitment_plot.html** helps assess the coverage and depth of MAGs across metagenomes.
- Open the file for visual inspection:

```bash
06_recplot/recruitment_plots/SLMU_bin5_sam/SLMU_bin_recruitment_plot.html
```
### 7. Taxonomic Classification of MAGs

Using online **MiGA** tool
https://uibk.microbial-genomes.org/


**In our COMPUTER**
To compare a MAG against closely related genomes, we use **MiGA**:
```bash
miga new -P 04_classification/Sal -t genomes

miga gtdb_get -P 04_classification/Sal -T g__Salinibacter --ref -v

miga add -P 04_classification/Sal -i assembly -t popgenome 03_binning/SLMU_bin.*.fa

miga index_wf -o 04_classification/Sal -v
```
- This step retrieves reference genomes and performs a taxonomic classification of our MAGs.
- Once complete, open the classification results:
```bash
04_classification/Sal/index.html
```

## Software Used
This workflow relies on various bioinformatics tools:
- **ARB** – for phylogenetic analysis.
- **QIIME2** – for microbiome analysis.
- **Nonpareil** – for sequencing coverage estimation.
- **SPAdes** – for metagenomic assembly.
- **Bowtie2** – for read alignment.
- **MetaBAT2** – for genome binning.
- **MiGA** – for genome classification.
- **RecruitPlotEasy (RPE)** – for recruitment plot visualization.
- Additional dependencies required by the above tools.

By the end of this workshop, you should be able to analyze metagenomic data and reconstruct microbial genomes with confidence!

