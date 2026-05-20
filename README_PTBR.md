# Guia Completo: Cloud para Bioinformática e Ciência de Dados

## Idioma / Language

- [English version / main README](README.md)
- [Versao PT-BR](README_PTBR.md)

---

## Índice
1. [Ponto de entrada](#ponto-de-entrada)
2. [Entender o Free Tier](#entender-o-free-tier)
3. [Arquitetura Recomendada](#arquitetura-recomendada)
4. [Serviços Essenciais](#serviços-essenciais)
5. [Proteção contra Custos AWS](#proteção-contra-custos-aws)
6. [Proteção contra Custos Azure](#proteção-contra-custos-azure)

---

## Ponto de entrada

### AWS

- [Preparação local](SETUP_LOCAL.md)
- [Guia completo](GUIDES/AWS_Guide.md)
- [Cheat sheet de comandos](GUIDES/AWS_CheatSheet.md)
- [Datasets públicos](GUIDES/AWS_Datasets.md)

### Azure

- [Preparação local](SETUP_LOCAL.md#preparacao-local-azure)
- [Guia completo](GUIDES/Azure_Guide.md)
- [Cheat sheet de comandos](GUIDES/Azure_CheatSheet.md)

---

## Entender o Free Tier

### AWS

### O que você tem GRATUITAMENTE (12 meses)

| Serviço | Limite | Importante |
|---------|--------|-----------|
| **EC2** | 750 horas/mês t2.micro | ~31 dias contínuos OU 750 horas dispersas |
| **S3** | 5 GB armazenamento | CUIDADO: download (egress) é caro |
| **RDS** | 750 horas/mês db.t2.micro | Banco de dados gratuito |
| **Data Transfer** | 1 GB/mês OUT | Acima disso = R$ 0,92/GB |
| **CloudWatch** | 10 alarms grátis | Monitoramento essencial |
| **Lambda** | 1 milhão requisições/mês | Permanente (não expira) |
| **SNS** | 1.000 notificações/mês | Para alertas |

### O que CUSTA (evite!)
- **Transferência de dados entre regiões** (egress)
- **Instâncias maiores que t2.micro**
- **Snapshots de EBS** (backups)
- **Dados enviados para fora da AWS**
- **IP elástico sem uso**

### Azure

### O que você recebe para começar

| Recurso | Uso no laboratório | Importante |
|---------|--------------------|------------|
| **Crédito inicial** | Testar VM, Storage e serviços gerenciados | Normalmente tem prazo limitado |
| **Serviços gratuitos por 12 meses** | Alguns serviços populares dentro de limites mensais | Confirme limites atuais no portal |
| **Serviços sempre gratuitos** | Automação e recursos pequenos selecionados | Exceder limites gera cobrança |
| **Cost Management** | Budgets, análise e alertas | Alertas não desligam recursos automaticamente |
| **Blob Storage** | Armazenar FASTQ, BAM, resultados e referências | Egress e redundância podem custar |
| **VMs pequenas** | Processamento Linux para aula | Desalocar quando terminar |

### O que CUSTA (evite!)
- **VM ligada sem uso**
- **Discos gerenciados esquecidos**
- **IP público reservado sem necessidade**
- **Transferência de dados para fora da Azure**
- **Storage com redundância maior que o necessário para laboratório**
- **Bancos gerenciados para testes simples**

---

## Arquitetura Recomendada

### AWS

```
┌─────────────────────────────────────────────────────────┐
│                    SEU WORKFLOW AWS                      │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  1. S3 (Dados)  ──→  2. EC2 (Processamento)             │
│     ↓                      ↓                              │
│  - Fastq files         - Python/Biopython                │
│  - Genomes             - R/Bioconductor                  │
│  - Resultados          - Samtools, BWA                   │
│                        - Jupyter Notebooks               │
│                                                           │
│                        ↓                                  │
│              3. RDS (Resultados)  /  LocalStack (local)  │
│                 ou SQLite em EC2                         │
│                                                           │
└─────────────────────────────────────────────────────────┘
```

### Azure

```
┌─────────────────────────────────────────────────────────┐
│                   SEU WORKFLOW AZURE                     │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  1. Blob Storage (Dados) ──→ 2. Virtual Machine          │
│     ↓                          ↓                         │
│  - Fastq files             - Ubuntu Server               │
│  - Genomas                 - Python/Biopython            │
│  - Resultados              - R/Bioconductor              │
│  - Referências             - Samtools, BWA, FastQC       │
│                                                           │
│                             ↓                             │
│                  3. SQLite na VM / Azure SQL             │
│                     para resultados estruturados          │
│                                                           │
│  4. Cost Management para budgets, alertas e limpeza       │
│                                                           │
└─────────────────────────────────────────────────────────┘
```

---

## Serviços Essenciais

### AWS

### EC2 (Elastic Compute Cloud)
**Por que:** máquina virtual Linux para rodar análises bioinformáticas

**O que fazer:**
```bash
# Seu workflow típico:
1. Conectar via SSH
2. Instalar ferramentas (Python, Biopython, Samtools, etc)
3. Processar dados do S3
4. Salvar resultados no S3
```

**Configuração segura:**
- Selecionar `t2.micro` (sempre gratuita)
- Linux (Amazon Linux 2 ou Ubuntu)
- Configurar Security Group (apenas SSH, HTTP, HTTPS)
- Usar par de chaves para acesso

### S3 (Simple Storage Service)
**Por que:** armazenar datasets grandes sem cobrança (dentro dos 5 GB)

**Como usar:**
```bash
# Estrutura recomendada:
s3://seu-bucket-bio/
  ├── raw-data/          (dados brutos FASTQ, genomas)
  ├── processed/         (dados processados)
  ├── references/        (genomas de referência)
  └── results/           (análises finais)
```

**Dica:** Use `aws s3 sync` (gratuito internamente, não consome egress)

### RDS (Relational Database Service)
**Por que:** armazenar metadados, anotações, resultados estruturados

**Alternativa mais barata:**
- Use SQLite no EC2 em vez de RDS (ainda mais grátis)
- RDS é útil apenas se precisar multi-user

### IAM (Identity & Access Management)
**Por que:** segurança (essencial!)

**O que fazer:**
```
1. NUNCA use credenciais da conta raiz
2. Crie usuário específico para desenvolvimento
3. Associe apenas permissões necessárias (S3 + EC2)
4. Gere access keys para CLI
```

### CloudWatch + Billing Alerts
**Por que:** monitorar custos em tempo real

**Configuração crítica:**
```
→ Vá em: Billing → Manage Billing Alerts
→ Configure alarme para > R$ 1,00/mês
→ Receberá email quando atingir limite
```

### Azure

### Virtual Machines
**Por que:** máquina virtual Linux para rodar análises bioinformáticas

**O que fazer:**
```bash
# Seu workflow típico:
1. Conectar via SSH
2. Instalar ferramentas (Python, Biopython, Samtools, etc)
3. Processar dados do Blob Storage
4. Salvar resultados no Blob Storage
5. Desalocar a VM quando terminar
```

**Configuração segura:**
- Usar Ubuntu LTS
- Usar chave SSH, não senha
- Abrir apenas portas necessárias
- Preferir túnel SSH para Jupyter
- Desalocar VM ao final do uso

### Blob Storage
**Por que:** armazenar datasets, referências e resultados

**Como usar:**
```bash
# Estrutura recomendada:
container: raw-data
  ├── raw-data/          (dados brutos FASTQ, genomas)
  ├── processed/         (dados processados)
  ├── references/        (genomas de referência)
  └── results/           (análises finais)
```

**Dica:** Use `az storage blob upload` para arquivos pequenos e `azcopy` para diretórios grandes.

### Resource Groups
**Por que:** organizar e apagar todos os recursos do laboratório juntos

**O que fazer:**
```
1. Criar um Resource Group por laboratório
2. Colocar VM, rede, disco, IP e Storage no mesmo grupo
3. Usar tags se houver múltiplos alunos/projetos
4. Deletar o grupo inteiro quando terminar o laboratório
```

### Azure RBAC
**Por que:** controlar permissões de forma segura

**O que fazer:**
```
1. Usar permissões no nível do Resource Group
2. Evitar Owner para tarefas simples
3. Usar Contributor para laboratório
4. Usar Storage Blob Data Contributor para acesso a blobs
```

### Cost Management + Budgets
**Por que:** monitorar gastos e receber alertas

**Configuração crítica:**
```
→ Vá em: Cost Management + Billing
→ Cost Management → Budgets
→ Configure budget mensal baixo para laboratório
→ Crie alertas em 50%, 80% e 100%
```

---

## PROTEÇÃO CONTRA CUSTOS AWS

### 1. Configurar Alertas de Billing

```
Passo 1: Vá em https://console.aws.amazon.com/billing/
Passo 2: Clique em "Manage Billing Alerts"
Passo 3: Crie alarme para:
  - R$ 1,00 por dia
  - R$ 5,00 por mês
  (valores sugeridos para começar)
```

### 2. Monitorar Uso Diariamente

**Dica:** Criar script para verificar custos:
```python
import boto3

ce = boto3.client('ce')  # Cost Explorer

response = ce.get_cost_and_usage(
    TimePeriod={'Start': '2024-04-01', 'End': '2024-04-30'},
    Granularity='DAILY',
    Metrics=['UnblendedCost']
)

for item in response['ResultsByTime']:
    print(f"{item['TimePeriod']['Start']}: ${item['Total']['UnblendedCost']['Amount']}")
```

### 3. Hábitos de Economia

| Ação | Economia |
|------|----------|
| **Desligar EC2 quando não usar** | -R$ 20-30/mês |
| **Usar t2.micro (não t2.small)** | -R$ 15-20/mês |
| **S3: 1 região apenas** | -R$ 5-10/mês |
| **Excluir snapshots desnecessários** | -R$ 10/mês |
| **Usar CloudWatch Logs Insights** | Menos logs |

### 4. Checklist Semanal

- [ ] Verificar instâncias EC2 rodando
- [ ] Excluir volumes EBS não usados
- [ ] Revisar bucket S3 (remover duplicatas)
- [ ] Verificar Elastic IPs não associados
- [ ] Checar faturamento acumulado

---

## PROTEÇÃO CONTRA CUSTOS AZURE

### 1. Configurar Budget

```
Passo 1: Vá em Cost Management + Billing
Passo 2: Abra Cost Management → Budgets
Passo 3: Crie budget para:
  - R$ 5,00 por mês
  - R$ 10,00 por mês
  (valores sugeridos para começar)
Passo 4: Configure alertas em 50%, 80% e 100%
```

### 2. Monitorar Recursos Ativos

```bash
az resource list \
  --resource-group rg-bioinformatica-lab \
  --output table
```

### 3. Hábitos de Economia

| Ação | Economia |
|------|----------|
| **Desalocar VM quando não usar** | Evita cobrança de compute |
| **Usar VM pequena para laboratório** | Reduz custo por hora |
| **Usar Standard_LRS no Storage** | Evita redundância cara para teste |
| **Excluir discos não usados** | Evita cobrança após apagar VM |
| **Centralizar recursos no Resource Group** | Facilita limpeza completa |

### 4. Checklist Semanal

- [ ] Verificar VMs rodando
- [ ] Desalocar VMs sem uso
- [ ] Excluir discos órfãos
- [ ] Revisar Blob Storage
- [ ] Verificar IPs públicos
- [ ] Checar Cost Management

---

## Próximos Passos

### Se quiser ir além do Free Tier na AWS:
1. **SageMaker** (machine learning) - $0,25/hora
2. **Glue** (ETL) - processamento big data
3. **Athena** (query SQL em S3) - $5 por TB
4. **ECR** (Docker registry)

### Se quiser ir além do laboratório na Azure:
1. **Azure Machine Learning** - experimentos e modelos
2. **Azure Batch** - processamento paralelo
3. **Azure Data Factory** - pipelines de dados
4. **Azure Functions** - automações serverless
5. **Azure Container Registry** - imagens Docker

### Alternativas para economizar:
- **LocalStack** - Simular AWS localmente (gratuito)
- **MinIO** - S3 local
- **Docker + seu laptop** - para prototipar
- **Azurite** - Simular Azure Storage localmente

---

## Recursos Recomendados

### Cursos
- AWS Fundamentals (YouTube - Free Code Camp): 4h
- A Cloud Guru - AWS Certified Cloud Practitioner: freemium
- Linux Academy: freemium

### Documentação
- AWS Free Tier: https://aws.amazon.com/pt/free/
- Pricing Calculator: https://calculator.aws/
- Documentação EC2: https://docs.aws.amazon.com/ec2/
- Azure Free Account: https://azure.microsoft.com/free/
- Azure Pricing Calculator: https://azure.microsoft.com/pricing/calculator/
- Documentação Azure CLI: https://learn.microsoft.com/cli/azure/
- Documentação Azure Virtual Machines: https://learn.microsoft.com/azure/virtual-machines/
- Documentação Azure Blob Storage: https://learn.microsoft.com/azure/storage/blobs/

### Comunidades
- AWS Community Brasil (Slack/Discord)
- Stack Overflow tag `amazon-ec2`
- GitHub - repositórios com scripts AWS
- Microsoft Learn
- Microsoft Q&A
- Stack Overflow tag `azure`
---
