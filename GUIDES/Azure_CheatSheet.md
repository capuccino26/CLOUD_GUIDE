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
export AZ_STORAGE_ACCOUNT="stbioseunome2026"
export AZ_CONTAINER="raw-data"
```

### Salvar e Carregar Variáveis
```bash
mkdir -p ~/azure-bioinformatica/credentials

cat > ~/azure-bioinformatica/credentials/.env << EOF
AZ_RESOURCE_GROUP=$AZ_RESOURCE_GROUP
AZ_LOCATION=$AZ_LOCATION
AZ_VM_NAME=$AZ_VM_NAME
AZ_STORAGE_ACCOUNT=$AZ_STORAGE_ACCOUNT
AZ_CONTAINER=$AZ_CONTAINER
EOF

source ~/azure-bioinformatica/credentials/.env
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

Comandos desta seção devem ser executados no notebook/terminal local. Se o prompt for `azureuser@vm-bio-lab`, você está dentro da VM; rode `exit` para voltar ao terminal local ou instale Azure CLI na VM antes.

### Criar Storage Account
```bash
az storage account create \
  --name $AZ_STORAGE_ACCOUNT \
  --resource-group $AZ_RESOURCE_GROUP \
  --location $AZ_LOCATION \
  --sku Standard_LRS \
  --kind StorageV2
```

### Confirmar Storage Account
```bash
source ~/azure-bioinformatica/credentials/.env
echo $AZ_STORAGE_ACCOUNT

az storage account show \
  --resource-group $AZ_RESOURCE_GROUP \
  --name $AZ_STORAGE_ACCOUNT \
  --query "{name:name, location:primaryLocation, status:statusOfPrimary}" \
  --output table
```

### Corrigir Nome da Storage Account
```bash
az storage account list \
  --resource-group $AZ_RESOURCE_GROUP \
  --query "[].name" \
  --output table

export AZ_STORAGE_ACCOUNT="NOME_QUE_APARECEU_NA_LISTA"
nano ~/azure-bioinformatica/credentials/.env
source ~/azure-bioinformatica/credentials/.env
```

### Criar Container
```bash
az storage container create \
  --name $AZ_CONTAINER \
  --account-name $AZ_STORAGE_ACCOUNT \
  --auth-mode login
```

### Permissao para Upload com Login
```bash
export AZ_USER_OBJECT_ID=$(az ad signed-in-user show --query id --output tsv)

export AZ_STORAGE_SCOPE=$(az storage account show \
  --resource-group $AZ_RESOURCE_GROUP \
  --name $AZ_STORAGE_ACCOUNT \
  --query id \
  --output tsv)

az role assignment create \
  --assignee $AZ_USER_OBJECT_ID \
  --role "Storage Blob Data Contributor" \
  --scope $AZ_STORAGE_SCOPE
```

Aguarde 1 a 5 minutos para a permissão propagar.

### Upload de Arquivo
```bash
az storage blob upload \
  --account-name $AZ_STORAGE_ACCOUNT \
  --container-name $AZ_CONTAINER \
  --name raw-data/arquivo.fastq \
  --file arquivo.fastq \
  --auth-mode login
```

### Upload de Diretório com Azure CLI
```bash
az storage blob upload-batch \
  --account-name $AZ_STORAGE_ACCOUNT \
  --destination $AZ_CONTAINER \
  --destination-path raw-data \
  --source ~/dados \
  --auth-mode login \
  --overwrite
```

### Upload de Diretório com Chave
```bash
az storage blob upload-batch \
  --account-name $AZ_STORAGE_ACCOUNT \
  --destination $AZ_CONTAINER \
  --destination-path raw-data \
  --source ~/dados \
  --auth-mode key \
  --overwrite
```

### AzCopy Opcional
```bash
# AzCopy pode exigir conta corporativa/escolar no login.
# Para conta pessoal Microsoft, prefira az storage blob upload-batch.

azcopy --version

cd /tmp
wget https://aka.ms/downloadazcopy-v10-linux -O azcopy.tar.gz
tar -xzf azcopy.tar.gz
sudo cp ./azcopy_linux_amd64_*/azcopy /usr/local/bin/
azcopy --version

azcopy login
azcopy copy "$HOME/dados/*" \
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

### Abrir Porta SSH
```bash
az vm open-port \
  --resource-group $AZ_RESOURCE_GROUP \
  --name $AZ_VM_NAME \
  --port 22
```

Para JupyterLab, use túnel SSH em vez de abrir a porta 8888.

### Obter IP Público da VM
```bash
export AZ_VM_IP=$(az vm show \
  --resource-group $AZ_RESOURCE_GROUP \
  --name $AZ_VM_NAME \
  --show-details \
  --query publicIps \
  --output tsv)

echo $AZ_VM_IP
```

Não use `ip a`, `hostname -I` ou `ifconfig` para preencher `AZ_VM_IP`. Esses comandos mostram IPs do notebook local, não o IP público da VM Azure.

### Conectar via SSH
```bash
ssh-keyscan -H $AZ_VM_IP >> ~/.ssh/known_hosts 2>/dev/null

ssh -i ~/azure-bioinformatica/keys/azure_bio_rsa azureuser@$AZ_VM_IP

ssh -i ~/azure-bioinformatica/keys/azure_bio_rsa \
  -L 8888:localhost:8888 \
  azureuser@$AZ_VM_IP
```

### IP Errado para SSH
```bash
# Estes IPs normalmente sao locais/VPN, nao sao o IP publico da VM:
# 192.168.x.x
# 10.x.x.x
# 172.16.x.x ate 172.31.x.x
# 100.x.x.x (Tailscale)

# Use sempre:
az vm show \
  --resource-group $AZ_RESOURCE_GROUP \
  --name $AZ_VM_NAME \
  --show-details \
  --query publicIps \
  --output tsv
```

### Parar, Confirmar e Iniciar VM
```bash
source ~/azure-bioinformatica/credentials/.env

# Parar cobrança de compute e manter VM para uso futuro
az vm deallocate \
  --resource-group $AZ_RESOURCE_GROUP \
  --name $AZ_VM_NAME

# Confirmar estado esperado: VM deallocated
az vm get-instance-view \
  --resource-group $AZ_RESOURCE_GROUP \
  --name $AZ_VM_NAME \
  --query instanceView.statuses[].displayStatus \
  --output table

# Iniciar novamente
az vm start \
  --resource-group $AZ_RESOURCE_GROUP \
  --name $AZ_VM_NAME

# Pegar IP publico novamente apos iniciar
export AZ_VM_IP=$(az vm show \
  --resource-group $AZ_RESOURCE_GROUP \
  --name $AZ_VM_NAME \
  --show-details \
  --query publicIps \
  --output tsv)
```

Com `VM deallocated`, o compute está parado. Discos, Storage Account e alguns recursos de rede continuam existindo para permitir retomar o laboratório depois.

### Remover VM
```bash
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

### Túnel SSH Local para Jupyter
```bash
source ~/azure-bioinformatica/credentials/.env

export AZ_VM_IP=$(az vm show \
  --resource-group $AZ_RESOURCE_GROUP \
  --name $AZ_VM_NAME \
  --show-details \
  --query publicIps \
  --output tsv)

ssh -i ~/azure-bioinformatica/keys/azure_bio_rsa \
  -L 8888:localhost:8888 \
  azureuser@$AZ_VM_IP
```

### Instalar e Rodar JupyterLab na VM
```bash
source ~/bio_env/bin/activate
pip install jupyter jupyterlab

jupyter lab --ip=127.0.0.1 --port=8888 --no-browser
```

Abra no navegador local:

```text
http://127.0.0.1:8888/lab?token=<seu_token>
```

### Configurar Azure CLI Dentro da VM
```bash
# Rode dentro da VM se quiser usar az storage direto dela.
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
az login --use-device-code

mkdir -p ~/azure-bioinformatica/credentials

cat > ~/azure-bioinformatica/credentials/.env << EOF
AZ_RESOURCE_GROUP=rg-bioinformatica-lab
AZ_LOCATION=brazilsouth
AZ_VM_NAME=vm-bio-lab
AZ_STORAGE_ACCOUNT=stbioseunome2026
AZ_CONTAINER=raw-data
EOF

source ~/azure-bioinformatica/credentials/.env
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

# Opção dentro da VM: requer Azure CLI e .env configurados na VM
source ~/azure-bioinformatica/credentials/.env

az storage blob upload \
  --account-name $AZ_STORAGE_ACCOUNT \
  --container-name $AZ_CONTAINER \
  --name results/resultado.json \
  --file resultado.json \
  --auth-mode login \
  --overwrite
```

### Copiar Resultado da VM para o Notebook
```bash
# Rode no terminal local, fora do SSH
source ~/azure-bioinformatica/credentials/.env

export AZ_VM_IP=$(az vm show \
  --resource-group $AZ_RESOURCE_GROUP \
  --name $AZ_VM_NAME \
  --show-details \
  --query publicIps \
  --output tsv)

scp -i ~/azure-bioinformatica/keys/azure_bio_rsa \
  azureuser@$AZ_VM_IP:/home/azureuser/datasets/resultado.json \
  ~/azure-bioinformatica/data/resultado.json

az storage blob upload \
  --account-name $AZ_STORAGE_ACCOUNT \
  --container-name $AZ_CONTAINER \
  --name results/resultado.json \
  --file ~/azure-bioinformatica/data/resultado.json \
  --auth-mode login \
  --overwrite
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

## SQLITE

### Criar Banco na VM
```bash
sudo apt install -y sqlite3

sqlite3 ~/datasets/resultados.db << 'EOF'
CREATE TABLE IF NOT EXISTS fastq_resultados (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  arquivo TEXT,
  total_reads INTEGER,
  tamanho_medio REAL,
  qualidade_media REAL,
  criado_em TEXT DEFAULT CURRENT_TIMESTAMP
);
EOF
```

### Consultar no Terminal
```bash
sqlite3 ~/datasets/resultados.db ".tables"
sqlite3 ~/datasets/resultados.db "SELECT * FROM fastq_resultados;"
```

### Ver no JupyterLab
```python
import sqlite3
import pandas as pd

conn = sqlite3.connect("/home/azureuser/datasets/resultados.db")
df = pd.read_sql_query("SELECT * FROM fastq_resultados", conn)
df
```

### Upload do Banco
```bash
az storage blob upload \
  --account-name $AZ_STORAGE_ACCOUNT \
  --container-name $AZ_CONTAINER \
  --name results/resultados.db \
  --file ~/datasets/resultados.db \
  --auth-mode login \
  --overwrite
```

No Azure Portal, o SQLite aparece como arquivo em `Storage accounts > Containers > raw-data > results/resultados.db`. O portal permite baixar o arquivo, mas não abre as tabelas.

---

## MONITORAR CUSTOS

### Listar Recursos do Grupo
```bash
az resource list \
  --resource-group $AZ_RESOURCE_GROUP \
  --output table
```

### Ver Uso e Estado de VM
```bash
az vm list \
  --resource-group $AZ_RESOURCE_GROUP \
  --show-details \
  --output table

az vm get-instance-view \
  --resource-group $AZ_RESOURCE_GROUP \
  --name $AZ_VM_NAME \
  --query instanceView.statuses[].displayStatus \
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

az storage account create \
  --name $AZ_STORAGE_ACCOUNT \
  --resource-group $AZ_RESOURCE_GROUP \
  --location $AZ_LOCATION \
  --sku Standard_LRS \
  --kind StorageV2
```

### Erro: Failed to resolve blob.core.windows.net
```bash
# 1. Confirmar se a conta existe
az storage account show \
  --resource-group $AZ_RESOURCE_GROUP \
  --name $AZ_STORAGE_ACCOUNT \
  --output table

# 2. Se nao existir, criar a conta antes do container
az storage account create \
  --name $AZ_STORAGE_ACCOUNT \
  --resource-group $AZ_RESOURCE_GROUP \
  --location $AZ_LOCATION \
  --sku Standard_LRS \
  --kind StorageV2

# 3. Se existir, testar DNS/rede local
nslookup ${AZ_STORAGE_ACCOUNT}.blob.core.windows.net
```

### Erro: AuthorizationPermissionMismatch no Blob
```bash
# Confirme login
az account show --output table

# Atribuir permissao de dados ao usuario logado
export AZ_USER_OBJECT_ID=$(az ad signed-in-user show --query id --output tsv)

export AZ_STORAGE_SCOPE=$(az storage account show \
  --resource-group $AZ_RESOURCE_GROUP \
  --name $AZ_STORAGE_ACCOUNT \
  --query id \
  --output tsv)

az role assignment create \
  --assignee $AZ_USER_OBJECT_ID \
  --role "Storage Blob Data Contributor" \
  --scope $AZ_STORAGE_SCOPE

# Aguardar 1 a 5 minutos e testar
az storage blob list \
  --account-name $AZ_STORAGE_ACCOUNT \
  --container-name $AZ_CONTAINER \
  --auth-mode login
```

### Alternativa de Laboratorio: Auth Mode Key
```bash
az storage blob upload \
  --account-name $AZ_STORAGE_ACCOUNT \
  --container-name $AZ_CONTAINER \
  --name raw-data/arquivo.fastq \
  --file arquivo.fastq \
  --auth-mode key
```

### Erro: SSH Connection Refused ou Timed Out
```bash
# Ver estado e IP publico da VM
az vm show \
  --resource-group $AZ_RESOURCE_GROUP \
  --name $AZ_VM_NAME \
  --show-details \
  --query "[powerState, publicIps]" \
  --output table

# Abrir porta 22
az vm open-port \
  --resource-group $AZ_RESOURCE_GROUP \
  --name $AZ_VM_NAME \
  --port 22

# Salvar IP publico correto
export AZ_VM_IP=$(az vm show \
  --resource-group $AZ_RESOURCE_GROUP \
  --name $AZ_VM_NAME \
  --show-details \
  --query publicIps \
  --output tsv)

ssh -i ~/azure-bioinformatica/keys/azure_bio_rsa azureuser@$AZ_VM_IP
```
