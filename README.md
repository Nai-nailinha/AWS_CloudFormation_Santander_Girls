
# Lab — Infra como Código com AWS CloudFormation

> “Automatizar infraestrutura é tipo montar LEGO… só que com YAML e cafeína.” ☕🧱

Este repositório contém um template **CloudFormation** que provisiona uma stack de estudo com:

- **VPC** com DNS habilitado  
- **2 Subnets Públicas** (AZ 1 e AZ 2)  
- **Internet Gateway + Route Table**  
- **Security Group** (HTTP/HTTPS e SSH opcional)  
- **EC2 Amazon Linux 2023** com **NGINX** via *UserData* (SSM habilitado)  
- **S3** com criptografia **SSE-KMS (AWS managed)** e **bloqueio de acesso público**  
- **IAM Role + Instance Profile** para **SSM Session Manager**

> ⚠️ **Aviso de custos:** recursos como EC2, NAT/IGW e transferência de dados podem gerar cobrança. Use instâncias pequenas (ex.: `t3.micro`) e **exclua a stack** quando terminar.


## Arquitetura (visão rápida)

```
Internet
   │
 ┌─┴───────────────┐
 │  Internet GW    │
 └─┬───────────────┘
   │           (default route 0.0.0.0/0)
┌──┴───────────────┐
│ Public RouteTable│
└──┬───────────┬───┘
   │           │
┌──▼───┐   ┌───▼──┐
│Sub-a │   │Sub-b │ (ambas públicas c/ MapPublicIpOnLaunch)
└──┬───┘   └───┬──┘
   │           │
   └──EC2 (NGINX) + SG (80/443 + SSH opcional)
```   
S3 Bucket (SSE-KMS, Block Public Access)

## Pré-requisitos

- Conta AWS com permissão para **CloudFormation, EC2, VPC, S3 e IAM**
- **AWS CLI** configurada (`aws configure`)
- (Opcional) **Key Pair** se quiser SSH. Caso contrário, use **Session Manager** (SSM).

## Conteúdo do repositório
```yaml
📁 AWS_CloudFormation_Santander_Girls
┣ 📜 template.yaml # Template principal (VPC + EC2 + S3)
┣ 📄 README.md # Este arquivo
┗ 📁 imagens/ # (Opcional) prints da execução/recursos
```

## Parâmetros do template

- `VpcCidr` — CIDR da VPC (padrão `10.0.0.0/16`)
- `PublicSubnet1Cidr` — Subnet pública 1 (padrão `10.0.1.0/24`)
- `PublicSubnet2Cidr` — Subnet pública 2 (padrão `10.0.2.0/24`)
- `InstanceType` — tipo da instância EC2 (padrão `t3.micro`)
- `KeyPairName` — (opcional) nome do Key Pair para SSH
- `AllowedSSHLocation` — (opcional) CIDR para SSH (ex.: `203.0.113.4/32`)

> 🔐 **Boa prática:** se não precisar de SSH, deixe `KeyPairName` vazio e acesse a instância pelo **SSM Session Manager**.

## Deploy — via AWS CLI (recomendado)

1) **Crie** a stack:

```bash
aws cloudformation create-stack \
  --stack-name lab-cfn \
  --template-body file://template.yaml \
  --capabilities CAPABILITY_NAMED_IAM
```

> Se for usar SSH: adicione `--parameters ParameterKey=KeyPairName,ParameterValue=SEU_KEYPAIR`.

2) **Aguarde a conclusão**:

```bash
aws cloudformation wait stack-create-complete --stack-name lab-cfn
```

3) **Veja os *outputs*** (para IP público, IDs, etc.):

```bash
aws cloudformation describe-stacks --stack-name lab-cfn
--query "Stacks[0].Outputs"
```

## Deploy — via Console (passo a passo)

1. Acesse **CloudFormation** → **Create stack** → *With new resources (standard)*  
2. Em *Template source*, selecione **Upload a template file** e envie `template.yaml`  
3. Dê um **Stack name** (ex.: `lab-cfn`)  
4. Ajuste **Parameters** conforme necessário (ex.: KeyPair, SSH CIDR)  
5. Avance → marque **CAPABILITY_NAMED_IAM** → **Create stack**  
6. Acompanhe até o status **CREATE_COMPLETE**  

## Validando a stack

- **EC2 / Instances** → verifique se a instância está `running`  
- **Abra o IP público** no navegador → deve exibir página NGINX com mensagem “Olá, AWS CloudFormation! 🚀”  
- **S3 / Buckets** → bucket criado com **Block Public Access** e **SSE-KMS**  
- **VPC** → confirme a VPC, subnets públicas, IGW e route table padrão `0.0.0.0/0`  

> Sem SSH? Use **SSM**: EC2 → instância → *Connect* → **Session Manager** → *Connect*.

## Limpeza (evitar custos)

```bash
aws cloudformation delete-stack --stack-name lab-cfn
aws cloudformation wait stack-delete-complete --stack-name lab-cfn
```

## Problemas comuns & soluções rápidas

- **`ROLLBACK_IN_PROGRESS`**: algo falhou (permissão/limite/param). Veja **Events** da stack para a causa.  
- **Não abre a página**:  
  - Confirme **SG** (porta 80 aberta)  
  - Confirme **Public IP/Route** na subnet  
  - Confira o **UserData** (NGINX iniciado)  
- **SSH não conecta**:  
  - Verifique `KeyPairName` existe na região  
  - Ajuste `AllowedSSHLocation` para seu IP (ex.: `x.x.x.x/32`)  
  - Veja se o SG contém a regra **22/tcp**  

## Próximos passos (para brilhar mais ✨)

- Colocar um **ALB** na frente do EC2  
- Migrar EC2 para **subnets privadas** + **NAT Gateway**  
- Criar **Auto Scaling Group** + **Launch Template**  
- Usar **Change Sets** antes de aplicar alterações  
- Separar templates (VPC, Compute, Storage) e **aninhá-los** (nested stacks)

## Sobre

Feito com ☕, nuvem e um toque de humor por **Enaile (Criadora do TI com Café & Neurônios - ticomcafeeneuronios.com.br)**.  
- LinkedIn: **https://www.linkedin.com/in/enailelopes**  
- Projeto: **TI com Café & Neurônios** — tech real, acessível e com memes inteligentes - http://ticomcafeeneuronios.com.br.
