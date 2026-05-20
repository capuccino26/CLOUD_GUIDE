# Local Setup - AWS and Azure

Run these steps on your own notebook/local terminal before creating cloud resources.

---

## AWS Local Setup

### Step 1: Check AWS CLI

```bash
aws --version
```

Expected: AWS CLI 2.x.

Install if needed: https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html

### Step 2: Prepare a local workspace

```bash
mkdir -p ~/aws-bioinformatica/{credentials,keys,scripts,data}
cd ~/aws-bioinformatica
pwd
ls -la
```

Expected structure:

```text
aws-bioinformatica/
├── credentials/
├── keys/
├── scripts/
└── data/
```

### Step 3: Prepare SSH keys

If you already have an SSH key:

```bash
ls -la ~/.ssh/
cp ~/.ssh/id_rsa ~/aws-bioinformatica/keys/aws_rsa
cp ~/.ssh/id_rsa.pub ~/aws-bioinformatica/keys/aws_rsa.pub
chmod 400 ~/aws-bioinformatica/keys/aws_rsa
```

Never commit private keys.

### Step 4: Create an AWS credentials template

```bash
cat > ~/aws-bioinformatica/credentials/.env.template << 'EOF'
AWS_ACCESS_KEY_ID=your_access_key_here
AWS_SECRET_ACCESS_KEY=your_secret_key_here
AWS_DEFAULT_REGION=sa-east-1
AWS_ACCOUNT_ID=your_account_id
EOF
```

Copy this template to `.env` only on your local machine. Do not commit `.env`.

### Step 5: Configure AWS CLI

```bash
aws configure
aws sts get-caller-identity
```

Use:

- Region: `sa-east-1`
- Output format: `json`

---

## Azure Local Setup

### Step 1: Check Azure CLI

```bash
az version
```

Install on Ubuntu/Debian:

```bash
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
```

Install on macOS:

```bash
brew update
brew install azure-cli
```

### Step 2: Login

```bash
az login
```

For terminals without a browser:

```bash
az login --use-device-code
```

Check the active subscription:

```bash
az account list --output table
az account show --output table
```

Set a subscription if needed:

```bash
az account set --subscription "<SUBSCRIPTION_ID>"
```

### Step 3: Create the Azure local workspace

```bash
mkdir -p ~/azure-bioinformatica/{credentials,keys,scripts,data}
cd ~/azure-bioinformatica
pwd
ls -la
```

Expected structure:

```text
azure-bioinformatica/
├── credentials/
├── keys/
├── scripts/
└── data/
```

### Step 4: Create an SSH key for Azure VM

```bash
ssh-keygen -t rsa -b 4096 \
  -f ~/azure-bioinformatica/keys/azure_bio_rsa \
  -C "azure-bioinformatica"

chmod 400 ~/azure-bioinformatica/keys/azure_bio_rsa
```

Public key:

```bash
cat ~/azure-bioinformatica/keys/azure_bio_rsa.pub
```

### Step 5: Save Azure lab variables

```bash
cat > ~/azure-bioinformatica/credentials/.env << 'EOF'
AZ_RESOURCE_GROUP=rg-bioinformatica-lab
AZ_LOCATION=brazilsouth
AZ_VM_NAME=vm-bio-lab
AZ_STORAGE_ACCOUNT=stbioseunome2026
AZ_CONTAINER=raw-data
EOF

source ~/azure-bioinformatica/credentials/.env
```

Use a globally unique Storage Account name. If you change it, update `.env` and run `source` again.

### Step 6: Test local setup

```bash
az account show --output table
ssh-keygen -l -f ~/azure-bioinformatica/keys/azure_bio_rsa.pub
```
