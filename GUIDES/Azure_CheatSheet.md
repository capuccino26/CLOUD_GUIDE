## AZURE CONFIGURATION

### Login e Contexto
```bash
az login
az login --use-device-code

az account list --output table
az account show --output table
az account set --subscription "<SUBSCRIPTION_ID>"
```

### Variáveis de Laboratório
```bash
export AZ_RESOURCE_GROUP="rg-bioinformatica-lab"
export AZ_LOCATION="brazilsouth"
export AZ_VM_NAME="vm-bio-lab"
export AZ_STORAGE_ACCOUNT="stbio$RANDOM$RANDOM"
export AZ_CONTAINER="raw-data"
```

### Criar Resource Group
```bash
az group create \
  --name $AZ_RESOURCE_GROUP \
  --location $AZ_LOCATION

az group list --output table
az group show --name $AZ_RESOURCE_GROUP --output table
```

---

## STORAGE - BLOB

### Criar Storage Account
```bash
az storage account create \
  --name $AZ_STORAGE_ACCOUNT \
  --resource-group $AZ_RESOURCE_GROUP \
  --location $AZ_LOCATION \
  --sku Standard_LRS \
  --kind StorageV2
```

### Criar Container
```bash
az storage container create \
  --name $AZ_CONTAINER \
  --account-name $AZ_STORAGE_ACCOUNT \
  --auth-mode login
```

### Upload de Arquivo
```bash
az storage blob upload \
  --account-name $AZ_STORAGE_ACCOUNT \
  --container-name $AZ_CONTAINER \
  --name raw-data/arquivo.fastq \
  --file arquivo.fastq \
  --auth-mode login
```

### Upload de Diretório com AzCopy
```bash
azcopy login

azcopy copy "~/dados/*" \
  "https://$AZ_STORAGE_ACCOUNT.blob.core.windows.net/$AZ_CONTAINER/raw-data/" \
  --recursive=true
```

### Download de Arquivo
```bash
az storage blob download \
  --account-name $AZ_STORAGE_ACCOUNT \
  --container-name $AZ_CONTAINER \
  --name results/resultado.json \
  --file resultado.json \
  --auth-mode login
```

### Listar Blobs
```bash
az storage blob list \
  --account-name $AZ_STORAGE_ACCOUNT \
  --container-name $AZ_CONTAINER \
  --output table \
  --auth-mode login

az storage blob list \
  --account-name $AZ_STORAGE_ACCOUNT \
  --container-name $AZ_CONTAINER \
  --prefix results/ \
  --output table \
  --auth-mode login
```

### Deletar Blobs
```bash
az storage blob delete \
  --account-name $AZ_STORAGE_ACCOUNT \
  --container-name $AZ_CONTAINER \
  --name raw-data/arquivo.fastq \
  --auth-mode login

az storage blob delete-batch \
  --account-name $AZ_STORAGE_ACCOUNT \
  --source $AZ_CONTAINER \
  --pattern "tmp/*" \
  --auth-mode login
```

---

## VIRTUAL MACHINES

### Criar Chave SSH
```bash
mkdir -p ~/azure-bioinformatica/keys

ssh-keygen -t rsa -b 4096 \
  -f ~/azure-bioinformatica/keys/azure_bio_rsa \
  -C "azure-bioinformatica"

chmod 400 ~/azure-bioinformatica/keys/azure_bio_rsa
```

### Criar VM Linux
```bash
az vm create \
  --resource-group $AZ_RESOURCE_GROUP \
  --name $AZ_VM_NAME \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --admin-username azureuser \
  --ssh-key-values ~/azure-bioinformatica/keys/azure_bio_rsa.pub \
  --public-ip-sku Standard
```

### Abrir Porta
```bash
az vm open-port \
  --resource-group $AZ_RESOURCE_GROUP \
  --name $AZ_VM_NAME \
  --port 22

az vm open-port \
  --resource-group $AZ_RESOURCE_GROUP \
  --name $AZ_VM_NAME \
  --port 8888
```

### Obter IP Público
```bash
az vm show \
  --resource-group $AZ_RESOURCE_GROUP \
  --name $AZ_VM_NAME \
  --show-details \
  --query publicIps \
  --output tsv
```

### Conectar via SSH
```bash
ssh -i ~/azure-bioinformatica/keys/azure_bio_rsa azureuser@<SEUIP>

ssh -i ~/azure-bioinformatica/keys/azure_bio_rsa \
  -L 8888:localhost:8888 \
  azureuser@<SEUIP>
```

### Parar, Iniciar e Remover VM
```bash
# Parar cobrança de compute
az vm deallocate \
  --resource-group $AZ_RESOURCE_GROUP \
  --name $AZ_VM_NAME

# Iniciar novamente
az vm start \
  --resource-group $AZ_RESOURCE_GROUP \
  --name $AZ_VM_NAME

# Remover VM
az vm delete \
  --resource-group $AZ_RESOURCE_GROUP \
  --name $AZ_VM_NAME \
  --yes
```

---

## BIOINFORMÁTICA

### Instalar Dependências na VM
```bash
sudo apt update
sudo apt install -y build-essential python3-dev python3-pip python3-venv git wget unzip curl
sudo apt install -y samtools bwa fastqc bedtools sra-toolkit
```

### Ambiente Python
```bash
python3 -m venv ~/bio_env
source ~/bio_env/bin/activate

pip install --upgrade pip
pip install biopython pandas numpy matplotlib seaborn scipy scikit-learn azure-storage-blob
```

### Script Python: Upload em Blob Storage
```python
from azure.storage.blob import BlobServiceClient

connection_string = "DefaultEndpointsProtocol=https;AccountName=..."
service = BlobServiceClient.from_connection_string(connection_string)
client = service.get_blob_client(
    container="raw-data",
    blob="results/resultado.json"
)

with open("resultado.json", "rb") as data:
    client.upload_blob(data, overwrite=True)

print("Upload concluído")
```

### Script Python: Estatísticas FASTQ
```python
from Bio import SeqIO
import numpy as np

tamanhos = []
qualidades = []

for record in SeqIO.parse("arquivo.fastq", "fastq"):
    tamanhos.append(len(record.seq))
    qualidades.append(np.mean(record.letter_annotations["phred_quality"]))

print(f"Reads: {len(tamanhos)}")
print(f"Tamanho médio: {np.mean(tamanhos):.0f}")
print(f"Qualidade média: {np.mean(qualidades):.2f}")
```

---

## JUPYTER

### Instalar e Rodar JupyterLab
```bash
source ~/bio_env/bin/activate
pip install jupyter jupyterlab

jupyter lab --ip=127.0.0.1 --port=8888 --no-browser
```

### Túnel SSH Local
```bash
ssh -i ~/azure-bioinformatica/keys/azure_bio_rsa \
  -L 8888:localhost:8888 \
  azureuser@<SEUIP>
```

### Ver Servidores Jupyter
```bash
jupyter server list
```

---

## PIPELINES

### Pipeline Básico: FASTQ para Resultado JSON
```bash
mkdir -p ~/datasets
cd ~/datasets

cat > teste.fastq << 'EOF'
@read1
ATCGATCGATCGATCG
+
IIIIIIIIIIIIIIII
@read2
GCTAGCTAGCTAGCTA
+
IIIIIIIIIIIIIIII
EOF

python3 ~/analisar_fastq.py teste.fastq > resultado.json

az storage blob upload \
  --account-name $AZ_STORAGE_ACCOUNT \
  --container-name $AZ_CONTAINER \
  --name results/resultado.json \
  --file resultado.json \
  --auth-mode login
```

### Pipeline FASTQ para BAM
```bash
bwa index ref.fasta

bwa mem ref.fasta reads.fastq \
  | samtools view -b - \
  | samtools sort -o saida.bam

samtools index saida.bam
samtools flagstat saida.bam > stats.txt

az storage blob upload \
  --account-name $AZ_STORAGE_ACCOUNT \
  --container-name $AZ_CONTAINER \
  --name results/saida.bam \
  --file saida.bam \
  --auth-mode login
```

---

## MONITORAR CUSTOS

### Listar Recursos do Grupo
```bash
az resource list \
  --resource-group $AZ_RESOURCE_GROUP \
  --output table
```

### Ver Uso de VM
```bash
az vm list \
  --resource-group $AZ_RESOURCE_GROUP \
  --show-details \
  --output table
```

### Criar Budget pelo Portal
```text
Cost Management + Billing
Cost Management
Budgets
Add
Scope: assinatura ou resource group
Alertas: 50%, 80%, 100%
```

### Limpeza Completa do Laboratório
```bash
az group delete \
  --name $AZ_RESOURCE_GROUP \
  --yes
```

---

## TROUBLESHOOTING

### Erro: "Please run az login"
```bash
az login
az account show --output table
```

### Erro: "The subscription is not registered"
```bash
az provider register --namespace Microsoft.Compute
az provider register --namespace Microsoft.Storage
az provider register --namespace Microsoft.Network
```

### Erro: Storage Account Name Unavailable
```bash
export AZ_STORAGE_ACCOUNT="stbio$RANDOM$RANDOM"
echo $AZ_STORAGE_ACCOUNT
```

### Erro: AuthorizationPermissionMismatch no Blob
```bash
# Confirme login
az account show --output table

# Use --auth-mode login
az storage blob list \
  --account-name $AZ_STORAGE_ACCOUNT \
  --container-name $AZ_CONTAINER \
  --auth-mode login
```

### Erro: SSH Connection Timed Out
```bash
# Ver IP público
az vm show \
  --resource-group $AZ_RESOURCE_GROUP \
  --name $AZ_VM_NAME \
  --show-details \
  --query publicIps \
  --output tsv

# Abrir porta 22
az vm open-port \
  --resource-group $AZ_RESOURCE_GROUP \
  --name $AZ_VM_NAME \
  --port 22
```
