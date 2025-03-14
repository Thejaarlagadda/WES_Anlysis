# WES_Anlysis

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


## esearch and efetch are not available in your environment. These tools are part of NCBI Entrez Direct, which is separate from SRA Toolkit.

cd ~
sh -c "$(curl -fsSL https://ftp.ncbi.nlm.nih.gov/entrez/entrezdirect/install-edirect.sh)"
export PATH=${HOME}/edirect:$PATH


## Step 1: Download Data We retrieved WES data from NCBI SRA using:

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



