# Datasets
---
### **SRA Lite (Sequence Read Archive - NCBI)**

**O que é:** Repositório público de sequenciamento global

**Acesso gratuito:**
```bash
# Instalar SRA Toolkit
# macOS
brew install sra-tools

# Linux (Amazon Linux 2)
sudo yum install -y sra-tools

# Ubuntu
sudo apt install -y sra-tools

# Verificar instalação
vdb-config --help
```

**Baixar dados pequenos (starter):**
```bash
# Dataset muito pequeno (~5 MB) - bactéria
fasterq-dump SRR2584857
# Arquivo resultante: ~20 MB FASTQ

# Dataset pequeno (~30 MB) - fungo
fasterq-dump SRR5863885
# Arquivo: ~100 MB FASTQ

# Dataset médio (~200 MB) - humano
fasterq-dump SRR1234567  # (exemplo)
```

**Encontrar datasets:**
- Ir em: https://www.ncbi.nlm.nih.gov/sra
- Procurar por palavra-chave (ex: "Arabidopsis")
- Copiar ID (SRRxxx)

---

### **1000 Genomes Project**

**O que é:** Genomas sequenciados de 2.504 indivíduos (pública)

**Acesso:**
```bash
# Genomas de referência (pequenos)
wget http://ftp.1000genomes.ebi.ac.uk/vol1/ftp/technical/reference/README_reference_genomes.txt

# Chromosome humano (tamanho variável)
# Chr22 (~50 MB, menor)
wget ftp://ftp.ncbi.nlm.nih.gov/genomes/H_sapiens/Assembled_chromosomes/seq/hs_ref_GRCh37.p13_chr22.fa.gz

# Chr1 (~250 MB, maior)
wget ftp://ftp.ncbi.nlm.nih.gov/genomes/H_sapiens/Assembled_chromosomes/seq/hs_ref_GRCh37.p13_chr1.fa.gz

# Descompactar
gunzip hs_ref_GRCh37.p13_chr22.fa.gz
```

**Via AWS S3 (mais rápido):**
```bash
# 1000 Genomes está em bucket S3 público
aws s3 ls s3://1000genomes/ --no-sign-request

# Baixar aligned BAM (chr22, amostra)
aws s3 cp s3://1000genomes/phase3/data/HG00096/alignment/HG00096.mapped.ILLUMINA.bwa.GBR.high_coverage_pcr_free.20140203.bam . --no-sign-request --region us-east-1
# Arquivo: ~200 MB
```

**Uso prático:**
```bash
# Praticar samtools/bwa com dados reais pequenos
samtools index HG00096.mapped.ILLUMINA.bwa.GBR.high_coverage_pcr_free.20140203.bam
samtools view -h arquivo.bam | head -20
```

---

### **GTEx (Genotype-Tissue Expression)**

**O que é:** Expressão gênica em 53 tecidos humanos diferentes

**Acesso web:**
```
https://www.gtexportal.org/
```

**Baixar dados (programático):**
```bash
# Portal de downloads
wget https://storage.googleapis.com/gtex_analysis_v8/rna_seq_data/GTEx_Analysis_v8_RNA-SEQ_read_counts_T.txt.gz

# Descompactar
gunzip GTEx_Analysis_v8_RNA-SEQ_read_counts_T.txt.gz

# Explorar
head GTEx_Analysis_v8_RNA-SEQ_read_counts_T.txt | cut -f1-5
```

**Python: Análise rápida**
```python
import pandas as pd

# Carregar
df = pd.read_csv('GTEx_Analysis_v8_RNA-SEQ_read_counts_T.txt', sep='\t', index_col=0)

# Visualizar
print(df.shape)  # Número de genes × amostras
print(df.iloc[:5, :5])  # Primeiros genes

# Análise
print(df.mean(axis=1).sort_values(ascending=False).head(10))  # Genes mais expressos
```

---

### **TCGA (The Cancer Genome Atlas) - via AWS**

**O que é:** Dados genômicos de >11.000 tumores

**Acesso (gratuito via AWS):**
```bash
# TCGA está em bucket S3 público e também em "TCGA harmonized" via Broad Institute

# Via AWS Open Data
# Listar dados (requer AWS CLI)
aws s3 ls s3://gdc-center-of-excellence/ --no-sign-request

# Genomes inteiros (GBs) - use se tiver paciência
# Dados de expressão (MBs) - melhor para começar
```

**Alternativa: GDC Portal (mais fácil)**
```
https://portal.gdc.cancer.gov/
- Clicar "Download Portal"
- Selecionar cancer type
- Selecionar tipo de dado (RNA-seq, SNPs, etc)
- Baixar manifestos e arquivos
```

---

### **Gencode - Referência Gênica (RECOMENDADO!)**

**O que é:** Anotação completa de genes humanos

**Baixar:**
```bash
# Arquivo GFF3 (~100 MB compactado)
wget ftp://ftp.ebi.ac.uk/pub/databases/gencode/Gencode_human/release_45/gencode.v45.annotation.gff3.gz

# Arquivo FASTA de genes (~300 MB)
wget ftp://ftp.ebi.ac.uk/pub/databases/gencode/Gencode_human/release_45/gencode.v45.transcripts.fa.gz

# Descompactar
gunzip gencode.v45.annotation.gff3.gz
gunzip gencode.v45.transcripts.fa.gz
```

**Explorar:**
```bash
# Ver estrutura
head gencode.v45.annotation.gff3 | grep -v "^#"

# Contar genes
grep -c "^#.*chromosome" gencode.v45.annotation.gff3

# Filtrar apenas exons
grep "exon" gencode.v45.annotation.gff3 | head

# Com Python (BED tools)
bedtools merge -i <(grep "exon" gencode.v45.annotation.gff3 | awk '{print $1"\t"$4"\t"$5}')
```

---

## DATASETS ESPECÍFICOS POR ORGANISMO

### Humano
- **1000 Genomes:** Variação genética populacional
- **GTEx:** Expressão gênica por tecido
- **TCGA:** Dados tumorais
- **Gencode:** Anotação de genes

### Plantas
```bash
# Arabidopsis (pequeno, ~100 MB)
wget ftp://ftp.arabidopsis.org/home/tair/Genes/TAIR10_genome_release/

# Maize (intermediário)
wget http://ftp.maizesequence.org/
```

### Bactérias
```bash
# Escherichia coli K-12 (MUITO pequeno, ~5 MB)
wget ftp://ftp.ncbi.nlm.nih.gov/genomes/Bacteria/Escherichia_coli_K_12/

# Salmonella (pequeno)
wget ftp://ftp.ncbi.nlm.nih.gov/genomes/Bacteria/Salmonella_enterica/
```

### Fungos
```bash
# Saccharomyces cerevisiae (pequeno, ~12 MB)
wget ftp://ftp.ncbi.nlm.nih.gov/genomes/Fungi/Saccharomyces_cerevisiae/

# Candida albicans
wget ftp://ftp.ncbi.nlm.nih.gov/genomes/Fungi/Candida_albicans/
```

---

## TUTORIAIS COM DATASETS PRONTOS

### Tutorial 1: Análise FASTQ Básica (SRA pequeno)

```bash
# 1. Download dataset pequeno
cd ~/bioinformatica
fasterq-dump SRR2584857 -q  # silencioso

# 2. QC
fastqc SRR2584857.fastq -o . -q

# 3. Análise Python
python3 << 'EOF'
from Bio import SeqIO
import numpy as np

reads = list(SeqIO.parse("SRR2584857.fastq", "fastq"))
print(f"Total reads: {len(reads)}")
print(f"Read médio: {np.mean([len(r.seq) for r in reads]):.0f} bp")
print(f"Qualidade média: {np.mean([np.mean(r.letter_annotations['phred_quality']) for r in reads]):.2f}")
EOF

# 4. Upload resultado
aws s3 cp SRR2584857.fastq s3://seu-bucket/raw-data/
```

### Tutorial 2: Mapeamento (Chr22 humano)

```bash
# 1. Baixar chr22 (pequeno)
wget ftp://ftp.ncbi.nlm.nih.gov/genomes/H_sapiens/Assembled_chromosomes/seq/hs_ref_GRCh37.p13_chr22.fa.gz
gunzip hs_ref_GRCh37.p13_chr22.fa.gz

# 2. Baixar reads pequenos
fasterq-dump SRR1234567 -q

# 3. Index
bwa index hs_ref_GRCh37.p13_chr22.fa

# 4. Alinhar
bwa mem hs_ref_GRCh37.p13_chr22.fa SRR1234567.fastq | samtools sort -o saida.bam
samtools index saida.bam

# 5. Stats
samtools flagstat saida.bam
```

### Tutorial 3: Análise Gencode (Anotação)

```bash
# 1. Baixar arquivo pequeno (apenas genes)
wget ftp://ftp.ebi.ac.uk/pub/databases/gencode/Gencode_human/release_45/gencode.v45.basic.annotation.gff3.gz
gunzip gencode.v45.basic.annotation.gff3.gz

# 2. Python: análise estrutura
python3 << 'EOF'
import pandas as pd

# Ler GFF (simples)
gff = pd.read_csv("gencode.v45.basic.annotation.gff3", sep="\t", comment="#", header=None)
gff.columns = ['chr', 'source', 'type', 'start', 'end', 'score', 'strand', 'frame', 'attributes']

# Genes
genes = gff[gff['type'] == 'gene']
print(f"Total genes: {len(genes)}")
print(f"Genes por chromosome:\n{genes['chr'].value_counts()}")

# Exons
exons = gff[gff['type'] == 'exon']
print(f"Total exons: {len(exons)}")
EOF

# 3. BEDTools: merge exons
grep "exon" gencode.v45.basic.annotation.gff3 | cut -f1,4,5 | bedtools merge | wc -l
```

---

## ACESSO VIA AWS OPEN DATA

**Muitos datasets estão hospedados em S3 gratuitamente:**

```bash
# Datasets disponíveis (public buckets)
# Listar:
aws s3 ls --no-sign-request | grep -i genome

# Exemplos:
aws s3 ls s3://1000genomes --no-sign-request
aws s3 ls s3://aws-genomics-datasets --no-sign-request
```

**Vantagem:** Sem cobrar egress (transfer para EC2 é grátis)

---

## ESTRATÉGIA DE DOWNLOAD EFICIENTE

### Para EC2 (recomendado)
```bash
# 1. SSH na EC2
ssh -i ~/.ssh/minha-chave.pem ec2-user@IP

# 2. Baixar direto (não no local)
cd ~/dados
wget ftp://...  # ou aws s3 cp

# 3. Processar na EC2
bwa mem ref.fasta reads.fastq | samtools sort -o saida.bam

# 4. Upload resultado (não bruto)
aws s3 cp saida.bam s3://seu-bucket/results/

# 5. Deletar arquivo bruto para economizar storage
rm reads.fastq ref.fasta
```

### Para Laptop Local
```bash
# 1. Baixar arquivo pequeno APENAS
curl https://... -o arquivo.fastq  # < 500 MB

# 2. Processar
fastqc arquivo.fastq

# 3. Upload em S3 depois
aws s3 cp resultado.html s3://seu-bucket/results/
```

---

## QUICK START: 5 MINUTOS

**Dentro da EC2:**

```bash
# 1. Instalar tudo (2 min)
sudo yum update -y && sudo yum install -y samtools fastqc wget sra-tools
pip3 install --user biopython

# 2. Baixar dataset (1 min)
fasterq-dump SRR2584857 -q

# 3. Análise (2 min)
python3 << 'EOF'
from Bio import SeqIO
reads = list(SeqIO.parse("SRR2584857.fastq", "fastq"))
print(f"✓ {len(reads)} reads processados!")
EOF

# 4. Upload (1 min)
aws s3 cp SRR2584857.fastq s3://seu-bucket/raw-data/
echo "✓ Pronto!"
```

---
