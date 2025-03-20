# Whole-Exome Sequencing Analysis in Inflammatory Breast Cancer

This repository contains scripts and workflows for replicating the WES analysis from the paper:

**"Whole-exome sequencing identifies somatic mutations and intratumor heterogeneity in inflammatory breast cancer"**  
Published in *npj Breast Cancer (2021)* [Link](https://www.nature.com/articles/s41523-021-00278-w)

## 🔬 **Project Workflow**
1. **Download Data** from NCBI SRA (PRJNA713359)
2. **Quality Control** using FastQC & Trimming
3. **Read Alignment** using BWA
4. **Variant Calling** using GATK MuTect2
5. **Copy Number Alteration (CNA) Analysis** using TITAN
6. **Subclone Identification** using PyClone
7. **Mutation Signature Analysis** using MutationalPatterns in R

## 📁 **Directory Structure**

```bash
# Reference files
wget -c ftp://hgdownload.cse.ucsc.edu/goldenPath/hg19/bigZips/hg19.fa.gz
gunzip hg19.fa.gz


# bwa indexing
bwa index ~/WES_project/reference/hg19.fa

# samtools indexing
samtools faidx ~/WES_project/reference/hg19.fa

# gatk creating dictionary
gatk CreateSequenceDictionary -R ~/WES_project/reference/hg19.fa -O ~/WES_project/reference/hg19.dict

```

### Downloading Known Sites for BQSR
BQSR requires known sites of variation. Since you are using GRCh37, we will need:

dbSNP vcf: Contains common SNPs.
1000G Omni vcf: Commonly known variants from the 1000 Genomes Project.
1000G Phase 1 indels: Indels from the 1000 Genomes Project.
Mills and 1000G gold standard indels: High-confidence indels.

```bash
# Download Known sites from GATK Resource Bundle
wget -c https://storage.googleapis.com/gcp-public-data--broad-references/hg19/v0/dbsnp_138.b37.vcf.gz
wget -c https://storage.googleapis.com/gcp-public-data--broad-references/hg19/v0/1000G_phase1.snps.high_confidence.b37.vcf.gz

wget -c https://storage.googleapis.com/gcp-public-data--broad-references/hg19/v0/1000G_phase1.indels.b37.vcf.gz
# If it is not worked use this command for this site only 1000G_phase1.indels.b37.vcf.gz
gsutil cp gs://gatk-legacy-bundles/b37/1000G_phase1.indels.b37.vcf.gz .

wget -c https://storage.googleapis.com/gcp-public-data--broad-references/hg19/v0/Mills_and_1000G_gold_standard.indels.b37.vcf.gz
```



```bash
## esearch and efetch are not available in your environment. These tools are part of NCBI Entrez Direct, which is separate from SRA Toolkit.
cd ~
sh -c "$(curl -fsSL https://ftp.ncbi.nlm.nih.gov/entrez/entrezdirect/install-edirect.sh)"
export PATH=${HOME}/edirect:$PATH
```

### Step 1: Download Data We retrieved WES data from NCBI SRA using:
```bash
## command to list available sequencing runs under PRJNA713359:
esearch -db sra -query PRJNA713359 | efetch -format runinfo > PRJNA713359_runinfo.csv

# Download Raw FASTQ Files
## Example: Download a specific SRA run (Replace SRR_ID with actual ID) for single files
prefetch SRR_ID  # Download SRA file

## Convert SRA to FASTQ format
fastq-dump --split-files --gzip SRR_ID

## to download Multiple sra files we have to use a loop
cat PRJNA713359_runinfo.csv | awk -F',' '{print $1}' | grep SRR > SRA_files/SRR_ids.txt
cd SRA_files

# Download all SRA files
for id in $(cat SRR_ids.txt); do
    prefetch $id
    fastq-dump --split-files --gzip $id
done

# Run the following command to compare downloaded files with the expected list:
comm -23 <(sort SRR_ids.txt) <(ls | grep SRR | sort)

# Redownload Missing Files
for id in $(comm -23 <(sort SRR_ids.txt) <(ls | grep SRR | sort)); do
    prefetch $id
    fastq-dump --split-files --gzip $id
done
```
## We will start with the main workflow now

```bash
# -------------------
# STEP 1: QC - Run fastqc 
# -------------------
echo "STEP 1: QC - Run fastqc"

