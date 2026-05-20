# Public Datasets for Bioinformatics Labs

Use small public datasets for training. Avoid downloading huge files until your cost controls are in place.

---

## SRA Lite - NCBI Sequence Read Archive

**What it is:** public sequencing data repository.

Install tools:

```bash
# Ubuntu
sudo apt update
sudo apt install -y sra-toolkit

# macOS
brew install sra-tools
```

Download small examples:

```bash
fasterq-dump SRR2584857 -q
fasterq-dump SRR5863885 -q
```

Find datasets:

- Go to https://www.ncbi.nlm.nih.gov/sra
- Search by organism or keyword.
- Copy an accession such as `SRR...`.

---

## 1000 Genomes Project

**What it is:** public human variation dataset.

Reference file example:

```bash
wget http://ftp.1000genomes.ebi.ac.uk/vol1/ftp/technical/reference/README_reference_genomes.txt
```

AWS Open Data example:

```bash
aws s3 ls s3://1000genomes/ --no-sign-request
```

Download only small subsets for training.

---

## GTEx

**What it is:** gene expression across human tissues.

```bash
wget https://storage.googleapis.com/gtex_analysis_v8/rna_seq_data/GTEx_Analysis_v8_RNA-SEQ_read_counts_T.txt.gz
gunzip GTEx_Analysis_v8_RNA-SEQ_read_counts_T.txt.gz
head GTEx_Analysis_v8_RNA-SEQ_read_counts_T.txt | cut -f1-5
```

Quick Python inspection:

```python
import pandas as pd

df = pd.read_csv("GTEx_Analysis_v8_RNA-SEQ_read_counts_T.txt", sep="\t", index_col=0)
print(df.shape)
print(df.iloc[:5, :5])
```

---

## TCGA

**What it is:** cancer genomics data.

Portal:

```text
https://portal.gdc.cancer.gov/
```

AWS Open Data examples change over time. Always inspect bucket paths before downloading large data.

---

## GENCODE

**What it is:** gene annotation and transcript reference data.

```bash
wget ftp://ftp.ebi.ac.uk/pub/databases/gencode/Gencode_human/release_45/gencode.v45.basic.annotation.gff3.gz
gunzip gencode.v45.basic.annotation.gff3.gz
head gencode.v45.basic.annotation.gff3 | grep -v "^#"
```

Count feature types:

```bash
cut -f3 gencode.v45.basic.annotation.gff3 | sort | uniq -c | sort -nr | head
```

---

## Starter Tutorial: FASTQ QC

```bash
mkdir -p ~/bioinformatics
cd ~/bioinformatics

fasterq-dump SRR2584857 -q
fastqc SRR2584857.fastq -o . -q

python3 << 'EOF'
from Bio import SeqIO
import numpy as np

reads = list(SeqIO.parse("SRR2584857.fastq", "fastq"))
print(f"Reads: {len(reads)}")
print(f"Average length: {np.mean([len(r.seq) for r in reads]):.0f}")
EOF
```

Upload result:

```bash
aws s3 cp SRR2584857.fastq s3://your-bucket/raw-data/
```

---

## Efficient Download Strategy

For cloud labs:

1. Download data directly inside the VM.
2. Process data near storage.
3. Upload only final results.
4. Delete temporary large files.
5. Avoid moving large datasets to your laptop unless necessary.
