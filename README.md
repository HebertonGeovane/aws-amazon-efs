# aws-amazon-efs

### Este laboratório foi desenvolvido como parte do grupo de estudos Run as Cloud  e tem como objetivo criar e testar um sistema de arquivos Amazon EFS utilizando apenas recursos elegíveis ao Free Tier da AWS.

### O Amazon EFS (Elastic File System) é um serviço de armazenamento em nuvem da AWS (Amazon Web Services) que fornece um sistema de arquivos elástico e escalável, acessível simultaneamente por múltiplas instâncias do Amazon EC2 (e outros serviços compatíveis).

---
## ⚠️ Avisos antes de começar

- Todos os recursos utilizados neste guia são elegíveis ao Free Tier da AWS.
- Verifique periodicamente o [AWS Billing Dashboard](https://console.aws.amazon.com/billing/home) para garantir que você não está gerando custos inesperados.
- Remova os recursos ao final do laboratório.
- **Nunca compartilhe prints com IDs de conta, IPs privados ou informações sensíveis.**

---
## ✅ Pré-requisitos

- Conta na AWS com acesso à criação de recursos EC2, VPC, EFS, IAM e CloudWatch
- Acesso via navegador ao Console da AWS
- Git Bash (Windows/macOS/Linux) instalado
- Conhecimento básico de terminal/linha de comando

---
## 🧱 Estrutura do laboratório

1. Criar grupo de segurança para EFS
2. Criar sistema de arquivos EFS
3. Conectar à instância EC2 via Git Bash
4. Montar EFS na instância
5. Avaliar o desempenho com `fio`
6. Monitorar no Amazon CloudWatch

---

## ⚙️ Passo Prévio: Criar o grupo de segurança `EFSClient` e Instância `EC2`

Antes de iniciar a Tarefa 1, é necessário criar o grupo de segurança `EFSClient`, que será usado para permitir o tráfego NFS entre as instâncias EC2 e o Amazon EFS.

1. Acesse o **Console de Gerenciamento da AWS**
2. No menu **Serviços**, selecione **EC2**
3. No painel à esquerda, clique em **Grupos de segurança**
4. Clique em **Criar grupo de segurança**
   - **Nome do grupo de segurança**: `EFSClient`
   - **Descrição**: `Inbound NFS access from EFS clients`
   - **VPC**: selecione a mesma VPC onde estão suas instâncias EC2 
5. Em **Regras de entrada**, clique em **Adicionar regra**
   - **Tipo**: `NFS`
   - **Origem**: `Personalizado`
   - **Valor**: (Cole aqui o ID do grupo de segurança das instâncias EC2 ou outro grupo confiável)
6. Clique em **Criar grupo de segurança**

📸 *Adicione aqui um print da tela de criação do grupo de segurança `EFSClient`*

![image](https://github.com/user-attachments/assets/b982977b-dca1-49c7-a9cf-adca1ed35229) 

##  Criar instância EC2
- Vá para **EC2 > Instâncias**
- Clique em **Executar instância**
- Configure:
  - AMI: Amazon Linux 2 (ou equivalente)
  - Tipo: `t2.micro` (para testes gratuitos)
  - VPC/Sub-rede: mesma VPC do EFS
  - Grupo de segurança: selecione `EFSClient` e seu Grupo de segurança das instâncias `EC2` para conexão via SSH
  - Par de chaves: selecione ou crie um para conexão via SSH continuaremos com o exemplo EC2-Tutorial

📸 Adicione aqui um print da tela de criação da Intância `EC2`

![image](https://github.com/user-attachments/assets/e6d7ffed-6a91-465a-9e66-0c9822ef547f)


![image](https://github.com/user-attachments/assets/50287a6f-ea9d-406c-a188-a558483fb842)


---
## 🛡️ Tarefa 1: Criar grupo de segurança para o EFS

1. Acesse **EC2 > Grupos de segurança**
2. Localize o grupo de segurança `EFSClient` e copie seu **ID** (ex: `sg-xxxxxxxxxxxxxx`)
3. Clique em **Criar grupo de segurança**
   - Nome: `EFS Mount Target`
   - Descrição: `Inbound NFS access from EFS clients`
   - VPC: selecione sua VPC 
4. Na aba **Regras de entrada**, clique em **Adicionar regra**
   - Tipo: `NFS`
   - Origem: `Personalizado` → Cole o ID do grupo `EFSClient`
5. Clique em **Criar grupo de segurança**

📸 *Adicione aqui um print da tela de criação do grupo de segurança*

![image](https://github.com/user-attachments/assets/4b396480-a466-4083-a5a0-2efa05514aa7)

---

## 📁 Tarefa 2: Criar o sistema de arquivos EFS

1. Acesse **Serviços > EFS > Criar sistema de arquivos**
2. Clique em **Personalizar**
3. Na Etapa 1:
   - Desmarque **Habilitar backups automáticos**
   - Gerenciamento do ciclo de vida: `Nenhum`
   - Tags: `Name: My First EFS File System`
4. Etapa 2:
   - Selecione a VPC do laboratório
   - Desmarque os grupos de segurança padrão de cada destino de montagem
   - Associe o grupo `EFS Mount Target`
5. Avance pelas etapas e clique em **Criar**

📸 *Adicione aqui um print da tela com os destinos de montagem associados ao SG correto*

-  Etapa 1

![image](https://github.com/user-attachments/assets/b8f337b0-da24-47a2-b5a9-83f1c776e307)

- desmarque Enable encryption of data at rest
  

![image](https://github.com/user-attachments/assets/3ef7f8dc-dbbd-44a7-95c5-b8ad324d359c)

-  Etapa 2

![image](https://github.com/user-attachments/assets/7e0a80af-6cb8-4b72-94ed-81f560c3c64b)

- Amazon EFS File systems
  
![image](https://github.com/user-attachments/assets/1a17d6c1-6d4b-4d0b-af19-b474f8e0b6fb)

---

## 🔐 Tarefa 3: Conectar à instância EC2 via SSH usando Git Bash (Windows, macOS e Linux)

### 🧰 Requisitos:

- Ter o **Git Bash** instalado: https://git-scm.com/
- Ter baixado sua chave privada exmplo `EC2-Tutorial.pem` da instância EC2
- Ter anotado o **IPv4 Public IP** da sua instância EC2 (console da AWS)

### 👣 Passos:

1. Abra o **Git Bash**
2. Navegue até o diretório onde está o arquivo `.pem`:

```bash
cd ~/Downloads  # Ou o diretório onde o EC2-Tutorial.pem foi salvo
```
3. Dê permissão ao arquivo PEM
 ```
 chmod 400 EC2-Tutorial.pem
 ```
4. Conecte à instância EC2 via SSH (substitua <EC2PublicIP>)
 ```
ssh -i  EC2-Tutorial.pem ec2-user@<EC2PublicIPv4DNS>
 ```
5. Digite `yes` para confirmar a conexão na primeira vez

📸 Adicione aqui um print da conexão bem-sucedida via Git Bash

![image (99)](https://github.com/user-attachments/assets/3f1ae7fa-5b3d-4319-8d47-9e9e9ed6bd39)

---

## 📂 Tarefa 4: Montar o EFS na instância EC2

1. crie um diretório para o EFS
```
sudo mkdir /efs
```

2. No Console AWS, vá em EFS > My First EFS File System > Attach

3. Copie o comando da seção `Usar o cliente NFS`

4. Cole no terminal do Git Bash (exemplo):
```
sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport fs-07e19d035c1c9daa9.efs.us-east-1.amazonaws.com:/ efs
```
5. Verifique se foi montado corretamente
```
df -h
```

📸 Adicione aqui um print do terminal com o comando de montagem e o diretório /efs visível


![image](https://github.com/user-attachments/assets/3e4cc15a-1ee4-4228-9ee6-fe5ce6dad021)

---
## ⚙️ Tarefa 5: Avaliar desempenho do EFS com `fio`

1.  Instalar o `fio`
   
```
sudo yum install -y fio
```

2. Execute o seguinte comando na instância
```
sudo fio --name=fio-efs --filesize=10G --filename=./efs/fio-efs-test.img --bs=1M --nrfiles=1 --direct=1 --sync=0 --rw=write --iodepth=200 --ioengine=libaio

```

3. Aguarde 5–10 minutos. Analise o resultado da escrita no EFS.

📸 Adicione aqui um print do resultado do fio

- Instalando `fio`
  
![image](https://github.com/user-attachments/assets/057ba350-4327-44d7-8a58-b0a1e5e60bbe)

- Avaliar desempenho
  
![image](https://github.com/user-attachments/assets/ecc927d1-f353-4bcf-88bf-71b195f94bc2)

---

## 📊 Tarefa 6: Monitorar métricas no Amazon CloudWatch
Vá para Serviços > CloudWatch > Métricas > EFS > File System Metrics

1. Visualize `PermittedThroughput`

2. Depois, visualize `DataWriteIOBytes`

3. Configure o período para `1 minuto`, a estatística como `Sum`

📸 Adicione aqui prints dos gráficos de desempenho

- PermittedThroughput
  
![image](https://github.com/user-attachments/assets/19a7162d-bbfd-424d-8b68-d19f8d3ae625)

- DataWriteIOBytes
  
  Graphed metrics (1)

![image](https://github.com/user-attachments/assets/6c455f2d-dbbc-4cc9-ab24-d49481e847b4)

---

## 🧹 Limpeza dos recursos

- Exclua a instância EC2

- Exclua o sistema de arquivos EFS

- Remova os grupos de segurança criados

- Verifique o painel de billing

---
##  Conclusão - AWS Amazon EFS 
Neste laboratório prático conduzido como parte do grupo de estudos Run as Cloud, aprendemos a criar, configurar, montar e testar um sistema de arquivos usando o Amazon Elastic File System (EFS), um serviço escalável de armazenamento em rede da AWS.

- O que aprendemos hoje:
  
✅ Criar grupos de segurança personalizados para permitir tráfego NFS entre EC2 e EFS com regras adequadas.

✅ Criar e configurar um sistema de arquivos Amazon EFS usando somente recursos elegíveis ao Free Tier, com pontos de montagem em múltiplas zonas de disponibilidade.

✅ Conectar-se à instância EC2 via SSH usando Git Bash, gerenciando permissões corretamente com .pem.

✅ Montar o EFS na instância EC2 e validar a conexão usando comandos como df -h.

✅ Instalar e executar o utilitário fio para medir o desempenho do EFS em operações de gravação com um arquivo de 10 GB.

✅ Monitorar métricas no Amazon CloudWatch, como PermittedThroughput e DataWriteIOBytes, para analisar o comportamento em tempo real durante a execução de carga.

📈 Resultados:

- O teste com fio mostrou que o EFS foi capaz de atingir uma média de ~121 MiB/s de throughput em escrita.

A monitoração pelo CloudWatch confirmou o volume de dados gravados e o throughput permitido, reforçando a visibilidade e controle operacional sobre o desempenho do sistema.
## 📢 Compartilhe seu progresso!

Marque a comunidade **Run as Cloud** no LinkedIn:  
🔗 [Run as Cloud no LinkedIn](https://www.linkedin.com/company/run-as-cloud/?viewAsMember=true)

**Organizadores:**

- [Heberton Geovane](https://www.linkedin.com/in/heberton-geovane/)
- [Maik Biazi](https://www.linkedin.com/in/maik-biazi-47b9291a5/)
- [Michel Ernesto](https://www.linkedin.com/in/mernesto/)



