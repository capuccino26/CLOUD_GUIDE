## AWS CONFIGURATION

### Configurar Credenciais (primeira vez)
```bash
aws configure
# Após completar, verifica arquivo salvo:
cat ~/.aws/credentials
cat ~/.aws/config
```

### Definir Região para Comando Específico
```bash
aws ec2 describe-instances --region sa-east-1
aws s3 ls --region sa-east-1
```

### Mudar Região Padrão
```bash
export AWS_DEFAULT_REGION=sa-east-1
# Ou editar ~/.aws/config
```

---

## S3 - STORAGE

### Listar Buckets
```bash
aws s3 ls
aws s3api list-buckets --query 'Buckets[].Name'
```

### Criar Bucket
```bash
aws s3 mb s3://seu-bucket-nome --region sa-east-1
```

### Upload Arquivo Único
```bash
# Arquivo pequeno
aws s3 cp arquivo.fastq s3://seu-bucket/raw-data/

# Com progresso
aws s3 cp arquivo.bam s3://seu-bucket/results/ --progress

# Com ACL privado
aws s3 cp arquivo.txt s3://seu-bucket/ --sse AES256
```

### Upload Diretório Inteiro
```bash
# Sincronizar (mais eficiente - apenas o novo)
aws s3 sync ~/dados-locais/ s3://seu-bucket/raw-data/

# Sincronizar com exclusões
aws s3 sync ~/dados/ s3://seu-bucket/ --delete

# Apenas ver o que seria feito (dry-run)
aws s3 sync ~/dados/ s3://seu-bucket/ --dryrun
```

### Download de S3
```bash
# Arquivo único
aws s3 cp s3://seu-bucket/results/resultado.bam ~/download/

# Diretório
aws s3 sync s3://seu-bucket/processed/ ~/dados-processados/

# Com filtro
aws s3 sync s3://seu-bucket/ . --exclude "*" --include "*.bam"
```

### Listar Conteúdo Bucket
```bash
# Básico
aws s3 ls s3://seu-bucket/

# Com tamanhos (recursivo)
aws s3 ls s3://seu-bucket/ --recursive --summarize --human-readable

# Apenas arquivos .fastq
aws s3 ls s3://seu-bucket/ --recursive | grep "\.fastq"
```

### Deletar Arquivos S3
```bash
# Deletar arquivo único
aws s3 rm s3://seu-bucket/arquivo.txt

# Deletar diretório inteiro
aws s3 rm s3://seu-bucket/pasta/ --recursive

# Com confirmação
aws s3 rm s3://seu-bucket/grande.bam --recursive --debug
```

### Gerenciar Permissões S3
```bash
# Tornar público (NÃO recomendado!)
aws s3api put-object-acl --bucket seu-bucket --key arquivo.txt --acl public-read

# Tornar privado
aws s3api put-object-acl --bucket seu-bucket --key arquivo.txt --acl private

# Ver ACL
aws s3api get-object-acl --bucket seu-bucket --key arquivo.txt
```

---

## EC2 - MÁQUINAS VIRTUAIS

### Listar Instâncias
```bash
# Todas
aws ec2 describe-instances --region sa-east-1

# Apenas nomes e IPs
aws ec2 describe-instances --query 'Reservations[].Instances[].[InstanceId, InstanceType, State.Name, PublicIpAddress, Tags[?Key==`Name`].Value|[0]]' --output table

# Apenas rodando
aws ec2 describe-instances --filters "Name=instance-state-name,Values=running" --query 'Reservations[].Instances[].InstanceId'
```

### Lançar Instância (via CLI)
```bash
# Versão simples (t2.micro, Amazon Linux 2)
aws ec2 run-instances \
  --image-id ami-0c2a1acae6667e438 \
  --instance-type t2.micro \
  --key-name minha-chave-bio \
  --security-groups default \
  --region sa-east-1

# Versão completa (com subnet, storage, tags)
aws ec2 run-instances \
  --image-id ami-0c2a1acae6667e438 \
  --instance-type t2.micro \
  --key-name minha-chave-bio \
  --security-group-ids sg-0123abcd \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=bioinformatica-lab}]' \
  --block-device-mappings DeviceName=/dev/xvda,Ebs='{VolumeSize=30,VolumeType=gp2,DeleteOnTermination=true}' \
  --region sa-east-1
```

### Conectar via SSH
```bash
# Básico (substitua ID por seu IP)
ssh -i ~/.ssh/minha-chave-bio.pem ec2-user@<SEUIP>

# Sem pedir confirmação
ssh -i ~/.ssh/minha-chave-bio.pem -o StrictHostKeyChecking=no ec2-user@<SEUIP>

# Ubuntu (usuário ubuntu, não ec2-user)
ssh -i ~/.ssh/minha-chave-bio.pem ubuntu@<SEUIP>

# Com port forwarding (Jupyter)
ssh -i ~/.ssh/minha-chave-bio.pem -L 8888:localhost:8888 ec2-user@<SEUIP>
```

### Parar/Iniciar Instâncias
```bash
# Parar (não deleta, apenas pausado)
aws ec2 stop-instances --instance-ids <instanceid> --region sa-east-1

# Iniciar
aws ec2 start-instances --instance-ids <instanceid> --region sa-east-1

# Terminar (DELETA!)
aws ec2 terminate-instances --instance-ids <instanceid> --region sa-east-1

# Monitorar status
aws ec2 describe-instances --instance-ids <instanceid> --query 'Reservations[].Instances[].State'
```

### Gerir Chaves SSH
```bash
# Listar pares de chaves
aws ec2 describe-key-pairs --region sa-east-1

# Deletar par de chaves (local)
rm ~/.ssh/minha-chave.pem

# Deletar par na AWS (pode impedir acesso!)
aws ec2 delete-key-pair --key-name minha-chave-bio --region sa-east-1
```

---

## BIOINFORMÁTICA

### Instalar Dependências (na EC2)
```bash
# Python basics
python3 -m pip install --upgrade pip
pip3 install --user virtualenv

# Bioinformática
pip3 install --user biopython numpy pandas matplotlib seaborn scipy scikit-learn

# AWS SDK
pip3 install --user boto3 awscli

# Análise
pip3 install --user jupyter jupyterlab statsmodels sympy

# Desenvolvimento
pip3 install --user black flake8 pytest
```

### Script Python: Ler FASTQ
```python
from Bio import SeqIO

# Ler arquivo
for record in SeqIO.parse("arquivo.fastq", "fastq"):
    print(f"ID: {record.id}")
    print(f"Sequência: {record.seq}")
    print(f"Qualidade: {record.letter_annotations['phred_quality']}")
    print("---")
```

### Script Python: Contar Reads
```python
from Bio import SeqIO

num_reads = sum(1 for _ in SeqIO.parse("arquivo.fastq", "fastq"))
print(f"Total de reads: {num_reads}")
```

### Script Python: Converter FASTQ para FASTA
```python
from Bio import SeqIO

SeqIO.convert("entrada.fastq", "fastq", "saida.fasta", "fasta")
print("Convertido com sucesso!")
```

### Script Python: Baixar de S3
```python
import boto3

s3 = boto3.client('s3')
s3.download_file('seu-bucket', 'raw-data/arquivo.fastq', '/tmp/local.fastq')
print("Arquivo baixado!")
```

### Script Python: Fazer Upload em S3
```python
import boto3

s3 = boto3.client('s3')
s3.upload_file('/tmp/resultado.bam', 'seu-bucket', 'results/resultado.bam')
print("Upload concluído!")
```

### Script Python: Listar Conteúdo S3
```python
import boto3

s3 = boto3.resource('s3')
bucket = s3.Bucket('seu-bucket')

for obj in bucket.objects.all():
    print(f"{obj.key} ({obj.size} bytes)")
```

### Script Python: Estatísticas FASTQ
```python
from Bio import SeqIO
import numpy as np

tamanhos = []
qualidades = []

for record in SeqIO.parse("arquivo.fastq", "fastq"):
    tamanhos.append(len(record.seq))
    qual = np.mean(record.letter_annotations['phred_quality'])
    qualidades.append(qual)

print(f"Reads: {len(tamanhos)}")
print(f"Tamanho médio: {np.mean(tamanhos):.0f}")
print(f"Qualidade média: {np.mean(qualidades):.2f}")
print(f"Tamanho min/max: {min(tamanhos)}/{max(tamanhos)}")
```

---

## FERRAMENTAS

### Instalar Ferramentas
```bash
# Amazon Linux 2
sudo yum install -y samtools bwa fastqc bedtools

# Ubuntu
sudo apt install -y samtools bwa fastqc bedtools

# Com package manager específico (Miniconda)
conda install -c bioconda samtools bwa fastqc bedtools
```

### Samtools - Manipular BAM/SAM
```bash
# Ver arquivo SAM
samtools view arquivo.sam | head

# Converter SAM para BAM
samtools view -b -S arquivo.sam > arquivo.bam

# Ordenar BAM
samtools sort arquivo.bam -o arquivo.sorted.bam

# Indexar BAM
samtools index arquivo.sorted.bam

# Ver stats
samtools flagstat arquivo.sorted.bam

# Contar reads
samtools view -c arquivo.bam

# Filtrar apenas reads mapeados
samtools view -b -F 4 arquivo.bam > mapeados.bam
```

### BWA - Alinhador
```bash
# Criar índice de referência
bwa index genoma.fasta

# Alinhar reads
bwa mem genoma.fasta reads.fastq > alinhamento.sam

# Versão com mais threads
bwa mem -t 4 genoma.fasta reads.fastq > alinhamento.sam

# SAM + BAM (pipeline completo)
bwa mem genoma.fasta reads.fastq | samtools view -b - | samtools sort -o saida.bam
samtools index saida.bam
```

### FastQC - Controle de Qualidade
```bash
# Análise de um arquivo
fastqc arquivo.fastq

# Análise de múltiplos
fastqc *.fastq

# Output em diretório específico
fastqc -o ~/resultados/ arquivo.fastq

# Gerar HTML somente (sem ZIP)
fastqc -q arquivo.fastq
```

### BEDTools - Operações com Coordenadas
```bash
# Intersecção entre dois BED
bedtools intersect -a arquivo1.bed -b arquivo2.bed

# Merge de intervalos
bedtools merge -i arquivo.bed

# Cobertura
bedtools coverage -a arquivo.bed -b alinhamento.bam
```

---

## PIPELINES

### Pipeline Básico: FASTQ → BAM
```bash
# Simples
bwa mem ref.fasta reads.fastq | samtools view -b - | samtools sort -o saida.bam
samtools index saida.bam

# Com compressão e stats
bwa mem ref.fasta reads.fastq \
  | samtools view -b - \
  | samtools sort -o saida.bam \
  && samtools index saida.bam \
  && samtools flagstat saida.bam > stats.txt
```

### Pipeline: Download SRA → Análise
```bash
# Requisitos: sra-toolkit instalado
ID="SRR12345678"

# Download
fasterq-dump $ID

# QC
fastqc ${ID}.fastq

# Mapeamento (com ref pequena)
bwa mem ref.fasta ${ID}.fastq | samtools sort -o ${ID}.bam
samtools index ${ID}.bam

# Stats
samtools flagstat ${ID}.bam

# Upload resultado
aws s3 cp ${ID}.bam s3://seu-bucket/results/
```

### Processar Múltiplos Arquivos
```bash
# Loop simples
for arquivo in *.fastq; do
    echo "Processando $arquivo"
    fastqc $arquivo
done

# Com paralelo (mais rápido)
ls *.fastq | parallel fastqc {}

# Com pipes
cat lista_arquivos.txt | while read arquivo; do
    bwa mem ref.fasta $arquivo | samtools view -b - > ${arquivo%.fastq}.bam
done
```

## Projetos Práticos

### **Projeto 1: "Hello Bioinformatics" (Semana 1-2)**

Objetivo: Verificar se ambiente está OK

```bash
# No EC2:
# 1. Criar pasta de trabalho
mkdir -p ~/bioinformatics && cd ~/bioinformatics

# 2. Teste Python + Biopython
python3 << 'EOF'
from Bio import SeqIO
from Bio.Seq import Seq

seq = Seq("ATCGATCGATCG")
print(f"Sequência: {seq}")
print(f"Complemento: {seq.complement()}")
EOF

# 3. Upload no S3
aws s3 cp . s3://seu-bucket-bio/teste/ --recursive
```

---

### **Projeto 2: Pipeline FASTQ (Semana 3-4)**

**Dados:** Usar dataset público gratuito

```bash
# 1. Download de dados (SRA Toolkit)
sudo apt install sra-toolkit -y
fasterq-dump SRR12345678  # substitua pelo ID real

# 2. QC com FastQC
fastqc *.fastq

# 3. Mapeamento com BWA
# Baixar genoma referência (humano = ~3 GB, use versão small)
wget ftp://ftp.ncbi.nlm.nih.gov/refseq/H_sapiens/annotation_releases/109.20210226/GCF_000001405.39_GRCh38.p13/GCF_000001405.39_GRCh38.p13_genomic.fna.gz

# 4. Indexar e mapear
bwa index -a bwtsw referencia.fasta
bwa mem referencia.fasta reads.fastq > mapeado.sam

# 5. Converter e comprimir
samtools view -b -S mapeado.sam | samtools sort -o mapeado.bam
samtools index mapeado.bam

# 6. Upload resultados
aws s3 cp mapeado.bam s3://seu-bucket-bio/resultados/
```

---

### **Projeto 3: Análise Estatística com Pandas (Semana 5)**

```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt

# Simular dados de sequenciamento
dados = pd.DataFrame({
    'amostra': ['A', 'B', 'C'],
    'reads_mapeados': [1000000, 950000, 1100000],
    'cobertura': [50, 47, 55]
})

# Análise
print(dados.describe())
plt.plot(dados['amostra'], dados['cobertura'])
plt.savefig('cobertura.png')

# Salvar em S3
import boto3
s3 = boto3.client('s3')
s3.upload_file('cobertura.png', 'seu-bucket-bio', 'resultados/grafico.png')
```

---

### **Projeto 4: Automação com Lambda (Semana 6)**

**Função Lambda: Converter FASTQ para FASTA automaticamente**

```python
import boto3
import json
from Bio import SeqIO
from io import StringIO

s3 = boto3.client('s3')

def lambda_handler(event, context):
    # Quando arquivo chega no S3
    bucket = event['Records'][0]['s3']['bucket']['name']
    key = event['Records'][0]['s3']['object']['key']
    
    # Download
    obj = s3.get_object(Bucket=bucket, Key=key)
    fastq_content = obj['Body'].read().decode('utf-8')
    
    # Converter
    fasta_content = StringIO()
    for record in SeqIO.parse(StringIO(fastq_content), "fastq"):
        fasta_content.write(f">{record.id}\n{str(record.seq)}\n")
    
    # Upload resultado
    output_key = key.replace('.fastq', '.fasta')
    s3.put_object(Bucket=bucket, Key=output_key, 
                  Body=fasta_content.getvalue())
    
    return {'statusCode': 200, 'body': 'Convertido com sucesso'}
```

---

## MONITORAR CUSTOS

### Ver Custos Diários
```bash
aws ce get-cost-and-usage \
  --time-period Start=2024-04-01,End=2024-04-30 \
  --granularity DAILY \
  --metrics "UnblendedCost" \
  --query 'ResultsByTime[*].[TimePeriod.Start,Total.UnblendedCost.Amount]' \
  --output table
```

### Ver Custos por Serviço
```bash
aws ce get-cost-and-usage \
  --time-period Start=2024-04-01,End=2024-04-30 \
  --granularity MONTHLY \
  --metrics "UnblendedCost" \
  --group-by Type=DIMENSION,Key=SERVICE \
  --output table
```

### Ver Custos EC2 Específico
```bash
aws ce get-cost-and-usage \
  --time-period Start=2024-04-01,End=2024-04-30 \
  --granularity DAILY \
  --metrics "UnblendedCost" \
  --filter file://filter.json \
  --group-by Type=DIMENSION,Key=PURCHASE_TYPE
```

Filter.json:
```json
{
  "Dimensions": {
    "Key": "SERVICE",
    "Values": ["Amazon EC2"]
  }
}
```

---

## TROUBLESHOOTING

### Erro: "Unable to locate credentials"
```bash
# Solução: Configurar credenciais
aws configure

# Ou especificar manualmente
export AWS_ACCESS_KEY_ID=sua_key
export AWS_SECRET_ACCESS_KEY=sua_secret
aws s3 ls
```

### Erro: "AccessDenied" em S3
```bash
# Problema: IAM sem permissões S3
# Solução: Adicionar permissão S3FullAccess ao usuário

# Verificar permissões
aws s3 ls
aws s3 ls s3://seu-bucket/

# Se não funcionar, criar nova chave/usuário
```

### Erro: "An error occurred (AuthFailure)" em EC2
```bash
# Problema: Credenciais expiradas
# Solução: Regenerar chaves SSH e access keys

# Listar instâncias para diagnosticar
aws ec2 describe-instances --region sa-east-1
```

### SSH: "Permission denied (publickey)"
```bash
# Problema: Permissões da chave .pem erradas
# Solução:
chmod 400 ~/.ssh/minha-chave.pem

# Também pode ser credencial errada:
ssh -i ~/.ssh/minha-chave.pem -v ec2-user@IP  # verbose
```

### EC2: Muito lento?
```bash
# t2.micro tem créditos limitados (20 créditos/hora, max 144/dia)
# Solução: Desligar e reiiciar para resetar créditos

aws ec2 stop-instances --instance-ids i-xxx
# Aguarde e:
aws ec2 start-instances --instance-ids i-xxx
```