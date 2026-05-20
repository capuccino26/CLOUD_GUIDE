# Setup Local - Preparação para AWS e Azure

Siga estes passos NO SEU NOTEBOOK/TERMINAL para preparar o ambiente local.

---

## PREPARACAO LOCAL AWS

---

## PASSO 1: Verificar se AWS CLI está instalado

```bash
aws --version
```

**Esperado:** AWS CLI 2.x.x

Se não encontrar → [Instale aqui](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)

---

## PASSO 2: Localizar suas chaves SSH existentes

```bash
ls -la ~/.ssh/
```

Você deve ver algo como:
- `id_rsa` (chave privada)
- `id_rsa.pub` (chave pública)
- `known_hosts`

**Guarde o caminho completo**: `~/.ssh/id_rsa`

---

## PASSO 3: Criar pasta de trabalho para AWS

```bash
# Já está aqui, mas vamos organizar:
mkdir -p ~/aws-bioinformatica/{credentials,keys,scripts,data}
cd ~/aws-bioinformatica

# Confirme:
pwd
ls -la
```

**Estrutura esperada:**
```
aws-bioinformatica/
├── credentials/    (aqui vão suas credenciais AWS)
├── keys/           (aqui vão copiar chaves SSH)
├── scripts/        (scripts Python e bash)
└── data/           (dados de teste)
```

---

## PASSO 4: Preparar chaves SSH (reutilizar as existentes)

```bash
# Copiar sua chave SSH existente para o diretório AWS
cp ~/.ssh/id_rsa ~/aws-bioinformatica/keys/aws_rsa

# Copiar chave pública também
cp ~/.ssh/id_rsa.pub ~/aws-bioinformatica/keys/aws_rsa.pub

# Verificar
ls -la ~/aws-bioinformatica/keys/

# Confirmar permissões (deve ser 400 ou 600)
chmod 400 ~/aws-bioinformatica/keys/aws_rsa
ls -la ~/aws-bioinformatica/keys/
```

**Saída esperada:**
```
-rw------- aws_rsa
-rw-r--r-- aws_rsa.pub
```

---

## PASSO 5: Criar arquivo de configuração AWS (template)

```bash
cat > ~/aws-bioinformatica/credentials/.env.template << 'EOF'
# Preencher APÓS receber credenciais do IAM (Dia 1)
# NÃO compartilhe este arquivo!

AWS_ACCESS_KEY_ID=seu_access_key_aqui
AWS_SECRET_ACCESS_KEY=sua_secret_key_aqui
AWS_DEFAULT_REGION=sa-east-1
AWS_ACCOUNT_ID=seu_account_id
EOF

# Ver o arquivo
cat ~/aws-bioinformatica/credentials/.env.template
```

---

## PASSO 7: Criar script de teste local

```bash
cat > ~/aws-bioinformatica/scripts/test_setup.sh << 'EOF'
#!/bin/bash

echo "Testando Setup Local AWS..."
echo ""

# Teste 1: AWS CLI
echo "✓ Teste 1: AWS CLI"
aws --version
echo ""

# Teste 2: Chaves SSH
echo "✓ Teste 2: Chaves SSH"
if [ -f ~/.ssh/id_rsa ]; then
    echo "  Chave SSH encontrada"
    ssh-keygen -l -f ~/.ssh/id_rsa.pub
else
    echo "  Chave SSH NÃO encontrada"
fi
echo ""

# Teste 3: Python
echo "✓ Teste 3: Python"
python3 --version
echo ""

# Teste 4: Estrutura de pastas
echo "✓ Teste 4: Estrutura de pastas"
for dir in credentials keys scripts data; do
    if [ -d ~aws-bioinformatica/$dir ]; then
        echo "  Pasta $dir existe"
    else
        echo "  Pasta $dir falta"
    fi
done
echo ""

echo "Setup local verificado!"
EOF

chmod +x ~/aws-bioinformatica/scripts/test_setup.sh
```

---

## PASSO 8: Executar o teste

```bash
~/aws-bioinformatica/scripts/test_setup.sh
```
---

## PASSO 9: Preparar credenciais

Quando a AWS ativar sua conta (24h) e você criar o usuário IAM, você vai receber:
- AWS Access Key ID
- AWS Secret Access Key

```bash
# Editar o arquivo .env
nano ~/aws-bioinformatica/credentials/.env
# (Cole suas credenciais lá)

# Depois, configure no AWS CLI:
aws configure
# Ele vai pedir: Access Key, Secret Key, região (sa-east-1), formato (json)
```

---

## PREPARACAO LOCAL AZURE

Siga estes passos no seu notebook/terminal antes de criar recursos no Azure.

---

## PASSO 1: Verificar se Azure CLI está instalado

```bash
az version
```

Se não encontrar, instale:

```bash
# Ubuntu/Debian
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# macOS
brew update
brew install azure-cli
```

No Windows, use o instalador oficial da Azure CLI ou rode pelo WSL.

---

## PASSO 2: Login no Azure

```bash
az login
```

Se estiver em terminal sem navegador:

```bash
az login --use-device-code
```

Confirmar assinatura ativa:

```bash
az account list --output table
az account show --output table
```

Selecionar assinatura específica:

```bash
az account set --subscription "<SUBSCRIPTION_ID>"
```

---

## PASSO 3: Criar pasta de trabalho para Azure

```bash
mkdir -p ~/azure-bioinformatica/{credentials,keys,scripts,data}
cd ~/azure-bioinformatica

pwd
ls -la
```

**Estrutura esperada:**
```
azure-bioinformatica/
├── credentials/    (templates e anotações locais)
├── keys/           (chaves SSH para VM)
├── scripts/        (scripts Python e bash)
└── data/           (dados de teste)
```

---

## PASSO 4: Criar chave SSH para VM Azure

```bash
ssh-keygen -t rsa -b 4096 \
  -f ~/azure-bioinformatica/keys/azure_bio_rsa \
  -C "azure-bioinformatica"

chmod 400 ~/azure-bioinformatica/keys/azure_bio_rsa

ls -la ~/azure-bioinformatica/keys/
```

**Arquivos esperados:**
```
azure_bio_rsa
azure_bio_rsa.pub
```

Use o conteúdo de `azure_bio_rsa.pub` ao criar a VM pelo portal:

```bash
cat ~/azure-bioinformatica/keys/azure_bio_rsa.pub
```

---

## PASSO 5: Criar arquivo de variáveis Azure

```bash
cat > ~/azure-bioinformatica/credentials/.env.template << 'EOF'
# Template para laboratório Azure
# Ajuste os nomes antes de executar comandos da aula.

AZ_RESOURCE_GROUP=rg-bioinformatica-lab
AZ_LOCATION=brazilsouth
AZ_VM_NAME=vm-bio-lab
AZ_STORAGE_ACCOUNT=stbioseunome2026
AZ_CONTAINER=raw-data
EOF

cat ~/azure-bioinformatica/credentials/.env.template
```

Para usar as variáveis:

```bash
cp ~/azure-bioinformatica/credentials/.env.template ~/azure-bioinformatica/credentials/.env
nano ~/azure-bioinformatica/credentials/.env
source ~/azure-bioinformatica/credentials/.env
```

---

## PASSO 6: Criar script de teste local Azure

```bash
cat > ~/azure-bioinformatica/scripts/test_setup_azure.sh << 'EOF'
#!/bin/bash

echo "Testando Setup Local Azure..."
echo ""

echo "Teste 1: Azure CLI"
az version --output table
echo ""

echo "Teste 2: Login Azure"
az account show --output table
echo ""

echo "Teste 3: Chave SSH"
if [ -f ~/azure-bioinformatica/keys/azure_bio_rsa.pub ]; then
    echo "  Chave SSH pública encontrada"
    ssh-keygen -l -f ~/azure-bioinformatica/keys/azure_bio_rsa.pub
else
    echo "  Chave SSH pública NAO encontrada"
fi
echo ""

echo "Teste 4: Estrutura de pastas"
for dir in credentials keys scripts data; do
    if [ -d ~/azure-bioinformatica/$dir ]; then
        echo "  Pasta $dir existe"
    else
        echo "  Pasta $dir falta"
    fi
done
echo ""

echo "Setup local Azure verificado!"
EOF

chmod +x ~/azure-bioinformatica/scripts/test_setup_azure.sh
```

---

## PASSO 7: Executar o teste Azure

```bash
~/azure-bioinformatica/scripts/test_setup_azure.sh
```

Se `az account show` falhar, rode `az login` novamente.
