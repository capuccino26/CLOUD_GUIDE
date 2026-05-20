# AWS: Guia Prático Executivo

## Segurança (30 minutos)

### Passo 1.1: Criar Usuário IAM (CRÍTICO!)

**Por que:** Nunca use credenciais root para desenvolvimento

1. No console AWS, vá para: **IAM** → **Usuários**
2. Clique em **Criar usuário**
3. Nome: `seunome` (recomendado: seu nome ou função)
4. Marque **Fornecer acesso ao AWS Management Console**
5. Defina senha (salve em local seguro)
6. Clique **Próximo**

**Exemplo de usuário criado:** `seunome`

### Passo 1.2: Dar Permissões Básicas

1. Clique em **Adicionar permissões**
2. Selecione **Anexar políticas existentes diretamente**
3. Procure e marque:
   - `AmazonEC2FullAccess`
   - `AmazonS3FullAccess`
   - `IAMReadOnlyAccess`
4. Clique **Criar usuário**

### Passo 1.3: Gerar Chaves de Acesso (para CLI)

1. Clique no usuário criado (ex: `seunome`)
2. Vá para **Credenciais de segurança**
3. Clique **Criar chave de acesso**
4. Selecione **Application running outside AWS**
5. **COPIE E SALVE EM LOCAL SEGURO:**
   - Access Key ID (ex: `XXXXX`)
   - Secret Access Key (não exponha!)

**Depois não consegue recuperar a Secret Key!**

**Salve em:** `~/aws-bioinformatica/credentials/accessKeys.csv` (não commit no Git!)

---

## EC2 - Sua Primeira Máquina (45 minutos)

### Atividade (AWS Explore): Iniciar uma instância utilizando o EC2

1. No console AWS, abra o painel "Explore a AWS" (geralmente disponível no topo direito do Console).
2. Localize a atividade "Iniciar uma instância utilizando o EC2" e clique em **Start activity** (ou "Iniciar atividade").
3. Siga o passo-a-passo guiado pela atividade — ele irá instruir você a lançar uma instância EC2 rápida e testar conexão básica.
4. Ao concluir a atividade, você receberá o crédito de recompensa (ex.: US$20) indicado no painel. O status da atividade é mostrado no próprio painel "Explore".

Dica de segurança: a atividade pode iniciar uma instância pública. Pare a instância quando terminar ou associe um Elastic IP se quiser manter acesso persistente.

### Passo 2.1: Criar Par de Chaves SSH
 (em "Rede e segurança")
3. Clique **Criar par de chaves**
4. Nome: `training_keys` (ou seu nome preferido)
5. Tipo: **RSA**
6. Formato: **.pem** (Mac/Linux) ou **.ppk** (Windows)
7. Clique **Criar par de chaves**
8. Arquivo será baixado automaticamente
9. **MUDE PERMISSÕES E MOVA:**
```bash
mv ~/Downloads/training_keys.pem ~/aws-bioinformatica/keys/
chmod 400 ~/aws-bioinformatica/keys/training_keys.pem
chmod 400 ~/Downloads/minha-chave-bio.pem
mv ~/Downloads/minha-chave-bio.pem ~/.ssh/
```

### Passo 2.2: Lançar Instância EC2

1. No EC2 → **Instâncias**
2. Clique **Executar instâncias**

**Configurações:**
- **Nome:** `training_lab` (ou seu nome)
- **AMI (imagem):** Amazon Linux 2023 ✓ (Recomendado - Free Tier)
- **Tipo de instância:** `t3.micro` ✓ (Free tier)
- **Par de chaves:** Selecione `training_keys`
- **Rede:** VPC padrão
- **Storage:** 8-30 GB SSD ✓ (Grátis no Free Tier)
- **Firewall (grupos de segurança):** Criar novo
  - Marque: "Permitir tráfego SSH" (0.0.0.0/0)
  - (Depois adicione HTTP 80/443 se precisar Jupyter/web)

Clique **Executar instâncias** e aguarde "running"

### Passo 2.3: Conectar via SSH

1. Instância em "running", clique nela
2. Copie o **IPv4 público** (exemplo: `<SEUIP>`)
3. No terminal (seu notebook):

```bash
# Adicionar aos known_hosts e criar alias SSH
ssh-keyscan -H <SEUIP> >> ~/.ssh/known_hosts 2>/dev/null

# Criar alias no .bashrc para conectar com um comando simples
cat >> ~/.bashrc << 'EOF'
# AWS Training Lab
alias aws-lab="ssh -i ~/aws-bioinformatica/keys/training_keys.pem ec2-user@<SEUIP>"
EOF

source ~/.bashrc

# Conectar:
aws-lab
```

**Responda `yes` se perguntarem sobre adicionar ao known_hosts**

```bash
# Pronto! Você está dentro da EC2
[ec2-user@ip-<IP> ~]$

# Teste:
uname -a
pwd
```

**Dica:** Use o alias `aws-lab` sempre! Para desconectar: `exit`.

## S3 - Armazenando Dados na Nuvem (30 minutos)

### Passo 3.1: Criar um Bucket S3

1. No console AWS, vá para: **S3** → **Criar bucket**
2. Nome do bucket: `seunome-bioinformatica-2026`
3. Região: **South America (São Paulo)**
4. Desmarque "Bloquear todo o acesso público" (para fins de teste)
5. Clique em **Criar bucket**

**Exemplo de bucket criado:** `seunome-bioinformatica-2026` ✅

### Passo 3.2: Fazer Upload de Arquivo no S3

1. Acesse o bucket criado: `seunome-bioinformatica-2026`
2. Clique em **Fazer upload**
3. Escolha o arquivo: `manifest.json` (12.3 KB)
4. Clique em **Fazer upload**

**Resultado:** Upload bem-sucedido! ✅

---

## AWS CLI - Automatizar Tudo (1 hora)

### Passo 4.1: Instalar AWS CLI na EC2

Na sua conexão SSH com EC2:

```bash
# Atualizar sistema
sudo yum update -y

# Para Amazon Linux 2:
sudo yum install -y gcc python3-devel
python3 -m pip install --upgrade pip
pip3 install --user awscli

# Para Ubuntu:
sudo apt update && sudo apt install -y awscli

# Verificar instalação
aws --version
```

### Passo 4.2: Configurar Credenciais

```bash
aws configure

# Confirmar que as credenciais foram salvas corretamente
aws sts get-caller-identity
```

Quando pedir:
- **AWS Access Key ID:** Cole a que você salvou no Dia 1
- **AWS Secret Access Key:** Cole a que você salvou no Dia 1
- **Default region:** `sa-east-1`
- **Default output format:** `json`

Se aparecer `Unable to locate credentials`, rode `aws configure` novamente e confirme que o arquivo `~/.aws/credentials` existe no mesmo usuário da sessão SSH.

### Passo 4.3: Testar S3 via CLI

```bash
# Listar todos seus buckets
aws s3 ls

# Sincronizar pasta local com S3 (dentro de EC2)
echo "Teste de arquivo" > /tmp/teste.txt
aws s3 cp /tmp/teste.txt s3://seunome-bioinformatica-2026/raw-data/

# Verificar upload
aws s3 ls s3://seunome-bioinformatica-2026/raw-data/

# Download de volta
aws s3 cp s3://seunome-bioinformatica-2026/raw-data/teste.txt ~/teste-download.txt
cat ~/teste-download.txt
```

### Passo 4.4: Conectar à Instância EC2 via SSH

1. Certifique-se de que o alias SSH foi configurado no Dia 2.
2. No terminal, use o comando abaixo para se conectar à sua instância EC2:

```bash
aws-lab
```

3. Você será conectado à sua instância EC2. O IP público da instância é o mesmo exibido no console da EC2.

**Resultado:** Conexão bem-sucedida.

---

## Instalar Ferramentas de Bioinformática (2 horas)

**Na sua conexão SSH com EC2:**

### Passo 5.1: Dependências Básicas

```bash
# Atualizar pacotes
sudo yum update -y

# Para Amazon Linux 2:
sudo yum install -y gcc python3-devel

# Para Ubuntu:
sudo apt install -y build-essential python3-dev
```

### Passo 5.2: Python + Bioinformática

```bash
# Verificar instalação
python3 << 'EOF'
from Bio import SeqIO
print("✓ Biopython OK")
import pandas as pd
print("✓ Pandas OK")
EOF
```

### Passo 5.3: Ferramentas do Sistema

```bash
# Samtools (para BAM/SAM)
# Instalado com sucesso: samtools 1.17

# BWA (alinhador)
sudo yum install -y bwa
# ou
sudo apt install -y bwa

# FastQC (qualidade)
cd ~
wget https://www.bioinformatics.babraham.ac.uk/projects/fastqc/fastqc_v0.11.9.zip
unzip fastqc_v0.11.9.zip
chmod +x FastQC/fastqc
export PATH=$PATH:~/FastQC

# Verificar
fastqc --version
samtools --version
bwa
```
Após a instalação, verifique novamente o FastQC:

```bash
fastqc --version
```

Se o comando retornar a versão do FastQC, a instalação foi bem-sucedida.

### Passo 5.4: Instalar e Configurar JupyterLab

Se você deseja usar o JupyterLab para análises interativas, siga os passos abaixo. Este fluxo usa a porta 80 para evitar bloqueios externos em portas não padrão:

1. **Criar um ambiente virtual para o JupyterLab**:
   ```bash
   python3 -m venv ~/jupyter_env
   ```

2. **Ativar o ambiente virtual**:
   ```bash
   source ~/jupyter_env/bin/activate
   ```

3. **Instalar o Jupyter e JupyterLab no ambiente virtual**:
   ```bash
   pip install jupyter jupyterlab
   ```

4. **Liberar a porta 80 no Security Group da EC2**:
   - No console AWS, vá para **EC2** → **Instâncias**.

5. **Ajustar permissões e criar log**:
   - Selecione sua instância e clique em **Grupos de segurança**.
   - Em **Regras de entrada**, adicione:
     - Tipo: **HTTP** ou **Custom TCP Rule**
     - Porta: **80**

6. **Permitir manualmente (arquivo de controle)**:
     - Origem: **0.0.0.0/0** ou seu IP público


7. **Testar manualmente sem desligar (modo DRY-RUN)**:
   ```bash
   sudo /home/ec2-user/jupyter_env/bin/jupyter lab \
     --ip=0.0.0.0 --port=80 --no-browser --allow-root

8. **Acessar o JupyterLab no navegador**:
   - Copie o URL gerado pelo comando acima, que será algo como:
     ```
     http://127.0.0.1/lab?token=<seu_token>
     ```
   - Substitua `127.0.0.1` pelo endereço IP público da sua instância EC2 (exemplo: `<SEUIP>`).
   - O URL final será algo como:
     ```
     http://<SEUIP>/lab?token=<seu_token>
     ```

9. **Ver o token atual quando reiniciar o Jupyter**:
   ```bash
   jupyter server list
   ```

10. **Acessar o JupyterLab**:
   - Abra o URL atualizado no navegador para acessar o JupyterLab.

Se você ainda não conseguir acessar, confirme se o processo está realmente escutando na porta 80 e se o link usado inclui o token atual.

---

## Primeiro Projeto - FASTQ Básico (2 horas)

### Passo 6.1: Criar Script Python

Na EC2, crie arquivo:

```bash
cat > ~/analisar_fastq.py << 'EOF'
#!/usr/bin/env python3

from Bio import SeqIO
import sys
import json

def analisar_fastq(arquivo):
    """Análise básica de arquivo FASTQ"""
    
    reads = []
    qualidades = []
    tamanhos = []
    
    for record in SeqIO.parse(arquivo, "fastq"):
        reads.append(len(record.seq))
        qualidades.append(sum(record.letter_annotations["phred_quality"]) / len(record.letter_annotations["phred_quality"]))
        tamanhos.append(len(record.seq))
    
    resultado = {
        "total_reads": len(reads),
        "tamanho_medio": sum(tamanhos) / len(tamanhos) if tamanhos else 0,
        "tamanho_min": min(tamanhos) if tamanhos else 0,
        "tamanho_max": max(tamanhos) if tamanhos else 0,
        "qualidade_media": sum(qualidades) / len(qualidades) if qualidades else 0
    }
    
    return resultado

if __name__ == "__main__":
    if len(sys.argv) < 2:
        print("Uso: python3 analisar_fastq.py arquivo.fastq")
        sys.exit(1)
    
    resultado = analisar_fastq(sys.argv[1])
    print(json.dumps(resultado, indent=2))
EOF

chmod +x ~/analisar_fastq.py
```

### Passo 6.2: Obter Dados de Teste

```bash
# Baixar pequeno dataset de teste (SRA público)
# Este é um pequeno arquivo (~50 MB)

mkdir -p ~/datasets
cd ~/datasets

# Exemplo: dataset pequeno de teste
# (Usar SRA Toolkit para dados reais)

# OU criar FASTQ fictício para teste:
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

### Passo 6.3: Executar Análise

```bash
cd ~/datasets
python3 ~/analisar_fastq.py teste.fastq

# Resultado esperado:
# {
#   "total_reads": 2,
#   "tamanho_medio": 16.0,
#   ...
# }
```

### Passo 6.4: Salvar Resultados no S3

```bash
# Criar arquivo de resultado
python3 ~/analisar_fastq.py teste.fastq > resultado.json

# Enviar para S3
aws s3 cp resultado.json s3://seunome-bioinformatica-2026/results/

# Verificar
aws s3 ls s3://seunome-bioinformatica-2026/results/
```

Se o upload falhar com `Unable to locate credentials`, volte para o Passo 4.2 e valide `aws sts get-caller-identity` antes de tentar novamente.

---

## Monitoramento e Economia (1 hora)

### Passo 7.1: Configurar Alertas de Faturamento

1. **Conta AWS** (canto superior direito) → **Billing e Cost Management**
2. **Painel de faturamento**
3. Clique em **Gerenciar alertas de cobrança**
4. Clique **Criar alarme**
5. Defina:
   - **Threshold:** R$ 5,00 (por exemplo)
   - **Email:** Seu email

### Passo 7.2: Monitorar CloudWatch

1. **Serviços** → **CloudWatch**
2. **Dashboards** → **Criar dashboard**
3. Adicione widgets:
   - EC2 CPU Utilization
   - Network In/Out
   - EBS Read/Write Operations

### Passo 7.3: Auto-shutdown seguro (cron)

Este guia inclui um script seguro para desligar a instância automaticamente quando ela estiver inativa por 6 horas. O comportamento que testamos durante a sessão é o seguinte:

- O script verifica: uptime (>= 21600s = 6h), nenhum usuário conectado, nenhuma sessão `tmux`/`screen` do `ec2-user` e a existência de um arquivo de controle `/home/ec2-user/auto_shutdown.enable`.
- O cron executa o script no topo de cada hora; quando todas as condições são satisfeitas, o script executa o shutdown.

Instalar e configurar (comandos para colar no terminal):

```bash
# instalar o cron (se necessário)
O FastQC requer o Java para funcionar. Caso o comando `fastqc --version` retorne um erro relacionado ao Java, siga os passos abaixo para instalar a dependência:


# criar o script seguro
```bash
# Instalar o Java (Amazon Linux 2)
sudo yum install -y java-1.8.0-openjdk

# Verificar a instalação do Java
java -version
```

### Passo 7.4: Boas Práticas de Economia

```bash
# SCRIPT PARA DESLIGAR EC2 AUTOMATICAMENTE

# Criar script de shutdown
cat > ~/auto_shutdown.sh << 'EOF'
#!/bin/bash
# Desligar instância após 6 horas de inatividade

INATIVO=21600  # 6 horas em segundos
UPTIME=$(cut -d. -f1 /proc/uptime)

if [ $UPTIME -gt $INATIVO ]; then
    sudo shutdown -h now
fi
EOF

# Agendar para rodar a cada hora
crontab -e
# Adicionar: 0 * * * * ~/auto_shutdown.sh

# OU: Desligar manualmente quando não usar
# No console EC2 → Instâncias → Desligar (NOT terminate)
```

### Passo 7.4: Checklist de Economia

- [ ] Verificar instâncias rodando
- [ ] Desligar EC2 quando sair
- [ ] Não criar volumes EBS extras
- [ ] Limpar snapshots
- [ ] Remover Elastic IPs não usados
- [ ] Monitorar transfer de dados (S3 → fora da AWS = caro!)

---


## ERROS COMUNS A EVITAR

### Erro 1: Esquecer de Desligar EC2
- **Custo:** R$ 30-50/mês
- **Solução:** Configure auto-shutdown ou desligue manualmente

### Erro 2: Criar Múltiplos Security Groups
- **Custo:** Não custa, mas bagunça
- **Solução:** Reutilize um único SG

### Erro 3: Transferir Dados Para Fora da AWS
- **Custo:** R$ 0,92 por GB
- **Solução:** Processe dados dentro da AWS

### Erro 4: Usar Instância Maior que t2.micro
- **Custo:** R$ 20-30/mês
- **Solução:** Mantenha t2.micro ou use Lambda

### Erro 5: Não Configurar Alertas
- **Custo:** Surpresa na fatura
- **Solução:** Configure alertas de R$ 1-5/mês

### Erro 1: Esquecer de Desligar EC2
- **Custo:** R$ 30-50/mês
- **Solução:** Configure auto-shutdown ou deslige manualmente

### Erro 2: Criar Múltiplos Security Groups
- **Custo:** Não custa, mas bagunça
- **Solução:** Reutilize um único SG

### Erro 3: Transferir Dados Para Fora da AWS
- **Custo:** R$ 0,92 por GB
- **Solução:** Processe dados dentro da AWS

### Erro 4: Usar Instância Maior que t2.micro
- **Custo:** R$ 20-30/mês
- **Solução:** Mantenha t2.micro ou use Lambda

### Erro 5: Não Configurar Alertas
- **Custo:** Surpresa na fatura
- **Solução:** Configure alertas de R$ 1-5/mês

---

## Dúvidas Técnicas

**P: Perdi minha Secret Key. E agora?**
R: Crie uma nova par de chaves no console IAM. Delete a antiga.

**P: EC2 ficou lento. O que fazer?**
R: t2.micro tem créditos de CPU. Use ou aguarde renovação mensal.

**P: Como faço SSH do Windows?**
R: Use PuTTY (converta .pem para .ppk) ou Windows Terminal + WSL.

**P: Posso usar t2.small ou t3.micro?**
R: t2.micro é grátis. t2.small = R$ 15-20/mês. t3.micro = R$ 8-10/mês.

**P: Quanto de egress posso usar?**
R: 1 GB/mês grátis. Acima disso = R$ 0,92/GB (MUITO CARO!)

---

## Desligar a instância pelo CLI e sair

Se você está conectado por SSH e terminou o trabalho, pode desligar a instância imediatamente e sua sessão SSH será encerrada automaticamente.

- Desligar a partir da própria instância (SSH):
```bash
sudo shutdown -h now
```
Este comando pede ao sistema que pare; a conexão SSH será fechada e a instância passará para o estado "stopped" (se o comportamento padrão for "stop").

- Se preferir usar o AWS CLI (de sua máquina local ou do CloudShell) para parar a instância:
1) Obtenha o InstanceId (se não souber):
```bash
curl -s http://169.254.169.254/latest/meta-data/instance-id
```
2) Pare a instância (substitua `i-XXXX`):
```bash
aws ec2 stop-instances --instance-ids i-XXXX
```

Observações:
- Ao reiniciar a instância, o endereço IP público pode mudar a menos que você use um Elastic IP.
- Verifique se o comportamento de "shutdown" da instância está configurado para "stop" e não para "terminate" para evitar exclusão acidental.

## Reiniciar a instância e reconectar (com placeholder para novo IP)

Se você desligou a instância e quer reiniciá-la e reconectar por SSH, siga estes comandos diretos (substitua `i-XXXX` pelo `InstanceId` da sua instância):

- Iniciar a instância via AWS CLI:
```bash
aws ec2 start-instances --instance-ids i-XXXX
aws ec2 wait instance-running --instance-ids i-XXXX
```

- Obter o novo IP público e reconectar (substitua `i-XXXX`):
```bash
NEW_IP=$(aws ec2 describe-instances --instance-ids i-XXXX --query 'Reservations[].Instances[].PublicIpAddress' --output text)
echo "New public IP: $NEW_IP"
ssh -i ~/aws-bioinformatica/keys/training_keys.pem ec2-user@${NEW_IP}
```

Se você prefere usar o Console, vá em **EC2 → Instâncias → selecione a instância → Instance state → Start** e, quando o estado for `running`, copie o `IPv4 público` e conecte com o comando SSH acima, substituindo `<NEW_PUBLIC_IP>`.

Nota: se quiser um IP público estável entre reinícios, atribua um Elastic IP à instância.
