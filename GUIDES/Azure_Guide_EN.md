# Azure: Practical Executive Guide

This guide builds a low-cost Azure lab for bioinformatics and data science.

---

## Account, Security, and Cost Controls

### Step 1.1: Check your Azure account

1. Open https://portal.azure.com
2. Confirm the active user and directory.
3. Open **Subscriptions**.
4. Confirm that you have an active subscription.
5. Open **Cost Management + Billing** to monitor credits and costs.

Azure free offers and limits can change. Always verify current limits at https://azure.microsoft.com/free/.

### Step 1.2: Create a Resource Group

Use one Resource Group for the whole lab. This makes cleanup easier.

```bash
export AZ_RESOURCE_GROUP="rg-bioinformatica-lab"
export AZ_LOCATION="brazilsouth"

az group create \
  --name $AZ_RESOURCE_GROUP \
  --location $AZ_LOCATION
```

### Step 1.3: Understand Azure permissions

Azure uses Microsoft Entra ID and Azure RBAC.

Common roles:

| Role | Use |
|------|-----|
| `Owner` | Full control, including access management |
| `Contributor` | Create and manage resources, but not access |
| `Reader` | Read-only access |
| `Storage Blob Data Contributor` | Read/write blobs |
| `Virtual Machine Contributor` | Manage VMs |

Important: resource management roles and blob data roles are different. You may be able to create a Storage Account but still fail to upload blobs until `Storage Blob Data Contributor` is assigned.

### Step 1.4: Create a budget

1. Open **Cost Management + Billing**.
2. Go to **Cost Management** > **Budgets**.
3. Create a monthly budget for the subscription or Resource Group.
4. Add alerts at 50%, 80%, and 100%.

Budgets alert you; they do not automatically stop resources.

---

## Azure CLI Setup

### Step 2.1: Login

```bash
az login
az account show --output table
```

Without a browser:

```bash
az login --use-device-code
```

### Step 2.2: Lab variables

```bash
export AZ_RESOURCE_GROUP="rg-bioinformatica-lab"
export AZ_LOCATION="brazilsouth"
export AZ_VM_NAME="vm-bio-lab"
export AZ_STORAGE_ACCOUNT="stbioseunome2026"
export AZ_CONTAINER="raw-data"
```

Storage Account names must be globally unique, lowercase, and without hyphens. If you use a random name, save it before continuing.

Persist variables locally:

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

## Azure Virtual Machine

### Step 3.1: Create SSH keys

```bash
mkdir -p ~/azure-bioinformatica/keys

ssh-keygen -t rsa -b 4096 \
  -f ~/azure-bioinformatica/keys/azure_bio_rsa \
  -C "azure-bioinformatica"

chmod 400 ~/azure-bioinformatica/keys/azure_bio_rsa
```

### Step 3.2: Create the VM

```bash
source ~/azure-bioinformatica/credentials/.env

az vm create \
  --resource-group $AZ_RESOURCE_GROUP \
  --name $AZ_VM_NAME \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --admin-username azureuser \
  --ssh-key-values ~/azure-bioinformatica/keys/azure_bio_rsa.pub \
  --public-ip-sku Standard
```

Open SSH only:

```bash
az vm open-port \
  --resource-group $AZ_RESOURCE_GROUP \
  --name $AZ_VM_NAME \
  --port 22
```

Do not open port 8888 for JupyterLab. Use an SSH tunnel instead.

### Step 3.3: Get the public IP

```bash
export AZ_VM_IP=$(az vm show \
  --resource-group $AZ_RESOURCE_GROUP \
  --name $AZ_VM_NAME \
  --show-details \
  --query publicIps \
  --output tsv)

echo $AZ_VM_IP
```

Do not use `ip a`, `hostname -I`, or `ifconfig` from your laptop for this. Those commands show local/VPN addresses, not the Azure VM public IP.

### Step 3.4: SSH into the VM

```bash
ssh-keyscan -H $AZ_VM_IP >> ~/.ssh/known_hosts 2>/dev/null

ssh -i ~/azure-bioinformatica/keys/azure_bio_rsa azureuser@$AZ_VM_IP
```

If you accidentally use a local IP such as `192.168.x.x`, `10.x.x.x`, `172.16.x.x` to `172.31.x.x`, or a Tailscale `100.x.x.x` address, SSH will not reach the Azure VM.

---

## Azure Blob Storage

Run this section from your local terminal, not inside the VM, unless Azure CLI is installed and configured inside the VM.

If your prompt looks like this, you are inside the VM:

```bash
azureuser@vm-bio-lab:~$
```

Exit back to local terminal:

```bash
exit
```

### Step 4.1: Create Storage Account

```bash
source ~/azure-bioinformatica/credentials/.env

az storage account create \
  --name $AZ_STORAGE_ACCOUNT \
  --resource-group $AZ_RESOURCE_GROUP \
  --location $AZ_LOCATION \
  --sku Standard_LRS \
  --kind StorageV2
```

Confirm it exists:

```bash
az storage account show \
  --resource-group $AZ_RESOURCE_GROUP \
  --name $AZ_STORAGE_ACCOUNT \
  --query "{name:name, location:primaryLocation, status:statusOfPrimary}" \
  --output table
```

If the variable points to the wrong name:

```bash
az storage account list \
  --resource-group $AZ_RESOURCE_GROUP \
  --query "[].name" \
  --output table

export AZ_STORAGE_ACCOUNT="NAME_FROM_THE_LIST"
nano ~/azure-bioinformatica/credentials/.env
source ~/azure-bioinformatica/credentials/.env
```

### Step 4.2: Create a container

```bash
az storage container create \
  --name $AZ_CONTAINER \
  --account-name $AZ_STORAGE_ACCOUNT \
  --auth-mode login
```

Recommended logical layout:

```text
raw-data/
processed/
references/
results/
```

### Step 4.3: Assign blob data permission

Before uploading with `--auth-mode login`, assign yourself data-plane permission:

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

Wait 1-5 minutes for propagation. If needed, run `az logout` and `az login` again.

### Step 4.4: Upload and download a file

```bash
mkdir -p ~/azure-bioinformatica/data
echo "Azure test file" > ~/azure-bioinformatica/data/teste.txt

az storage blob upload \
  --account-name $AZ_STORAGE_ACCOUNT \
  --container-name $AZ_CONTAINER \
  --name raw-data/teste.txt \
  --file ~/azure-bioinformatica/data/teste.txt \
  --auth-mode login \
  --overwrite

az storage blob list \
  --account-name $AZ_STORAGE_ACCOUNT \
  --container-name $AZ_CONTAINER \
  --prefix raw-data/ \
  --output table \
  --auth-mode login
```

If login mode is still blocked in a beginner lab, you can use key mode:

```bash
az storage blob upload \
  --account-name $AZ_STORAGE_ACCOUNT \
  --container-name $AZ_CONTAINER \
  --name raw-data/teste.txt \
  --file ~/azure-bioinformatica/data/teste.txt \
  --auth-mode key \
  --overwrite
```

### Step 4.5: Upload a directory

Use Azure CLI as the main path. Do not depend on AzCopy for this course.

```bash
az storage blob upload-batch \
  --account-name $AZ_STORAGE_ACCOUNT \
  --destination $AZ_CONTAINER \
  --destination-path raw-data \
  --source ~/azure-bioinformatica/data \
  --auth-mode login \
  --overwrite
```

AzCopy is optional. `azcopy login` may require a work or school account and may block personal Microsoft accounts such as Hotmail/Outlook.

---

## Bioinformatics Tools on the VM

SSH into the VM first.

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y build-essential python3-dev python3-pip python3-venv git wget unzip curl
sudo apt install -y samtools bwa fastqc bedtools sra-toolkit sqlite3
```

Python environment:

```bash
python3 -m venv ~/bio_env
source ~/bio_env/bin/activate
pip install --upgrade pip
pip install biopython pandas numpy matplotlib seaborn scipy scikit-learn azure-storage-blob jupyter jupyterlab
```

### Optional: configure Azure CLI inside the VM

Only needed if you want to run `az storage ...` directly inside the VM.

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

Validate:

```bash
az storage blob list \
  --account-name $AZ_STORAGE_ACCOUNT \
  --container-name $AZ_CONTAINER \
  --output table \
  --auth-mode login
```

---

## JupyterLab by SSH Tunnel

Do not open port 8888 in Azure. Use an SSH tunnel.

Local terminal:

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

Inside the VM:

```bash
source ~/bio_env/bin/activate
jupyter lab --ip=127.0.0.1 --port=8888 --no-browser
```

Open locally:

```text
http://127.0.0.1:8888/lab?token=<your_token>
```

---

## First Project - FASTQ Analysis

Inside the VM:

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
```

Create analysis script:

```bash
cat > ~/analisar_fastq.py << 'EOF'
#!/usr/bin/env python3

from Bio import SeqIO
import json
import sys

def analyze_fastq(path):
    sizes = []
    qualities = []
    for record in SeqIO.parse(path, "fastq"):
        size = len(record.seq)
        quality = sum(record.letter_annotations["phred_quality"]) / size
        sizes.append(size)
        qualities.append(quality)
    return {
        "total_reads": len(sizes),
        "average_length": sum(sizes) / len(sizes) if sizes else 0,
        "min_length": min(sizes) if sizes else 0,
        "max_length": max(sizes) if sizes else 0,
        "average_quality": sum(qualities) / len(qualities) if qualities else 0,
    }

if __name__ == "__main__":
    print(json.dumps(analyze_fastq(sys.argv[1]), indent=2))
EOF

chmod +x ~/analisar_fastq.py
source ~/bio_env/bin/activate
python3 ~/analisar_fastq.py ~/datasets/teste.fastq > ~/datasets/resultado.json
cat ~/datasets/resultado.json
```

### Upload option A: copy to local and upload

Local terminal:

```bash
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

### Upload option B: directly from VM

Inside the VM, after Azure CLI setup:

```bash
source ~/azure-bioinformatica/credentials/.env

az storage blob upload \
  --account-name $AZ_STORAGE_ACCOUNT \
  --container-name $AZ_CONTAINER \
  --name results/resultado.json \
  --file ~/datasets/resultado.json \
  --auth-mode login \
  --overwrite
```

---

## SQLite Results

Inside the VM:

```bash
sqlite3 ~/datasets/resultados.db << 'EOF'
CREATE TABLE IF NOT EXISTS fastq_resultados (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  arquivo TEXT,
  total_reads INTEGER,
  tamanho_medio REAL,
  qualidade_media REAL,
  criado_em TEXT DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO fastq_resultados (arquivo, total_reads, tamanho_medio, qualidade_media)
VALUES ('teste.fastq', 2, 16.0, 40.0);

SELECT * FROM fastq_resultados;
EOF
```

Upload:

```bash
az storage blob upload \
  --account-name $AZ_STORAGE_ACCOUNT \
  --container-name $AZ_CONTAINER \
  --name results/resultados.db \
  --file ~/datasets/resultados.db \
  --auth-mode login \
  --overwrite
```

Azure Portal can download `resultados.db` as a blob, but it cannot display SQLite tables. Use JupyterLab:

```python
import sqlite3
import pandas as pd

conn = sqlite3.connect("/home/azureuser/datasets/resultados.db")
df = pd.read_sql_query("SELECT * FROM fastq_resultados", conn)
df
```

---

## Stop the VM and Keep It for Later

To pause the lab without deleting it:

```bash
source ~/azure-bioinformatica/credentials/.env

az vm deallocate \
  --resource-group $AZ_RESOURCE_GROUP \
  --name $AZ_VM_NAME
```

Confirm:

```bash
az vm get-instance-view \
  --resource-group $AZ_RESOURCE_GROUP \
  --name $AZ_VM_NAME \
  --query instanceView.statuses[].displayStatus \
  --output table
```

Expected:

```text
VM deallocated
```

`VM deallocated` stops compute charges. Disks, Storage Account, and some network resources remain so you can resume later.

Start again:

```bash
az vm start \
  --resource-group $AZ_RESOURCE_GROUP \
  --name $AZ_VM_NAME
```

Get the public IP again after starting:

```bash
export AZ_VM_IP=$(az vm show \
  --resource-group $AZ_RESOURCE_GROUP \
  --name $AZ_VM_NAME \
  --show-details \
  --query publicIps \
  --output tsv)
```

Delete everything only when you are done permanently:

```bash
az group delete --name $AZ_RESOURCE_GROUP --yes
```
