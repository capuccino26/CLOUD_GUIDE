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
export AZ_STORAGE_ACCOUNT="stbioseunome2026"
export AZ_CONTAINER="raw-data"

az group create \
  --name $AZ_RESOURCE_GROUP \
  --location $AZ_LOCATION
```

**Dica:** nomes de Storage Account precisam ser globalmente únicos, minúsculos e sem hífen. Se escolher usar `stbio$RANDOM$RANDOM`, salve o nome gerado antes de continuar; gerar a variável de novo cria outro nome e os próximos comandos podem apontar para uma conta que não existe.

Salvar as variáveis para reaproveitar em novos terminais:

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

Obter IP público da VM Azure:

```bash
export AZ_VM_IP=$(az vm show \
  --resource-group $AZ_RESOURCE_GROUP \
  --name $AZ_VM_NAME \
  --show-details \
  --query publicIps \
  --output tsv)

echo $AZ_VM_IP
```

**Importante:** este IP precisa vir do Azure, pelo comando acima ou pelo portal. Não use o resultado de `ip a`, `hostname -I` ou `ifconfig` do seu notebook, porque esses comandos mostram IPs da sua máquina local.

### Passo 3.4: Conectar via SSH

Confirme que `AZ_VM_IP` contém o IP público da VM:

```bash
echo $AZ_VM_IP
```

Se não aparecer nenhum IP, busque novamente:

```bash
export AZ_VM_IP=$(az vm show \
  --resource-group $AZ_RESOURCE_GROUP \
  --name $AZ_VM_NAME \
  --show-details \
  --query publicIps \
  --output tsv)
```

Conectar:

```bash
ssh-keyscan -H $AZ_VM_IP >> ~/.ssh/known_hosts 2>/dev/null

ssh -i ~/azure-bioinformatica/keys/azure_bio_rsa azureuser@$AZ_VM_IP
```

**Erro comum:** se você usar um IP local como `192.168.x.x`, `10.x.x.x`, `172.16.x.x` até `172.31.x.x` ou IP do Tailscale como `100.x.x.x`, o SSH não vai chegar na VM Azure. Esses IPs pertencem à sua rede local/VPN, não ao IP público da VM.

Se aparecer `Connection refused`, você provavelmente tentou conectar no IP errado ou a porta 22 não está aberta no destino. Verifique:

```bash
az vm show \
  --resource-group $AZ_RESOURCE_GROUP \
  --name $AZ_VM_NAME \
  --show-details \
  --query "[powerState, publicIps]" \
  --output table

az vm open-port \
  --resource-group $AZ_RESOURCE_GROUP \
  --name $AZ_VM_NAME \
  --port 22
```

Criar alias:

```bash
cat >> ~/.bashrc << EOF
# Azure Training Lab
alias azure-lab="ssh -i ~/azure-bioinformatica/keys/azure_bio_rsa azureuser@$AZ_VM_IP"
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

**Onde rodar:** execute os comandos desta seção no seu notebook/terminal local, não dentro da VM, a menos que você já tenha instalado a Azure CLI na VM no Passo 5.4.

Se o prompt estiver assim, você está dentro da VM:

```bash
azureuser@vm-bio-lab:~$
```

Para voltar ao terminal local:

```bash
exit
```

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

Via CLI, no terminal local:

```bash
az storage account create \
  --name $AZ_STORAGE_ACCOUNT \
  --resource-group $AZ_RESOURCE_GROUP \
  --location $AZ_LOCATION \
  --sku Standard_LRS \
  --kind StorageV2
```

Confirmar que a Storage Account existe e que a variável ainda aponta para o nome correto:

```bash
source ~/azure-bioinformatica/credentials/.env
echo $AZ_STORAGE_ACCOUNT

az storage account show \
  --resource-group $AZ_RESOURCE_GROUP \
  --name $AZ_STORAGE_ACCOUNT \
  --query "{name:name, location:primaryLocation, status:statusOfPrimary}" \
  --output table
```

Se `az storage account show` retornar erro, liste as contas existentes e corrija a variável:

```bash
az storage account list \
  --resource-group $AZ_RESOURCE_GROUP \
  --query "[].name" \
  --output table

export AZ_STORAGE_ACCOUNT="NOME_QUE_APARECEU_NA_LISTA"

nano ~/azure-bioinformatica/credentials/.env
source ~/azure-bioinformatica/credentials/.env
```

Não gere um novo `AZ_STORAGE_ACCOUNT` com `$RANDOM` no meio da aula, porque isso muda o nome usado pelos próximos comandos.

### Passo 4.2: Criar Container

No terminal local, depois de confirmar que a Storage Account existe:

```bash
az storage container create \
  --name $AZ_CONTAINER \
  --account-name $AZ_STORAGE_ACCOUNT \
  --auth-mode login
```

Se aparecer `Failed to resolve ... blob.core.windows.net`, verifique primeiro se a Storage Account realmente existe com `az storage account show`. Se ela existir, o problema é DNS/rede local; teste novamente em outra rede, desligue VPN/proxy temporariamente ou altere o DNS do sistema.

Estrutura recomendada:

```bash
bioinformatica/
├── raw-data/
├── processed/
├── references/
└── results/
```

### Passo 4.3: Upload e Download de Arquivos

Antes do primeiro upload com `--auth-mode login`, confirme que seu usuário tem permissão de dados no Blob Storage. O papel `Contributor` da assinatura ou do Resource Group nem sempre é suficiente para ler/escrever blobs.

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

Aguarde 1 a 5 minutos para a permissão propagar. Se o upload ainda falhar, rode `az logout` e `az login` novamente.

Criar arquivo de teste no terminal local:

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

Se aparecer `You do not have the required permissions`, volte ao início do Passo 4.3 e atribua o papel `Storage Blob Data Contributor` ao seu usuário. Para laboratório, também é possível usar `--auth-mode key`, mas a opção com RBAC é a prática recomendada.

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

### Passo 4.4: Upload de Pastas

Use a própria Azure CLI como caminho principal. Ela já está instalada no terminal local, funciona com conta pessoal Microsoft e evita depender do AzCopy:

```bash
az storage blob upload-batch \
  --account-name $AZ_STORAGE_ACCOUNT \
  --destination $AZ_CONTAINER \
  --destination-path raw-data \
  --source ~/azure-bioinformatica/data \
  --auth-mode login \
  --overwrite
```

Se o modo login ainda apresentar permissão pendente, use a alternativa de laboratório:

```bash
az storage blob upload-batch \
  --account-name $AZ_STORAGE_ACCOUNT \
  --destination $AZ_CONTAINER \
  --destination-path raw-data \
  --source ~/azure-bioinformatica/data \
  --auth-mode key \
  --overwrite
```

**AzCopy é opcional.** Ele é útil para transferências grandes, mas o `azcopy login` pode exigir conta corporativa/escolar e bloquear contas pessoais como Hotmail/Outlook. Para esta aula, não dependa dele.

Se estiver usando uma conta corporativa/escolar e quiser testar AzCopy:

```bash
# verificar se existe
azcopy --version

# instalar no Ubuntu/Debian, se necessário
cd /tmp
wget https://aka.ms/downloadazcopy-v10-linux -O azcopy.tar.gz
tar -xzf azcopy.tar.gz
sudo cp ./azcopy_linux_amd64_*/azcopy /usr/local/bin/
azcopy --version

# login e upload
azcopy login
azcopy copy "$HOME/azure-bioinformatica/data/*" \
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

### Passo 5.4: Instalar e Configurar Azure CLI na VM

Este passo só é necessário se você quiser rodar comandos `az ...` de dentro da VM. Para a aula básica, a opção mais simples é manter a Azure CLI no notebook e usar a VM apenas para processamento.

Se quiser enviar resultados direto da VM para o Blob Storage, configure a CLI e as variáveis também dentro da VM:

```bash
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

az version
az login --use-device-code
az account show --output table
```

Criar arquivo de variáveis dentro da VM:

```bash
mkdir -p ~/azure-bioinformatica/credentials

cat > ~/azure-bioinformatica/credentials/.env << EOF
AZ_RESOURCE_GROUP=rg-bioinformatica-lab
AZ_LOCATION=brazilsouth
AZ_VM_NAME=vm-bio-lab
AZ_STORAGE_ACCOUNT=stbioseunome2026
AZ_CONTAINER=raw-data
EOF

source ~/azure-bioinformatica/credentials/.env

echo $AZ_STORAGE_ACCOUNT
echo $AZ_CONTAINER
```

Validar acesso ao container:

```bash
az storage blob list \
  --account-name $AZ_STORAGE_ACCOUNT \
  --container-name $AZ_CONTAINER \
  --output table \
  --auth-mode login
```

Se esse comando falhar por permissão, atribua `Storage Blob Data Contributor` no terminal local, aguarde alguns minutos e faça login novamente na VM.

**Dica:** Para laboratório, `az login --use-device-code` é simples. Para produção, prefira Managed Identity.

---

## JupyterLab na VM (45 minutos)

### Passo 6.1: Instalar JupyterLab

Dentro da VM:

```bash
source ~/bio_env/bin/activate
pip install jupyter jupyterlab
```

### Passo 6.2: Acessar JupyterLab com Túnel SSH

Não abra a porta 8888 no Azure para esta aula. Use túnel SSH: é mais seguro e evita conflitos de regra no Network Security Group, como `SecurityRuleConflict` entre `open-port-22` e `open-port-8888`.

No terminal local, fora do SSH:

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

Esse comando abre uma sessão SSH na VM e cria o túnel `localhost:8888` do seu notebook para a porta `8888` da VM.

Dentro da VM, na sessão SSH que abriu:

```bash
source ~/bio_env/bin/activate
jupyter lab --ip=127.0.0.1 --port=8888 --no-browser
```

Abra no navegador do notebook o link com token exibido pelo Jupyter, no formato:

```text
http://127.0.0.1:8888/lab?token=<seu_token>
```

**Onde cada comando roda:** `ssh -L ...` roda no terminal local. `jupyter lab ...` roda dentro da VM.

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

Você tem duas opções. A opção A mantém comandos Azure no notebook local. A opção B envia direto de dentro da VM, desde que o Passo 5.4 tenha sido configurado.

### Opção A: copiar para o notebook e fazer upload local

No terminal local, fora do SSH:

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

### Opção B: upload direto de dentro da VM

Dentro da VM:

```bash
source ~/azure-bioinformatica/credentials/.env

echo $AZ_STORAGE_ACCOUNT
echo $AZ_CONTAINER

az storage blob upload \
  --account-name $AZ_STORAGE_ACCOUNT \
  --container-name $AZ_CONTAINER \
  --name results/resultado.json \
  --file ~/datasets/resultado.json \
  --auth-mode login \
  --overwrite
```

Se aparecer `argument --account-name: expected one argument`, a variável `AZ_STORAGE_ACCOUNT` está vazia. Rode `source ~/azure-bioinformatica/credentials/.env` dentro da VM ou crie o arquivo conforme o Passo 5.4.

Verificar, no terminal local ou na VM configurada:

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
  --auth-mode login \
  --overwrite
```

### Passo 8.2: Visualizar o SQLite no Navegador

O Azure Portal mostra o arquivo `resultados.db` como blob, mas não abre as tabelas do SQLite. Pelo portal, você consegue baixar o arquivo em:

```text
Storage accounts
Storage account: stbioseunome2026
Data storage > Containers
Container: raw-data
Pasta: results/
Arquivo: resultados.db
Download
```

Para ver a tabela no navegador, use o JupyterLab que já está acessível pelo túnel SSH. No JupyterLab, crie um notebook e rode:

```python
import sqlite3
import pandas as pd

conn = sqlite3.connect("/home/azureuser/datasets/resultados.db")

df = pd.read_sql_query("SELECT * FROM fastq_resultados", conn)
df
```

Para listar as tabelas no notebook:

```python
pd.read_sql_query("SELECT name FROM sqlite_master WHERE type='table'", conn)
```

Também é possível conferir pelo terminal da VM:

```bash
sqlite3 ~/datasets/resultados.db ".tables"
sqlite3 ~/datasets/resultados.db "SELECT * FROM fastq_resultados;"
```

### Passo 8.3: Quando Usar Azure SQL

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

### Passo 9.3: Parar VM Quando Não Usar e Manter para Depois

Para pausar o laboratório e voltar outro dia, use `deallocate`. Esse comando para a VM e interrompe a cobrança de compute, mantendo o disco, a rede e o Storage Account.

```bash
source ~/azure-bioinformatica/credentials/.env

az vm deallocate \
  --resource-group $AZ_RESOURCE_GROUP \
  --name $AZ_VM_NAME
```

Confirmar que a VM está parada no estado correto:

```bash
az vm get-instance-view \
  --resource-group $AZ_RESOURCE_GROUP \
  --name $AZ_VM_NAME \
  --query instanceView.statuses[].displayStatus \
  --output table
```

Resultado esperado:

```text
VM deallocated
```

**Importante:** mesmo com `VM deallocated`, os recursos continuam aparecendo em `az resource list`. Isso é normal. A VM fica guardada para uso futuro, mas o compute fica parado. Ainda podem existir custos pequenos de disco gerenciado, Storage Account e, dependendo da configuração, IP público.

Ver recursos mantidos:

```bash
az resource list \
  --resource-group $AZ_RESOURCE_GROUP \
  --output table
```

Você pode ver recursos como VM, disco, VNet, NSG, interface de rede, IP público, Storage Account e `Microsoft.DevTestLab/schedules` de auto-shutdown. Isso não significa que a VM está rodando; confirme pelo estado `VM deallocated`.

Iniciar novamente no futuro:

```bash
source ~/azure-bioinformatica/credentials/.env

az vm start \
  --resource-group $AZ_RESOURCE_GROUP \
  --name $AZ_VM_NAME
```

Depois de iniciar, pegue o IP público novamente, porque ele pode mudar:

```bash
export AZ_VM_IP=$(az vm show \
  --resource-group $AZ_RESOURCE_GROUP \
  --name $AZ_VM_NAME \
  --show-details \
  --query publicIps \
  --output tsv)

echo $AZ_VM_IP
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
