## Azure Configuration

### Login

```bash
az login
az login --use-device-code
az account list --output table
az account show --output table
az account set --subscription "<SUBSCRIPTION_ID>"
```

### Lab variables

```bash
export AZ_RESOURCE_GROUP="rg-bioinformatica-lab"
export AZ_LOCATION="brazilsouth"
export AZ_VM_NAME="vm-bio-lab"
export AZ_STORAGE_ACCOUNT="stbioseunome2026"
export AZ_CONTAINER="raw-data"
```

### Save and load variables

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

---

## Resource Group

```bash
az group create --name $AZ_RESOURCE_GROUP --location $AZ_LOCATION
az group show --name $AZ_RESOURCE_GROUP --output table
az resource list --resource-group $AZ_RESOURCE_GROUP --output table
```

---

## Storage - Blob

### Create Storage Account

```bash
az storage account create \
  --name $AZ_STORAGE_ACCOUNT \
  --resource-group $AZ_RESOURCE_GROUP \
  --location $AZ_LOCATION \
  --sku Standard_LRS \
  --kind StorageV2
```

### Confirm Storage Account

```bash
az storage account show \
  --resource-group $AZ_RESOURCE_GROUP \
  --name $AZ_STORAGE_ACCOUNT \
  --query "{name:name, location:primaryLocation, status:statusOfPrimary}" \
  --output table
```

### Fix wrong Storage Account variable

```bash
az storage account list \
  --resource-group $AZ_RESOURCE_GROUP \
  --query "[].name" \
  --output table

export AZ_STORAGE_ACCOUNT="NAME_FROM_THE_LIST"
nano ~/azure-bioinformatica/credentials/.env
source ~/azure-bioinformatica/credentials/.env
```

### Create container

```bash
az storage container create \
  --name $AZ_CONTAINER \
  --account-name $AZ_STORAGE_ACCOUNT \
  --auth-mode login
```

### Assign upload permission

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

### Upload file

```bash
az storage blob upload \
  --account-name $AZ_STORAGE_ACCOUNT \
  --container-name $AZ_CONTAINER \
  --name raw-data/file.txt \
  --file file.txt \
  --auth-mode login \
  --overwrite
```

### Upload directory with Azure CLI

```bash
az storage blob upload-batch \
  --account-name $AZ_STORAGE_ACCOUNT \
  --destination $AZ_CONTAINER \
  --destination-path raw-data \
  --source ~/azure-bioinformatica/data \
  --auth-mode login \
  --overwrite
```

### Key-mode fallback for labs

```bash
az storage blob upload-batch \
  --account-name $AZ_STORAGE_ACCOUNT \
  --destination $AZ_CONTAINER \
  --destination-path raw-data \
  --source ~/azure-bioinformatica/data \
  --auth-mode key \
  --overwrite
```

### List blobs

```bash
az storage blob list \
  --account-name $AZ_STORAGE_ACCOUNT \
  --container-name $AZ_CONTAINER \
  --prefix results/ \
  --output table \
  --auth-mode login
```

### Download file

```bash
az storage blob download \
  --account-name $AZ_STORAGE_ACCOUNT \
  --container-name $AZ_CONTAINER \
  --name results/resultado.json \
  --file resultado.json \
  --auth-mode login
```

---

## Virtual Machines

### Create SSH key

```bash
mkdir -p ~/azure-bioinformatica/keys
ssh-keygen -t rsa -b 4096 -f ~/azure-bioinformatica/keys/azure_bio_rsa -C "azure-bioinformatica"
chmod 400 ~/azure-bioinformatica/keys/azure_bio_rsa
```

### Create VM

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

### Open SSH port only

```bash
az vm open-port \
  --resource-group $AZ_RESOURCE_GROUP \
  --name $AZ_VM_NAME \
  --port 22
```

Use SSH tunnel for JupyterLab instead of opening port 8888.

### Get public IP

```bash
export AZ_VM_IP=$(az vm show \
  --resource-group $AZ_RESOURCE_GROUP \
  --name $AZ_VM_NAME \
  --show-details \
  --query publicIps \
  --output tsv)

echo $AZ_VM_IP
```

### SSH

```bash
ssh -i ~/azure-bioinformatica/keys/azure_bio_rsa azureuser@$AZ_VM_IP
```

### Stop, confirm, start

```bash
az vm deallocate \
  --resource-group $AZ_RESOURCE_GROUP \
  --name $AZ_VM_NAME

az vm get-instance-view \
  --resource-group $AZ_RESOURCE_GROUP \
  --name $AZ_VM_NAME \
  --query instanceView.statuses[].displayStatus \
  --output table

az vm start \
  --resource-group $AZ_RESOURCE_GROUP \
  --name $AZ_VM_NAME
```

Expected stopped state:

```text
VM deallocated
```

---

## Bioinformatics

### Install on VM

```bash
sudo apt update
sudo apt install -y build-essential python3-dev python3-pip python3-venv git wget unzip curl
sudo apt install -y samtools bwa fastqc bedtools sra-toolkit sqlite3
```

### Python environment

```bash
python3 -m venv ~/bio_env
source ~/bio_env/bin/activate
pip install --upgrade pip
pip install biopython pandas numpy matplotlib seaborn scipy scikit-learn azure-storage-blob jupyter jupyterlab
```

---

## JupyterLab

### Local SSH tunnel

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

### Inside the VM

```bash
source ~/bio_env/bin/activate
jupyter lab --ip=127.0.0.1 --port=8888 --no-browser
```

Open locally:

```text
http://127.0.0.1:8888/lab?token=<your_token>
```

---

## Configure Azure CLI inside the VM

```bash
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
az login --use-device-code

mkdir -p ~/azure-bioinformatica/credentials

cat > ~/azure-bioinformatica/credentials/.env << 'EOF'
AZ_RESOURCE_GROUP=rg-bioinformatica-lab
AZ_LOCATION=brazilsouth
AZ_VM_NAME=vm-bio-lab
AZ_STORAGE_ACCOUNT=stbioseunome2026
AZ_CONTAINER=raw-data
EOF

source ~/azure-bioinformatica/credentials/.env
```

---

## FASTQ Pipeline

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
```

Upload from VM:

```bash
source ~/azure-bioinformatica/credentials/.env

az storage blob upload \
  --account-name $AZ_STORAGE_ACCOUNT \
  --container-name $AZ_CONTAINER \
  --name results/resultado.json \
  --file resultado.json \
  --auth-mode login \
  --overwrite
```

Copy to local and upload:

```bash
scp -i ~/azure-bioinformatica/keys/azure_bio_rsa \
  azureuser@$AZ_VM_IP:/home/azureuser/datasets/resultado.json \
  ~/azure-bioinformatica/data/resultado.json
```

---

## SQLite

```bash
sqlite3 ~/datasets/resultados.db ".tables"
sqlite3 ~/datasets/resultados.db "SELECT * FROM fastq_resultados;"
```

View in JupyterLab:

```python
import sqlite3
import pandas as pd

conn = sqlite3.connect("/home/azureuser/datasets/resultados.db")
pd.read_sql_query("SELECT * FROM fastq_resultados", conn)
```

Upload database:

```bash
az storage blob upload \
  --account-name $AZ_STORAGE_ACCOUNT \
  --container-name $AZ_CONTAINER \
  --name results/resultados.db \
  --file ~/datasets/resultados.db \
  --auth-mode login \
  --overwrite
```

---

## Troubleshooting

### `az: command not found`

You are likely inside the VM and Azure CLI is not installed there. Either run Azure commands locally or install Azure CLI inside the VM.

### `argument --account-name: expected one argument`

Your variable is empty.

```bash
source ~/azure-bioinformatica/credentials/.env
echo $AZ_STORAGE_ACCOUNT
```

### Failed to resolve blob endpoint

```bash
az storage account show \
  --resource-group $AZ_RESOURCE_GROUP \
  --name $AZ_STORAGE_ACCOUNT \
  --output table
```

If the account does not exist, create it or fix `.env`.

### Blob already exists

Add `--overwrite`.

### AuthorizationPermissionMismatch

Assign `Storage Blob Data Contributor` and wait for propagation.
