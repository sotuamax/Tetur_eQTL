# Expression QTL (eQTL) of <i>Tetranychus urticae</i> (a generalist spider-mite herbivore)
This is a repo for the data analysis pipeline of Tetranychus urticae eQTL project. </br>
eQTL is QTL explaining gene expression, and it can be identified via association analysis between genotype and gene expression.

## Experimental design
  To initiate eQTL project, we collected a total of 458 isogenic pools of the recombinant inbred lines (RIL). Briefly, a susceptible ROS-ITi (diploid mother, ♀) and more resistant MR-VPi (haploid father, ♂) inbred strains are employed as the founder strains (F0). By crossing the two parental strains, we collected F1 female (diploid). And F1 female lay eggs without ferterlization developing into males (F2, haploid), which are back crossed to the susceptible ROS-ITi female. For each F2 male backcross, all offsprings are collected to generate one single isogenic pool for RNA-seq extraction. 

## Table of contents

- [DNA-seq for variants calling](#DNA-seq-for-variants-calling)
- [Map RNA-seq against the reference genome](#Map-RNA-seq-against-the-reference-genome)
- [Genotype call for RILs based on RNA-seq alignment](#Genotype-call-for-RILs-based-on-RNA-seq-alignment)
- [Update GFF3 file for the reference genome](#Update-GFF3-file-for-the-reference-genome)
- [Gene expression level quantification](#Gene-expression-level-quantification)
- [Differential gene expression analysis](#Differential-gene-expression-analysis)
- [Association analysis between genotype and gene expression](#Association-analysis-between-genotype-and-gene-expression)
- [Allele-specific expression for determination of <i>cis</i>-distance](#Allele-specific-expression-for-determination-of-cis-distance)
- 

## Programs

- python3+ (packages: pysam v0.15.3; biopython v1.76; pandas v0.25; numpy v1.21; mpi4py v3.0)
- BWA v0.7.17-r1188
- STAR v2.7.3a
- GATK v4.0
- GSNAP version 2020-06-30
- htseq-count v2.0.1
- RSEM v1.3.3
- R v4.1.3 (packages: DESeq2 v1.34; MatrixEQTL v2.3; R/qtl v1.46; )
- 

[NOTE]To enable parallel processing, python model mpi4py need to be installed. 

## DNA-seq for variants calling
To call variants for the inbred ROS-ITi and MR-VPi strains, we mapped illumina DNA-seq against the three-chromosome reference genome (London strain, see [Wybouw, Kosterlitz, et al., 2019](https://academic.oup.com/genetics/article/211/4/1409/5931522)). <br>
GATK best practice for variants calling is refered [here](https://gatk.broadinstitute.org/hc/en-us/sections/360007226651-Best-Practices-Workflows). <br>
1. First, prepare index for the genome fasta file;
```bash
# make directory for bwa index files
mkdir bwa_index
# change working directory to the folder
cd bwa_index
# generate index files using bwa index command
bwa index Tetranychus_urticae_2017.11.21.fasta
```
2. Then, map DNA-seq of ROS-ITi and MR-VPi onto the reference fasta genome using BWA;
```bash
# make directory for bwa mapping
mkdir bwa_map
# change working directory to the mapping folder
cd bwa_map
# run bwa mapping for ROS-ITi sample (paired-end DNA sequences)
bwa mem -t 20 -R "@RG\tID:20190412_8\tSM:ROS-ITi\tPL:Illumina\tLB:ROS-ITi" bwa_index/Tetranychus_urticae_2017.11.21.fasta r1.fastq.gz r2.fastq.gz | samtools view -Su - | samtools sort -@ 20 - -o ROS-ITi.BWA.bam
# run bwa mapping for MR-VPi sample (paired-end DNA sequences)
bwa mem -t 20 -R "@RG\tID:20190312\tSM:MR-VPi\tPL:Illumina\tLB:MR-VPi" bwa_index/Tetranychus_urticae_2017.11.21.fasta r1.fastq.gz r2.fastq.gz | samtools view -Su - | samtools sort -@ 20 - -o MR-VPi.BWA.bam
```
3. Mark duplicated reads that are arised from PCR using picard MarkDuplicate;
```bash
# mark duplicated reads in BWA mapping files
picard MarkDuplicates I=ROS-ITi.BWA.bam O=ROS-IT_duplicate.bam M=ROS-IT_metrics.txt && samtools index ROS-IT_duplicate.bam
picard MarkDuplicates I=MR-VPi.BWA.bam O=MR-VP_duplicate.bam M=MR-VP_metrics.txt && samtools index MR-VP_duplicate.bam
# left align insertion and deletion mappings (optional)
gatk LeftAlignIndels -R Tetranychus_urticae_2017.11.21.fasta -I ROS-IT_duplicate.bam -O ROS-IT_leftalign.bam
gatk LeftAlignIndels -R Tetranychus_urticae_2017.11.21.fasta -I MR-VP_duplicate.bam -O MR-VP_leftalign.bam
```
4. Run the Best practice of GATK pipeline for variant calling;
```bash
# run gatk HaplotypeCaller to generate g.vcf file
gatk HaplotypeCaller -R Tetranychus_urticae_2017.11.21.fasta -I ROS-IT_leftalign.bam -ERC GVCF -O ROS-IT.g.vcf.gz
gatk HaplotypeCaller -R Tetranychus_urticae_2017.11.21.fasta -I MR-VP_leftalign.bam -ERC GVCF -O MR-VP.g.vcf.gz
# run gatk GenotypeGVCFs to call variants in vcf file
gatk GenotypeGVCFs -R Tetranychus_urticae_2017.11.21.fasta -V ROS-IT.g.vcf.gz -O ROS-IT.vcf.gz
gatk GenotypeGVCFs -R Tetranychus_urticae_2017.11.21.fasta -V MR-VP.g.vcf.gz -O MR-VP.vcf.gz
```
5. Select variants data in unfiltered vcf file;
```bash
### run the following steps for ROS-ITi and MR-VPi, respectively
# select SNPs
gatk SelectVariants -R Tetranychus_urticae_2017.11.21.fasta -V input.vcf.gz -select-type-to-include SNP -O SNP.vcf.gz
# select INDELs (insertion and deletion, optional)
gatk SelectVariants -R Tetranychus_urticae_2017.11.21.fasta -V input.vcf.gz -select-type-to-include INDEL -O INDEL.vcf.gz
# sort vcf files
bcftools sort -o sorted.vcf.gz -O z SNP.vcf.gz
# add index for sorted vcf file
bcftools index -t sorted.vcf.gz
# Filter SNPs based on RMS mapping quality and genotype field information (run script vcf_pass.py)
vcf_pass.py -vcf sorted.vcf.gz -R Tetranychus_urticae_2017.11.21.fasta -O filtered.vcf.gz
```
For Variants filtering, see [here](https://gatk.broadinstitute.org/hc/en-us/articles/360035890471-Hard-filtering-germline-short-variants) also for hard-filtering. <br>

6. Pick SNPs that are distinguishable between the two inbred parental strains (ROS-ITi vs. MR-VPi). Output in tab-separated file
```bash
# Comparing filtered VCF files for ROS-IT and MR-VP, and pick genotype-calls different between them
vcf_compare.py -vcf1 ROS-IT.filtered.vcf.gz -vcf2 MR-VP.filtered.vcf.gz -R Tetranychus_urticae_2017.11.21.fasta -O variant_ROSIT.vs.MRVP
```
## Map RNA-seq against the reference genome
The three-chromosome reference genome was used, the same for DNA-seq mapping. <br>
We used two RNA-seq aligners, [STAR](https://github.com/alexdobin/STAR) and [GSNAP](https://github.com/juliangehring/GMAP-GSNAP), for RNA-seq mapping.

1. Generate indices for genome fasta file.
```bash
# STAR index generation
STAR --runMode genomeGenerate --runThreadN 30 --genomeDir STAR_index --genomeFastaFiles Tetranychus_urticae_2017.11.21.fasta --genomeSAindexNbases 12
# GSNAP index generation
gmap_build -D GSNAP_index -d Tetur Tetranychus_urticae_2017.11.21.fasta
```
2. Map RNA-seq onto the reference genome given the index folder. 
```bash
# STAR mapping, sort and index BAM alignment file
STAR --genomeDir STAR_index --runThreadN 20 --readFilesIn r1.fastq.gz r2.fastq.gz --twopassMode Basic --sjdbOverhang 99 --outFileNamePrefix sample_name. --readFilesCommand zcat --alignIntronMax 30000 --outSAMtype BAM Unsorted && samtools sort sample_name.Aligned.out.bam -o sample_name_sorted.bam -@ 8 && samtools index sample_name_sorted.bam 
```
  SNP-tolerant mapping using GSNAP require preparation of variant files
```bash
# make folder for SNP data
mkdir Tetur_SNP
# Format SNP allele information from vcf_compare.py output (see above) using SNP_prep.py
# SNP_prep.py [input] [output]
SNP_prep.py variant_ROSIT.vs.MRVP.txt SNP_allele
# Prepare SNP information for GSNAP
cat SNP_allele.txt | iit_store -o SNP_allele
mv SNP_allele.iit Tetur_SNP/Tetur_SNP.maps
# create a reference space index and compressed genome
snpindex -d Tetur_SNP -v SNP_allele -D . -V .
# prepare know splice sites 
cat gtf_file | gtf_splicesites > Tu.splicesites
cat gtf_file | gtf_introns > Tu.introns
cat Tu.splicesites | iit_store -o Tu_splicesites
cat Tu.introns | iit_store -o Tu_introns
# move all generated file into genome index db, then run GSNAP mapping
gsnap -d GSNAP_index -N 1 -D . --gunzip -s Tu_splicesites -v SNP_allele -t 20 -A sam r1.fastq.gz r2.fastq.gz | samtools sort -o sample_name.bam -O bam -@ 20 - && samtools index sample_name.bam
```
## Genotype call for RILs based on RNA-seq alignment
We developed a customized pipeline for genotyping purposes of RIL isogenic pools. 
Inputs:
  - BAM file of RILs in coordinates sorted fashion and with its index file;
  - SNPs information that are distinguishable between the two inbred stains (output of vcf_compare.py, see above).

1. Count allele-specific reads on SNP sites for each sample separately
```bash
# run genotype_allele.py to count allele-specifc reads on the SNP sites
# this is a multiple-core processing program, adjust core usage via "-n"
mpiexec -n 10 genotype_allele.py -V variant_ROSIT.vs.MRVP.txt -bam sample_name.bam -O sample_allele_count
```
After running for all samples, place all of them in the same folder (raw_count). <br>

2. Collect genotype information for all samples, and count the genotype frequency at each SNP site  
```bash
# run genotype_freq.py for genotype frequency, either in homozygous or heterozygous genotype
# this is a multiple-core processing program, adjust core usage via "-n"
mpiexec -n 10 genotype_freq.py -dir raw_count -O SNP_geno_freq
```
3. A backcross experimental design indicates a 1:1 ratio of heterozygous:homozygous genotype at each SNP site. We performed Chi-square goodness of fit test to filter bad SNP sites which doesn't fit the ratio (adjusted p < 0.01). <br>

    See ```chisq_bad.Rmd```

4. For raw allele-specific read count, clean the dataset by dropping bad SNPs from last step
```bash
# run clean_count.R script to drop SNP rows in bad_SNP file
Rscript clean_count.R -raw sample_allele_count.txt -bad bad_SNPs.txt -O sample_allele_count.clean.txt
```
5. Call genotypic blocks based on allele-specific read count of good SNPs 
You need to set up the chromosomes of interested for genotype block assignment, and also provide chromosome length information in a tab-separated file. About how to provide chromosome length, see [here](https://biopython.org/docs/1.75/api/Bio.SeqIO.html). Example data set see under data folder.
```bash
# run genotype_block.py to call genotype blocks that arised from crossingover events 
genotype_block.py -chr chr.txt -chrLen chrlen.txt -C sample_allele_count.clean.txt -O sample_genotype_block
```
6. Visulization of genotype blocks on chromosome level
```bash
Rscript block_vis.R -geno sample_genotype_block.txt
```
  Example output (in PDF): <br>
<img width="300" alt="Screen Shot 2022-05-01 at 3 15 45 PM" src="https://user-images.githubusercontent.com/63678158/166165078-eaeace45-abfc-48ca-9301-684e0f670db4.png">

## Update GFF3 file for the reference genome
  To integrate all annotated gene information in the current three-chromosome reference genome, we added gene models from [Orcae database](https://bioinformatics.psb.ugent.be/gdb/tetranychus/) (version of 01252019) and updated the current GFF3 file. <br>
  To transfer gene models on fragmented scaffold genomes onto three-chromosome genome scale, see script ```GFF_record.py``` under GFF_update folder. <br>
  Combine the added gene models to the current GFF3 file and sort it using [gff3sort.pl](https://github.com/billzt/gff3sort). <br>
  Transform from GFF3 to GTF format using script ```gff2gtf.py``` under GFF_update folder. <br>
  To compress and add index for GFF/GTF, see below:
```bash
# sort gff using gff3sort.pl
gff3sort.pl input.gff > output.gff
# compress gff
bgzip output.gff
# add index for compressed gff
tabix -p gff output.gff.gz
```
## Gene expression level quantification
1. By taking the updated GFF version, we run htseq-count on the RNA-seq alignment BAM files and output read count on gene basis.
```bash
# htseq-count command line (adjust number of CPU "-n" based on sample BAMs number, per BAM per CPU)
htseq-count -r pos -s reverse -t exon -i gene_id --nonunique none --with-header -n 3 -c sample1-3.tsv sample1.bam sample2.bam sample3.bam $GTF 
```
2. For absolute read-count from htseq-count output, we performed library-size normalization using DESeq2 (under normalization folder). <br>
```bash 
# merge htseq-count output of all samples into one file, -O for output name
Rscript DESeq2_norm.R -count all_sample.txt -O all_sample_normalizedbyDESeq2
```
3. To calculate gene expression abundance on transcript per million (TPM) level and Fragments Per Kilobase of transcript per Million (FPKM) level, we run [RSEM](https://github.com/deweylab/RSEM). 
```bash
# prepare reference index for rsem expression calculation
rsem-prepare-reference --gtf $GTF --star --star-sjdboverhang 150 $GENOME Tetur_rsem -p 20
# calculate expression level using paired-end reads
rsem-calculate-expression --star-gzipped-read-file --paired-end --star --strandedness reverse -p 10 r1.fastq.gz r2.fastq.gz Tetur_rsem sample_name
```
## Differential gene expression analysis
We employed DESeq2 for differential expression analysis.
1. Prepare sample information for DESeq2 including conditions in contrast
```bash

```
2. Perform differential expression analysis by assigning contrasted conditions for comparison
```bash

```
## Association analysis between genotype and gene expression
  For each sample, its genotype blocks and gene expression data are available.
1. To alleviate the effects from outlier expression data and alleviate systematic inflation problem, we performed quantile normalization on gene expression data, an accepted remedy by the GTEx consortium (see also [here](http://www.bios.unc.edu/research/genomic_software/Matrix_eQTL/faq.html)). The normalization is on individual gene level, and gene expression quantity across all samples fit normal distribution while preserving the relative rankings. Run quantile_norm.R (under normalization folder):
```bash 
# input count file should be normalized by library-size (see Gene expression level quantification 2)
Rscript quantile_norm.R -norm_count all_sample_normalizedbyDESeq2 -O all_sample_quantile_normalization
``` 
    [Notice] Genes with absolute (raw) read-count<10 in more than 80% samples of the whole population are considered to be very lowly expressed and filtered out from further analysis.

2. To alleviate computational pressure for association tests that run for each combination of SNP genotype and individual gene expression level, we assigned genotype bins based on the overlap of genotype blocks among isogenic populations. 
```bash
# make a directory to store all samples' genotype block information
mkdir sample_genotype_block
# move genotype_block information (output from genotype_block.py) to the directory
mv sample_genotype_block*.txt sample_genotype_block/
# run block2bin.R to assign genotypic bins based on overlapped blocks 
# for this step, you also need to provide chromosome length information (chrlen.txt) and SNP position file (SNP_loc.txt)
Rscript block2bin.R -genodir sample_genotype_block/ -chrLen chrlen.txt -SNP SNP_loc.txt
```
3. Perform genotype-expression association analysis using [MatrixeQTL](https://github.com/andreyshabalin/MatrixEQTL). <br>
  About how to prepare input files for MatrixeQTL, see its tutorial [here](http://www.bios.unc.edu/research/genomic_software/Matrix_eQTL/runit.html).
```bash 
# command for using MatrixeQTL for association test (without taking gene/SNP location information)
# both ANOVA and LINEAR models are used for the association analysis, and output files in *.anova.txt and *.linear.txt
Rscript eQTL_identify.R -genotype <genotype.txt> -expression <expression.txt> -O <output>
```
4. Significant associations can arise from linkage disequilibruim (LD). To alleviate its effect, we take the bin genotype data to reconstruct linkage groups using [R/qtl](https://rqtl.org/download/). <br>

  For linkage group construction, see R script ```marker_association.R```

5. Based on the linkage group information, we parsed significant associations for each bin and its target gene. The most significant bin of a given linkage group was chosen as the "causal" eQTL. 
```bash
# use the developed script parse_eQTL.py (support multiple-core running, adjust core usage in "-n")
mpiexec -n 5 eQTL_parse.py -eQTL MatrixeQTL_output -assoc marker_association.txt -O parsed_eQTL
# the marker_association.txt is output from the last step 
```
## Allele-specific expression for determination of <i>cis</i>-distance 

One character that distinguish of cis from trans-regulatory element is the biased allele-specific expression (ASE) arising from cis control. To take that into consideration for our assignment of cis/trans, we performed ASE on gene basis.

1. Based on the SNP information on gene exon region, we count reads for allelic expression. 
```bash
# Prepare a file with all SNP data on exon region (example see data/SNP_exon.txt)
# run ASE_gene.py for allelic expression on individual gene (multi-core process, adjust core usage in "-n") by taking GTF and RNA-alignment BAM file
mpiexec -n 10 python ASE_gene.py -SNP SNP_exon.txt -gtf $GTF -bam $bam -O output
```
2. By taking gene allele-specific count and genotype file for individual isogenic population, only genes on heterozygous regions are informative for following analysis. 
```bash 
# you should prepare all samples' ASE files under one directory and all samples' genotype files under one directory, and also provide gene coordinate information as input to take genes' ASE within heterzygous region for following analysis.
python ASE_cis.py -countdir <ASE_dir> -genodir <geno_dir> -gene <gene_loc> -O output
```
3. By taking allele-specific count of all samples (on heterozygous region, from last step), ASE ratio (log2 transformed) was calculated on gene-basis. Student's t-test was performed to test if significant bias in ASE was observed that can arised from cis-regulatory control. 
```bash 
Rscript cis_effect.R -ASE <ASE> -remove_homo -O output
```
4. 








