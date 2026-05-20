# AWS: Practical Executive Guide

This guide walks through a low-cost AWS lab for bioinformatics and data science.

---

## Security (30 minutes)

### Step 1.1: Create an IAM user

**Why:** do not use root credentials for development.

1. Open the AWS Console.
2. Go to **IAM** > **Users**.
3. Click **Create user**.
4. Use a name such as `yourname` or `bio-lab-user`.
5. Enable console access only if you need it.
6. Save credentials securely.

### Step 1.2: Attach basic permissions

For a beginner lab, attach only what you need:

- `AmazonEC2FullAccess`
- `AmazonS3FullAccess`
- `IAMReadOnlyAccess`

For production, use least privilege instead of broad managed policies.

### Step 1.3: Create access keys for CLI

1. Open the IAM user.
2. Go to **Security credentials**.
3. Click **Create access key**.
4. Choose **Application running outside AWS**.
5. Save the CSV locally.

Never commit access keys. Store them outside the repository.

---

## EC2 - First Linux Machine (45 minutes)

### Step 2.1: Create an SSH key pair

In the EC2 console:

1. Go to **EC2** > **Key pairs**.
2. Click **Create key pair**.
3. Name: `training_keys`.
4. Type: RSA.
5. Format: `.pem` for Linux/macOS.

On your local machine:

```bash
mkdir -p ~/aws-bioinformatica/keys
mv ~/Downloads/training_keys.pem ~/aws-bioinformatica/keys/
chmod 400 ~/aws-bioinformatica/keys/training_keys.pem
```

### Step 2.2: Launch an EC2 instance

Recommended lab settings:

- Name: `training_lab`
- AMI: Amazon Linux 2023 or Ubuntu LTS
- Instance type: free-tier eligible small instance
- Key pair: `training_keys`
- Network: default VPC
- Storage: 8-30 GB SSD
- Security group: allow SSH from your IP when possible

Avoid leaving SSH open to the whole internet in real projects.

### Step 2.3: Connect by SSH

Get the public IPv4 address from the EC2 console. Do not use your local machine IP.

```bash
export AWS_EC2_IP="<YOUR_EC2_PUBLIC_IP>"

ssh-keyscan -H $AWS_EC2_IP >> ~/.ssh/known_hosts 2>/dev/null

ssh -i ~/aws-bioinformatica/keys/training_keys.pem ec2-user@$AWS_EC2_IP
```

Ubuntu AMIs usually use the `ubuntu` username instead of `ec2-user`.

---

## S3 - Cloud Storage (30 minutes)

### Step 3.1: Create a bucket

1. Go to **S3** > **Create bucket**.
2. Name: `yourname-bioinformatics-2026`.
3. Region: choose one region and keep the lab there.
4. Keep public access blocked unless the exercise explicitly requires public data.
5. Create the bucket.

Recommended layout:

```text
s3://your-bucket-bio/
├── raw-data/
├── processed/
├── references/
└── results/
```

### Step 3.2: Upload a test file

```bash
echo "test file" > /tmp/test.txt
aws s3 cp /tmp/test.txt s3://your-bucket-bio/raw-data/
aws s3 ls s3://your-bucket-bio/raw-data/
```

---

## AWS CLI (1 hour)

### Step 4.1: Install AWS CLI on EC2

Amazon Linux:

```bash
sudo yum update -y
sudo yum install -y unzip curl
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version
```

Ubuntu:

```bash
sudo apt update
sudo apt install -y awscli
aws --version
```

### Step 4.2: Configure credentials

```bash
aws configure
aws sts get-caller-identity
```

Use:

- Region: `sa-east-1`
- Output: `json`

If credentials are not found, check `~/.aws/credentials` and confirm you are using the same Linux user.

---

## Bioinformatics Tools (2 hours)

Run on the EC2 instance.

```bash
# Amazon Linux
sudo yum update -y
sudo yum install -y gcc python3-devel wget unzip

# Ubuntu
sudo apt update
sudo apt install -y build-essential python3-dev wget unzip
```

Python environment:

```bash
python3 -m venv ~/bio_env
source ~/bio_env/bin/activate
pip install --upgrade pip
pip install biopython pandas numpy matplotlib seaborn scipy scikit-learn jupyter jupyterlab
```

System tools:

```bash
# Ubuntu
sudo apt install -y samtools bwa fastqc bedtools sra-toolkit
```

---

## JupyterLab

Use SSH port forwarding instead of opening Jupyter publicly.

Local terminal:

```bash
ssh -i ~/aws-bioinformatica/keys/training_keys.pem \
  -L 8888:localhost:8888 \
  ec2-user@$AWS_EC2_IP
```

Inside EC2:

```bash
source ~/bio_env/bin/activate
jupyter lab --ip=127.0.0.1 --port=8888 --no-browser
```

Open the local URL with the token shown by Jupyter.

---

## First Project - Basic FASTQ Analysis

Create a test FASTQ:

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

Create the analysis script:

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
    if len(sys.argv) < 2:
        print("Usage: python3 analisar_fastq.py file.fastq")
        sys.exit(1)

    print(json.dumps(analyze_fastq(sys.argv[1]), indent=2))
EOF

chmod +x ~/analisar_fastq.py
```

Run and upload:

```bash
cd ~/datasets
python3 ~/analisar_fastq.py teste.fastq > resultado.json
aws s3 cp resultado.json s3://your-bucket-bio/results/
aws s3 ls s3://your-bucket-bio/results/
```

---

## Monitoring and Cost Control

### Stop EC2 when not using it

```bash
aws ec2 stop-instances --instance-ids <INSTANCE_ID> --region sa-east-1
```

Stopping EC2 stops compute charges, but EBS volumes and snapshots may still cost money.

### Weekly checklist

- Check running EC2 instances.
- Delete unused EBS volumes.
- Remove unused Elastic IPs.
- Review S3 storage.
- Check billing dashboard.
- Keep budgets and alerts enabled.

### Cleanup lab resources

Delete only when you are sure you no longer need the lab:

```bash
aws s3 rm s3://your-bucket-bio --recursive
aws s3 rb s3://your-bucket-bio
```

Terminate EC2 only if you no longer need the machine:

```bash
aws ec2 terminate-instances --instance-ids <INSTANCE_ID> --region sa-east-1
```
