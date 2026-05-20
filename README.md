# Complete Cloud Guide for Bioinformatics and Data Science

## Language / Idioma

### English
Use these links if you want the English version of the course:

- [Local setup](SETUP_LOCAL_EN.md)
- [AWS practical guide](GUIDES/AWS_Guide_EN.md)
- [AWS command cheat sheet](GUIDES/AWS_CheatSheet_EN.md)
- [Public datasets](GUIDES/AWS_Datasets_EN.md)
- [Azure practical guide](GUIDES/Azure_Guide_EN.md)
- [Azure command cheat sheet](GUIDES/Azure_CheatSheet_EN.md)

### Portugues do Brasil
Use estes links se quiser seguir a versao em PT-BR:

- [README em portugues](README_PTBR.md)
- [Preparacao local](SETUP_LOCAL.md)
- [Guia pratico AWS](GUIDES/AWS_Guide.md)
- [Cheat sheet AWS](GUIDES/AWS_CheatSheet.md)
- [Datasets publicos](GUIDES/AWS_Datasets.md)
- [Guia pratico Azure](GUIDES/Azure_Guide.md)
- [Cheat sheet Azure](GUIDES/Azure_CheatSheet.md)

---

## What This Repository Covers

This repository is a practical, beginner-friendly cloud lab for bioinformatics and data science. It focuses on safe, low-cost workflows using AWS and Microsoft Azure.

The core workflow is:

```text
1. Store input data in object storage
   AWS S3 or Azure Blob Storage

2. Process data in a Linux virtual machine
   AWS EC2 or Azure Virtual Machine

3. Run bioinformatics tools
   Python, Biopython, Samtools, BWA, FastQC, BEDTools, JupyterLab

4. Save structured results
   JSON, SQLite, and object storage outputs

5. Control costs
   budgets, alerts, VM shutdown/deallocation, cleanup checklists
```

---

## Recommended Path

### If you are starting with AWS

1. Read [Local setup](SETUP_LOCAL_EN.md).
2. Follow [AWS practical guide](GUIDES/AWS_Guide_EN.md).
3. Keep [AWS command cheat sheet](GUIDES/AWS_CheatSheet_EN.md) open while working.
4. Use [Public datasets](GUIDES/AWS_Datasets_EN.md) when you need real sample data.

### If you are starting with Azure

1. Read [Local setup](SETUP_LOCAL_EN.md).
2. Follow [Azure practical guide](GUIDES/Azure_Guide_EN.md).
3. Keep [Azure command cheat sheet](GUIDES/Azure_CheatSheet_EN.md) open while working.
4. Use SSH tunnels for JupyterLab and deallocate the VM when finished.

---

## Cost Safety Rules

- Never commit credentials, private keys, `.env` files, datasets, SQLite databases, or generated reports.
- Use the included `.gitignore` as a safety net, but do not rely on it as your only protection.
- Deallocate Azure VMs and stop AWS EC2 instances when you are done.
- Keep budgets and billing alerts enabled.
- Treat exposed cloud credentials as compromised and rotate them immediately.

---

## Repository Layout

```text
.
├── README.md                  English entry point and language selector
├── README_PTBR.md             Portuguese entry point
├── SETUP_LOCAL.md             Local setup in Portuguese
├── SETUP_LOCAL_EN.md          Local setup in English
└── GUIDES/
    ├── AWS_Guide.md
    ├── AWS_Guide_EN.md
    ├── AWS_CheatSheet.md
    ├── AWS_CheatSheet_EN.md
    ├── AWS_Datasets.md
    ├── AWS_Datasets_EN.md
    ├── Azure_Guide.md
    ├── Azure_Guide_EN.md
    ├── Azure_CheatSheet.md
    └── Azure_CheatSheet_EN.md
```

---

## Official Documentation

- AWS Free Tier: https://aws.amazon.com/free/
- AWS Pricing Calculator: https://calculator.aws/
- AWS EC2 documentation: https://docs.aws.amazon.com/ec2/
- Azure Free Account: https://azure.microsoft.com/free/
- Azure Pricing Calculator: https://azure.microsoft.com/pricing/calculator/
- Azure CLI documentation: https://learn.microsoft.com/cli/azure/
- Azure Virtual Machines: https://learn.microsoft.com/azure/virtual-machines/
- Azure Blob Storage: https://learn.microsoft.com/azure/storage/blobs/
