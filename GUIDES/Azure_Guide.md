# Azure: Guia Prático Executivo

## Segurança e Conta (30 minutos)

### Passo 1.1: Criar ou Validar a Conta Azure

**Por que:** O Azure organiza tudo por assinatura, grupos de recursos e permissões. Antes de criar máquinas ou storage, confirme que você está no diretório e na assinatura corretos.

1. Acesse: https://portal.azure.com
2. No canto superior direito, confirme seu usuário.
3. Abra **Subscriptions** ou **Assinaturas**.
4. Confirme que existe uma assinatura ativa.
5. Se estiver usando a conta gratuita, acompanhe o crédito em **Cost Management**.

**Atenção:** A conta gratuita da Azure costuma oferecer crédito inicial por período limitado e serviços gratuitos por limites mensais. Os limites podem mudar, então valide sempre em: https://azure.microsoft.com/free/

### Passo 1.2: Criar Grupo de Recursos

**Por que:** O Resource Group é a pasta lógica onde ficam VM, storage, rede, IP e discos. Para laboratório, coloque tudo no mesmo grupo para limpar depois com segurança.

1. No portal Azure, vá em **Resource groups**.
2. Clique em **Create**.
3. Selecione sua assinatura.
4. Nome: `rg-bioinformatica-lab`
5. Região: **Brazil South** ou a região mais próxima disponível.
6. Clique em **Review + create**.
7. Clique em **Create**.

**Exemplo de grupo criado:** `rg-bioinformatica-lab`

### Passo 1.3: Entender Permissões Básicas

No Azure, permissões são controladas por **Microsoft Entra ID** e **Azure RBAC**.

**Regras práticas:**

1. Não use conta administrativa para automações permanentes.
2. Use sua conta pessoal apenas para o laboratório.
3. Para projetos reais, crie grupos e atribua papéis específicos.
4. Prefira permissões no nível do Resource Group, não da assinatura inteira.

**Papéis comuns:**

| Papel | Uso |
|-------|-----|
| `Owner` | Controle total, inclusive permissões |
| `Contributor` | Cria e altera recursos, mas não gerencia acesso |
| `Reader` | Apenas leitura |
| `Storage Blob Data Contributor` | Ler e escrever blobs no Storage |
| `Virtual Machine Contributor` | Criar e gerenciar VMs |

### Passo 1.4: Criar Budget e Alertas de Custo

**Por que:** Cloud cobra por recursos esquecidos. O budget avisa antes da conta sair do controle.

1. No portal, abra **Cost Management + Billing**.
2. Vá em **Cost Management** → **Budgets**.
3. Clique em **Add**.
4. Escopo: sua assinatura ou `rg-bioinformatica-lab`.
5. Nome: `budget-lab-azure`
6. Período: mensal.
7. Valor sugerido: R$ 5,00 ou R$ 10,00 para laboratório.
8. Alertas:
   - 50% do orçamento
   - 80% do orçamento
   - 100% do orçamento
9. Informe seu email e salve.

**Importante:** Budgets avisam, mas não desligam recursos automaticamente.

---

## Azure CLI - Automatizar Tudo (45 minutos)

### Passo 2.1: Instalar Azure CLI no Notebook

No Ubuntu/Debian:

```bash
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

az version
```

No Windows, instale pelo MSI oficial ou use WSL. No macOS:

```bash
brew update
brew install azure-cli

az version
```

### Passo 2.2: Login e Seleção de Assinatura

```bash
az login

# Listar assinaturas
az account list --output table

# Selecionar assinatura correta
az account set --subscription "<SUBSCRIPTION_ID>"

# Confirmar contexto atual
az account show --output table
```

Se estiver em uma máquina sem navegador:

```bash
az login --use-device-code
```

### Passo 2.3: Definir Variáveis do Laboratório

```bash
export AZ_RESOURCE_GROUP="rg-bioinformatica-lab"
export AZ_LOCATION="brazilsouth"
export AZ_VM_NAME="vm-bio-lab"
export AZ_STORAGE_ACCOUNT="stbio$RANDOM$RANDOM"
export AZ_CONTAINER="raw-data"

az group create \
  --name $AZ_RESOURCE_GROUP \
  --location $AZ_LOCATION
```

**Dica:** nomes de Storage Account precisam ser globalmente únicos, minúsculos e sem hífen.

---

## Virtual Machine - Sua Primeira Máquina (1 hora)

### Passo 3.1: Criar Chave SSH

No seu notebook:

```bash
mkdir -p ~/azure-bioinformatica/keys

ssh-keygen -t rsa -b 4096 \
  -f ~/azure-bioinformatica/keys/azure_bio_rsa \
  -C "azure-bioinformatica"

chmod 400 ~/azure-bioinformatica/keys/azure_bio_rsa
```

Arquivos criados:

```bash
~/azure-bioinformatica/keys/azure_bio_rsa
~/azure-bioinformatica/keys/azure_bio_rsa.pub
```

### Passo 3.2: Criar VM pelo Portal

1. No portal, vá em **Virtual machines**.
2. Clique em **Create** → **Azure virtual machine**.
3. Resource group: `rg-bioinformatica-lab`
4. Nome da VM: `vm-bio-lab`
5. Região: **Brazil South** ou região próxima.
6. Imagem: **Ubuntu Server 22.04 LTS**.
7. Tamanho: escolha uma opção pequena de laboratório, como série `B`.
8. Autenticação: **SSH public key**.
9. Username: `azureuser`
10. SSH public key source: **Use existing public key**.
11. Cole o conteúdo de:

```bash
cat ~/azure-bioinformatica/keys/azure_bio_rsa.pub
```

12. Portas de entrada: permitir **SSH (22)**.
13. Clique em **Review + create**.
14. Clique em **Create**.

**Dica de segurança:** Se possível, restrinja a porta 22 ao seu IP público em vez de deixar `0.0.0.0/0`.

### Passo 3.3: Criar VM via Azure CLI

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

Liberar SSH:

```bash
az vm open-port \
  --resource-group $AZ_RESOURCE_GROUP \
  --name $AZ_VM_NAME \
  --port 22
```

Obter IP público:

```bash
az vm show \
  --resource-group $AZ_RESOURCE_GROUP \
  --name $AZ_VM_NAME \
  --show-details \
  --query publicIps \
  --output tsv
```

### Passo 3.4: Conectar via SSH

```bash
export AZ_VM_IP="<SEUIP>"

ssh-keyscan -H $AZ_VM_IP >> ~/.ssh/known_hosts 2>/dev/null

ssh -i ~/azure-bioinformatica/keys/azure_bio_rsa azureuser@$AZ_VM_IP
```

Criar alias:

```bash
cat >> ~/.bashrc << 'EOF'
# Azure Training Lab
alias azure-lab="ssh -i ~/azure-bioinformatica/keys/azure_bio_rsa azureuser@<SEUIP>"
EOF

source ~/.bashrc

azure-lab
```

Teste dentro da VM:

```bash
uname -a
pwd
whoami
```

Para sair:

```bash
exit
```

---

## Blob Storage - Armazenando Dados na Nuvem (45 minutos)

### Passo 4.1: Criar Storage Account

Pelo portal:

1. Vá em **Storage accounts**.
2. Clique em **Create**.
3. Resource group: `rg-bioinformatica-lab`.
4. Storage account name: `stbioseunome2026`.
5. Região: mesma da VM.
6. Performance: **Standard**.
7. Redundancy: **Locally-redundant storage (LRS)** para laboratório.
8. Clique em **Review** → **Create**.

Via CLI:

```bash
az storage account create \
  --name $AZ_STORAGE_ACCOUNT \
  --resource-group $AZ_RESOURCE_GROUP \
  --location $AZ_LOCATION \
  --sku Standard_LRS \
  --kind StorageV2
```

### Passo 4.2: Criar Container

```bash
az storage container create \
  --name $AZ_CONTAINER \
  --account-name $AZ_STORAGE_ACCOUNT \
  --auth-mode login
```

Estrutura recomendada:

```bash
bioinformatica/
├── raw-data/
├── processed/
├── references/
└── results/
```

### Passo 4.3: Upload e Download de Arquivos

Criar arquivo de teste:

```bash
mkdir -p ~/azure-bioinformatica/data
echo "Teste de arquivo Azure" > ~/azure-bioinformatica/data/teste.txt
```

Upload:

```bash
az storage blob upload \
  --account-name $AZ_STORAGE_ACCOUNT \
  --container-name $AZ_CONTAINER \
  --name raw-data/teste.txt \
  --file ~/azure-bioinformatica/data/teste.txt \
  --auth-mode login
```

Listar:

```bash
az storage blob list \
  --account-name $AZ_STORAGE_ACCOUNT \
  --container-name $AZ_CONTAINER \
  --output table \
  --auth-mode login
```

Download:

```bash
az storage blob download \
  --account-name $AZ_STORAGE_ACCOUNT \
  --container-name $AZ_CONTAINER \
  --name raw-data/teste.txt \
  --file ~/azure-bioinformatica/data/teste-download.txt \
  --auth-mode login

cat ~/azure-bioinformatica/data/teste-download.txt
```

### Passo 4.4: Usar AzCopy para Pastas Grandes

AzCopy é mais eficiente para sincronizar diretórios grandes.

```bash
# Login no AzCopy
azcopy login

# Upload de pasta
azcopy copy "~/azure-bioinformatica/data/*" \
  "https://$AZ_STORAGE_ACCOUNT.blob.core.windows.net/$AZ_CONTAINER/raw-data/" \
  --recursive=true
```

---

## Instalar Ferramentas de Bioinformática (1 hora)

**Na sua VM Azure via SSH:**

### Passo 5.1: Dependências Básicas

```bash
sudo apt update
sudo apt upgrade -y

sudo apt install -y \
  build-essential \
  python3-dev \
  python3-pip \
  python3-venv \
  git \
  wget \
  unzip \
  curl
```

### Passo 5.2: Python + Bibliotecas

```bash
python3 -m venv ~/bio_env
source ~/bio_env/bin/activate

pip install --upgrade pip
pip install biopython pandas numpy matplotlib seaborn scipy scikit-learn azure-storage-blob

python3 << 'EOF'
from Bio import SeqIO
import pandas as pd
import azure.storage.blob

print("Biopython OK")
print("Pandas OK")
print("Azure Storage Blob SDK OK")
EOF
```

### Passo 5.3: Ferramentas do Sistema

```bash
sudo apt install -y samtools bwa fastqc bedtools sra-toolkit

samtools --version
bwa
fastqc --version
bedtools --version
```

### Passo 5.4: Instalar Azure CLI na VM

```bash
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

az version
az login --use-device-code
az account show --output table
```

**Dica:** Para laboratório, `az login --use-device-code` é simples. Para produção, prefira Managed Identity.

---

## JupyterLab na VM (45 minutos)

### Passo 6.1: Instalar JupyterLab

Dentro da VM:

```bash
source ~/bio_env/bin/activate
pip install jupyter jupyterlab
```

### Passo 6.2: Liberar Porta 8888 com Cuidado

No Azure CLI local:

```bash
az vm open-port \
  --resource-group $AZ_RESOURCE_GROUP \
  --name $AZ_VM_NAME \
  --port 8888
```

**Mais seguro:** use túnel SSH em vez de abrir a porta publicamente.

No notebook local:

```bash
ssh -i ~/azure-bioinformatica/keys/azure_bio_rsa \
  -L 8888:localhost:8888 \
  azureuser@$AZ_VM_IP
```

Na VM:

```bash
source ~/bio_env/bin/activate
jupyter lab --ip=127.0.0.1 --port=8888 --no-browser
```

Abra no navegador local o link com token exibido pelo Jupyter.

### Passo 6.3: Ver Token Atual

```bash
jupyter server list
```

---

## Primeiro Projeto - FASTQ Básico (2 horas)

### Passo 7.1: Criar FASTQ de Teste

Na VM:

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

### Passo 7.2: Criar Script Python

```bash
cat > ~/analisar_fastq.py << 'EOF'
#!/usr/bin/env python3

from Bio import SeqIO
import json
import sys

def analisar_fastq(arquivo):
    reads = []
    qualidades = []
    tamanhos = []

    for record in SeqIO.parse(arquivo, "fastq"):
        tamanho = len(record.seq)
        qualidade = sum(record.letter_annotations["phred_quality"]) / tamanho
        reads.append(record.id)
        tamanhos.append(tamanho)
        qualidades.append(qualidade)

    return {
        "total_reads": len(reads),
        "tamanho_medio": sum(tamanhos) / len(tamanhos) if tamanhos else 0,
        "tamanho_min": min(tamanhos) if tamanhos else 0,
        "tamanho_max": max(tamanhos) if tamanhos else 0,
        "qualidade_media": sum(qualidades) / len(qualidades) if qualidades else 0
    }

if __name__ == "__main__":
    if len(sys.argv) < 2:
        print("Uso: python3 analisar_fastq.py arquivo.fastq")
        sys.exit(1)

    print(json.dumps(analisar_fastq(sys.argv[1]), indent=2))
EOF

chmod +x ~/analisar_fastq.py
```

### Passo 7.3: Executar Análise

```bash
source ~/bio_env/bin/activate

cd ~/datasets
python3 ~/analisar_fastq.py teste.fastq

python3 ~/analisar_fastq.py teste.fastq > resultado.json
cat resultado.json
```

### Passo 7.4: Enviar Resultado ao Blob Storage

Se ainda não estiver logado:

```bash
az login --use-device-code
```

Upload:

```bash
az storage blob upload \
  --account-name $AZ_STORAGE_ACCOUNT \
  --container-name $AZ_CONTAINER \
  --name results/resultado.json \
  --file ~/datasets/resultado.json \
  --auth-mode login
```

Verificar:

```bash
az storage blob list \
  --account-name $AZ_STORAGE_ACCOUNT \
  --container-name $AZ_CONTAINER \
  --prefix results/ \
  --output table \
  --auth-mode login
```

---

## Banco de Dados e Resultados Estruturados (45 minutos)

### Passo 8.1: Opção Mais Barata - SQLite na VM

Para laboratório, SQLite resolve muito bem.

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

Inserir resultado:

```bash
sqlite3 ~/datasets/resultados.db << 'EOF'
INSERT INTO fastq_resultados (arquivo, total_reads, tamanho_medio, qualidade_media)
VALUES ('teste.fastq', 2, 16.0, 40.0);

SELECT * FROM fastq_resultados;
EOF
```

Enviar o banco para Blob Storage:

```bash
az storage blob upload \
  --account-name $AZ_STORAGE_ACCOUNT \
  --container-name $AZ_CONTAINER \
  --name results/resultados.db \
  --file ~/datasets/resultados.db \
  --auth-mode login
```

### Passo 8.2: Quando Usar Azure SQL

Use Azure SQL Database apenas se precisar:

- Acesso multiusuário
- Consultas SQL remotas
- Backup gerenciado
- Integração com dashboards
- Controle fino de permissões

Para aula básica, SQLite na VM é mais barato e mais simples.

---

## Monitoramento e Economia (1 hora)

### Passo 9.1: Ver Custos no Portal

1. Abra **Cost Management + Billing**.
2. Vá em **Cost analysis**.
3. Filtre por Resource Group: `rg-bioinformatica-lab`.
4. Agrupe por **Service name**.
5. Verifique VM, Disks, Storage e Bandwidth.

### Passo 9.2: Ver Recursos Ativos

```bash
az resource list \
  --resource-group $AZ_RESOURCE_GROUP \
  --output table
```

### Passo 9.3: Parar VM Quando Não Usar

Parar pela CLI:

```bash
az vm deallocate \
  --resource-group $AZ_RESOURCE_GROUP \
  --name $AZ_VM_NAME
```

**Importante:** Use `deallocate`, não apenas shutdown dentro do Linux. VM desalocada para cobrança de compute, mas discos e IPs ainda podem gerar custo.

Iniciar novamente:

```bash
az vm start \
  --resource-group $AZ_RESOURCE_GROUP \
  --name $AZ_VM_NAME
```

### Passo 9.4: Auto-shutdown pelo Portal

1. Abra a VM no portal.
2. Vá em **Operations** → **Auto-shutdown**.
3. Marque **Enabled**.
4. Escolha horário diário, por exemplo `19:00`.
5. Configure timezone.
6. Opcional: informe email para notificação.
7. Salve.

### Passo 9.5: Checklist de Economia

- [ ] VM desalocada quando terminar.
- [ ] Discos não utilizados removidos.
- [ ] IP público não utilizado removido.
- [ ] Storage sem arquivos duplicados.
- [ ] Budget configurado.
- [ ] Região única para evitar transferência desnecessária.
- [ ] Recursos do laboratório no mesmo Resource Group.

---

## Limpeza do Laboratório (15 minutos)

### Opção 1: Desligar e Manter Dados

```bash
az vm deallocate \
  --resource-group $AZ_RESOURCE_GROUP \
  --name $AZ_VM_NAME
```

### Opção 2: Remover Tudo do Laboratório

**Cuidado:** este comando remove todos os recursos dentro do grupo.

```bash
az group delete \
  --name $AZ_RESOURCE_GROUP \
  --yes
```

Confirmar:

```bash
az group exists --name $AZ_RESOURCE_GROUP
```

Se retornar `false`, o grupo foi removido.

---

## Próximos Passos

### Serviços para Explorar Depois

1. **Azure Machine Learning** - experimentos e modelos.
2. **Azure Batch** - execução paralela de jobs.
3. **Azure Data Factory** - pipelines de dados.
4. **Azure Functions** - automações serverless.
5. **Azure Container Registry** - imagens Docker.
6. **Azure Kubernetes Service** - workloads em containers.

### Alternativas para Economizar

- Rodar testes localmente antes da VM.
- Usar VM pequena da série B para laboratório.
- Enviar resultados ao Blob Storage e apagar dados temporários.
- Usar SQLite antes de criar banco gerenciado.
- Criar tudo em um Resource Group para limpeza rápida.

---

## Recursos Recomendados

### Documentação

- Azure Free Account: https://azure.microsoft.com/free/
- Azure CLI: https://learn.microsoft.com/cli/azure/
- Criar VM Linux: https://learn.microsoft.com/azure/virtual-machines/linux/
- Azure Blob Storage: https://learn.microsoft.com/azure/storage/blobs/
- Azure Cost Management: https://learn.microsoft.com/azure/cost-management-billing/

### Comunidades

- Microsoft Learn
- Microsoft Q&A
- Stack Overflow tag `azure`
- GitHub Azure Samples
