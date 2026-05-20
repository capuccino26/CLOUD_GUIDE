## AWS Configuration

### Configure credentials

```bash
aws configure
aws sts get-caller-identity
cat ~/.aws/config
cat ~/.aws/credentials
```

### Region

```bash
export AWS_DEFAULT_REGION=sa-east-1
aws ec2 describe-instances --region sa-east-1
```

---

## S3 Storage

### Buckets

```bash
aws s3 ls
aws s3 mb s3://your-bucket-name --region sa-east-1
aws s3 rb s3://your-bucket-name
```

### Upload

```bash
aws s3 cp file.fastq s3://your-bucket/raw-data/
aws s3 cp result.json s3://your-bucket/results/ --sse AES256
aws s3 sync ~/local-data/ s3://your-bucket/raw-data/
aws s3 sync ~/local-data/ s3://your-bucket/raw-data/ --dryrun
```

### Download

```bash
aws s3 cp s3://your-bucket/results/result.json ~/download/
aws s3 sync s3://your-bucket/processed/ ~/processed/
```

### List and clean

```bash
aws s3 ls s3://your-bucket/ --recursive --summarize --human-readable
aws s3 rm s3://your-bucket/tmp/file.txt
aws s3 rm s3://your-bucket/tmp/ --recursive
```

---

## EC2

### List instances

```bash
aws ec2 describe-instances --query 'Reservations[].Instances[].[InstanceId,InstanceType,State.Name,PublicIpAddress,Tags[?Key==`Name`].Value|[0]]' --output table
```

### SSH

```bash
ssh -i ~/aws-bioinformatica/keys/training_keys.pem ec2-user@<PUBLIC_IP>
ssh -i ~/aws-bioinformatica/keys/training_keys.pem ubuntu@<PUBLIC_IP>
ssh -i ~/aws-bioinformatica/keys/training_keys.pem -L 8888:localhost:8888 ec2-user@<PUBLIC_IP>
```

### Stop, start, terminate

```bash
aws ec2 stop-instances --instance-ids <INSTANCE_ID> --region sa-east-1
aws ec2 start-instances --instance-ids <INSTANCE_ID> --region sa-east-1
aws ec2 terminate-instances --instance-ids <INSTANCE_ID> --region sa-east-1
```

### Instance state

```bash
aws ec2 describe-instances --instance-ids <INSTANCE_ID> --query 'Reservations[].Instances[].State'
```

---

## Bioinformatics Tools

### Install

```bash
sudo apt update
sudo apt install -y samtools bwa fastqc bedtools sra-toolkit
python3 -m venv ~/bio_env
source ~/bio_env/bin/activate
pip install biopython pandas numpy matplotlib seaborn scipy scikit-learn jupyter jupyterlab boto3
```

### FASTQ stats in Python

```python
from Bio import SeqIO
import numpy as np

sizes = []
qualities = []

for record in SeqIO.parse("file.fastq", "fastq"):
    sizes.append(len(record.seq))
    qualities.append(np.mean(record.letter_annotations["phred_quality"]))

print(f"Reads: {len(sizes)}")
print(f"Average length: {np.mean(sizes):.0f}")
print(f"Average quality: {np.mean(qualities):.2f}")
```

### BWA and Samtools

```bash
bwa index reference.fasta
bwa mem reference.fasta reads.fastq | samtools view -b - | samtools sort -o output.bam
samtools index output.bam
samtools flagstat output.bam > stats.txt
```

### FastQC

```bash
fastqc reads.fastq
fastqc *.fastq -o ~/results/
```

---

## Costs

```bash
aws ce get-cost-and-usage \
  --time-period Start=2026-05-01,End=2026-05-31 \
  --granularity DAILY \
  --metrics "UnblendedCost" \
  --query 'ResultsByTime[*].[TimePeriod.Start,Total.UnblendedCost.Amount]' \
  --output table
```

---

## Troubleshooting

### Unable to locate credentials

```bash
aws configure
aws sts get-caller-identity
```

### AccessDenied on S3

Check IAM permissions and bucket name:

```bash
aws s3 ls
aws s3 ls s3://your-bucket/
```

### SSH permission denied

```bash
chmod 400 ~/aws-bioinformatica/keys/training_keys.pem
```
