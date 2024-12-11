# MetaEnzMap (Metagenomic Enzyme mapping in Microbe)

## Description

This pipeline processes metagenomic data (primarily bacterial) through standard quality control, assembly, binning, ORF prediction, and protein enzyme function prediction, ultimately generating the distribution profile of microbial enzymes in the metagenomic dataset.

## Attentions

1. This pipeline only supports paired-end fastq files in the phred33 format.
2. This pipeline is mainly used to process high-quality sequencing data. If the sequencing data quality is bad, it may result in the generation of low-quality bins, thereby hindering the subsequent prediction of protein enzyme functions.
3. The complete analytical pipeline requires **GPU** support.

## Hardware requirements

**CPU**: It is recommended to use at least an 8-core, 16-thread processor or higher, with memory ranging from 32GB to 128GB or more.

**GPU**: A GPU with a high number of CUDA cores and large memory is recommended, such as NVIDIA A100, A40, RTX 3090, or V100, with a minimum of 24GB VRAM or higher.

## Steps to install this pipeline (Internet connection required)

### Step 1: prepare the conda environment

1. Download the MetaEnzMap software from github

`git clone https://github.com/Stevenleizheng/MetaEnzMap.git`

2. Go to the directory of MetaEnzMap

`cd MetaEnzMap`

3. Install required softwares

`conda env create -f MetaEnzMap.yaml`

### Step 2: get into the prepared conda environment

`conda activate MetaEnzMap`

### Step 3: install checkm software and database

Detailed installation infomation is seen: (https://github.com/Ecogenomics/CheckM/wiki/Installation#how-to-install-checkm)

Simple installation method: 

The required software for CheckM has already been installed. Next, you only need to install CheckM and the relevant database via pip.

The default path is now inside the folder MetaEnzMap.

```
pip3 install checkm-genome
wget -c https://data.ace.uq.edu.au/public/CheckM_databases
checkm data setRoot <checkm_data_dir>
```

### Step 4: build Bowtie2 indexes for removing host sequences

The default path is now inside the folder MetaEnzMap.

1. download the genomic fasta file of a certain host (e.g. human genome)

```
wget -c https://ftp.ensembl.org/pub/release-112/fasta/homo_sapiens/dna/Homo_sapiens.GRCh38.dna.toplevel.fa.gz
tar xzvf Homo_sapiens.GRCh38.dna.toplevel.fa.gz
mv Homo_sapiens.GRCh38.dna.toplevel.fa human.fa
```

2. build a Bowtie2 index of human 

`python build_index.py -f human.fa -t 4`

### Step 5: install FEDKEA software (FEDKEA needs gpu.)

FEDKEA: A deep learning model for the enzyme annotation of proteins.

Detailed installation infomation is seen: (https://github.com/Stevenleizheng/FEDKEA)

Simple installation method: 

The default path is now inside the folder MetaEnzMap.
1. Download the FEDKEA software from github

`git clone https://github.com/Stevenleizheng/FEDKEA.git`

2. Go to the directory of FEDKEA

`cd FEDKEA`

3. Install the following software

a. pytorch: If you want to use the CPU version, please run conda install pytorch torchvision torchaudio cpuonly -c pytorch. (not recommended)

If you want to use the GPU version, please go to https://pytorch.org/get-started and get the conda or pip install command according to your device and demand. (recommended)

b. fair-esm: `pip install fair-esm==2.0.0`

c. pandas: `pip install pandas==1.4.2`

d. biopython: `conda install -c bioconda biopython=1.78`

e. numpy: `conda install numpy=1.26.2` or `pip install numpy==1.22.3`

f. scikit-learn: `pip install scikit-learn==1.2.0`

4. download the trained model

```
wget -c https://zenodo.org/records/13364055/files/model_param.tar.gz
tar xzvf model_param.tar.gz
```

If the installation was successful, the directory structure of your MetaEnzMap folder will be as follows:  

```
|-- MetaEnzMap  
    |-- bowtie2_index  
        |-- human  
            └── human.1.bt2  
            └── human.2.bt2  
            └── human.3.bt2  
            └── human.4.bt2  
            └── human.rev.1.bt2  
            └── human.rev.2.bt2  
    |-- checkm_data  
    |-- FEDKEA  
    └── bin_quantity.py  
    └── build_index.py  
    └── calculate_rpkm.py  
    └── main.py  
    └── utils.py  
    └── MetaEnzMap.yaml  
    └── README.md  
```

## How to use this pipeline
The pipeline can be used in two modes: a one-step execution or a stepwise execution. Below, I will introduce both usage methods.

### a stepwise execution
Attentions: If your CPU and GPU are not installed on the same server, you will need to use the stepwise execution mode.

The CPU server needs to be configured with the MetaEnzMap-related software, while the GPU server should be set up with the FEDKEA-related software.

**Step 1: quality control, assembly, binning, ORF prediction (on cpu server)**  

By default, the current path is within a folder named 'test', which contains two files: test_1.fq.gz and test_2.fq.gz  

```
|-- test
    └── test_1.fq.gz
    └── test_2.fq.gz
```

`python {path}/MetaEnzMap/main.py -1 test_1.fq.gz -2 test_2.fq.gz -w 32 -x human`

Parameter explanation and other optional parameters:

-1: Path to the fastq(.gz) file of read1

-2: Path to the fastq(.gz) file of read2

-o: Save file path (Recommended to save in the current path)

-w: Threads used to run this pipeline (default:1)

-x: Name of Bowtie2 index (for example: "human") (default: None, which means this step will be skipped)

-s: If you want to run a specific step in the pipeline, you can specify this parameter. 1: run_fastp, 2: run_bowtie2, 3: acquire_contigs, 4: run_bowtie2_build_index, 5: run_bins, 6: run_checkm, 7: select_bins, info_prodigal, 8: run_prodigal, select_prodigal_partial00, default=0

-f: Choose the pipeline's running speed: for a faster option (parameter: 1), select MEGAHIT to assemble contigs [with good quality]; for a slower option (parameter: 0, default), select SPAdes to assemble contigs [with better quality], default=0

After successfully completing this step, the folder structure will be as follows:  

```
|-- test
    |-- 1.fastp 
    |-- 2.bowtie2
    |-- 3.contigs
    |-- 4.index
    |-- 5.bins
    |-- 6.checkm
    |-- 7.high_quality_bins
    |-- 8.medium_quality_bins
    |-- 9.prodigal
    └── test_1.fq.gz
    └── test_2.fq.gz
```

**Step 2: protein enzyme prediction (on gpu server)**

Before running the FEDKEA software on the GPU server, you need to transfer the test folder, the 9.prodigal folder, and the file test_high_bin_protein_partial00.fasta within it to the GPU server. 

Currently, your working directory is within the test folder.  

```
|-- test  
    |-- 9.prodigal
        └── test_high_bin_protein_partial00.fasta
```

run following command:

`python {path}/FEDKEA/main.py -i 9.prodigal/test_high_bin_protein_partial00.fasta -b 32 -d 10.fedkea/ -o 10.fedkea/ -g '0' -a 1`

Detailed usage is seen: (https://github.com/Stevenleizheng/FEDKEA)

After successfully completing this step, the folder structure will be as follows:  

```
|-- test
    |-- 9.prodigal
        └── test_high_bin_protein_partial00.fasta
    |-- 10.fedkea
        └── binary_result.txt
        └── enzyme_result.csv
```

Finally, transfer the results folder 10.fedkea obtained from the GPU server back to the test directory on the CPU server.

Now, the test folder structure will be as follows:  

```
|-- test
    |-- 1.fastp 
    |-- 2.bowtie2
    |-- 3.contigs
    |-- 4.index
    |-- 5.bins
    |-- 6.checkm
    |-- 7.high_quality_bins
    |-- 8.medium_quality_bins
    |-- 9.prodigal
    |-- 10.fedkea
    └── test_1.fq.gz
    └── test_2.fq.gz
```

**Step 3: Metagenomic Enzyme mapping in Microbe (on cpu server)**

The current path is within a folder named 'test'.

Run this command:

`python {path}/MetaEnzMap/calculate_rpkm.py -1 test_1.fq.gz`

Parameter explanation and other optional parameters:

-1: Path to the fastq(.gz) file of read1

-o: Save file path (Recommended to save in the current path)

A results folder named 11.enzyme_rpkm will eventually be generated in the test directory, containing the file enzyme_contig_abundance_rpkm.csv.

**Interpretation of results file**

The first column is the EC number, the second column represents the RPKM value for that class of enzymes, and each subsequent column corresponds to a specific microorganism, with the column headers being the taxonomic names. The value before the asterisk represents the number of ORF proteins predicted to have that enzyme function, while the value after the asterisk represents the abundance of the microorganism.

### a one-step execution