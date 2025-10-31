# 🚀 Projeto: Infraestrutura Web Escalável com AWS CloudFormation

Este repositório contém um template AWS CloudFormation (IaC - Infraestrutura como Código) para provisionar uma arquitetura web de alta disponibilidade e escalável, utilizando um Application Load Balancer (ALB) e um Auto Scaling Group (ASG).

Este projeto é a implementação automatizada do "Laboratório - Auto Scaling e Elastic Load Balancer"
<img width="880" height="282" alt="cloudformationinfra" src="https://github.com/user-attachments/assets/cb107f0b-62e4-46c1-93e3-c8a4e7f8b3ce" />
<img width="1887" height="807" alt="Captura de tela 2025-10-31 083243" src="https://github.com/user-attachments/assets/e82c3170-f928-4548-b173-ca4ca2dbb5dc" />
<img width="1697" height="676" alt="teste1" src="https://github.com/user-attachments/assets/526f85fb-8d2b-4cdc-8e7e-bfe41b622d55" />
<img width="1608" height="521" alt="test 2" src="https://github.com/user-attachments/assets/732e89b2-a877-4d44-9136-08c32122c52a" />

## 🎯 O Problema Resolvido

No cenário de uma aplicação web tradicional implantada em um único servidor (uma única instância EC2), existem dois problemas críticos:

1.  **Ponto Único de Falha (SPOF):** Se essa única instância falhar (seja por um problema de hardware, software ou rede), a aplicação inteira fica offline.
2.  **Falta de Escalabilidade:** Durante picos de acesso, o servidor fica sobrecarregado, resultando em lentidão ou falhas para os usuários.

Este projeto resolve esses problemas implementando uma arquitetura resiliente e distribuída.

## 🛠️ A Solução: Infraestrutura Provisionada

Este template CloudFormation provisiona uma arquitetura que garante **Alta Disponibilidade (HA)** e **Resiliência**.

Em vez de um único servidor, a infraestrutura distribui a carga de tráfego entre múltiplas instâncias EC2, que são implantadas em diferentes Zonas de Disponibilidade (Sub-redes).

**Alta Disponibilidade:** O **Application Load Balancer (ALB)** atua como o ponto de entrada único e distribui o tráfego de forma inteligente entre as instâncias saudáveis.
* Se uma instância falhar, o ALB para de enviar tráfego para ela, e os usuários não percebem a falha.
**Resiliência e Elasticidade:** O **Auto Scaling Group (ASG)** garante que um número mínimo (neste caso, 2) de instâncias esteja sempre em execução. Ele monitora a saúde das instâncias e, se uma falhar, o ASG a encerra automaticamente e lança uma nova para substituí-la, garantindo a auto-recuperação do ambiente.

---

## ⚙️ Recursos Provisionados

O template `lab-asg-alb.yaml` (a versão genérica) cria os seguintes recursos:

### 1. Rede e Segurança (Security)

* `AWS::EC2::SecurityGroup` (ALBSecurityGroup):
    * Um Security Group para o Load Balancer.
    * Permite tráfego de entrada (Inbound) da Internet (`0.0.0.0/0`) nas portas **HTTP (80)**.
* `AWS::EC2::SecurityGroup` (InstanceSecurityGroup):
    * Um Security Group para as instâncias EC2.
    * Permite tráfego de entrada na porta **HTTP (80)** *apenas* do Security Group do ALB (melhor prática de segurança).
    * Permite tráfego na porta **SSH (22)** da Internet (`0.0.0.0/0`) para fins de depuração.

### 2. Computação (Compute)

* `AWS::EC2::LaunchTemplate` (Modelo de Lançamento):
    * Define a configuração das instâncias que o ASG irá lançar.
    * **AMI:** Utiliza uma busca dinâmica no AWS SSM Parameter Store para sempre encontrar a **Amazon Linux 2 AMI (x86_64)** mais recente.
    * **Instance Type:** `t2.micro` (qualificado para o Free Tier).
    * **User Data:** Um script de inicialização que é executado quando a instância é ligada. Este script:
        1.  Atualiza os pacotes (`yum update -y`).
        2.  Instala o servidor web Apache (`yum install -y httpd`).
        3.  Inicia o serviço Apache (`systemctl start httpd`).
        4.  Cria uma página `index.html` simples que exibe o nome do host interno da instância (ex: `ip-172-31-X-X`).

### 3. Balanceamento e Escalabilidade (Load Balancing & Scaling)

* `AWS::ElasticLoadBalancingV2::LoadBalancer` (ALB):
    * O balanceador de carga público ("internet-facing") que recebe todo o tráfego dos usuários.
* `AWS::ElasticLoadBalancingV2::TargetGroup`:
    * Define o grupo de "alvos" (as instâncias EC2) para onde o ALB deve enviar o tráfego.
    * Configura os *Health Checks*: o ALB verifica se as instâncias estão saudáveis acessando o caminho `/` via HTTP.
* `AWS::ElasticLoadBalancingV2::Listener`:
    * "Ouve" o tráfego na porta **HTTP (80)** do ALB e o encaminha (`forward`) para o `TargetGroup`.
* `AWS::AutoScaling::AutoScalingGroup` (ASG):
    * O "cérebro" da operação.
    * Utiliza o `LaunchTemplate` para criar as instâncias.
    * Garante que a **Capacidade Desejada/Mínima/Máxima** seja sempre de **2 instâncias**.
    * Registra automaticamente as novas instâncias no `TargetGroup` do ALB.
    * Utiliza os *Health Checks* do ELB para determinar a saúde das instâncias.

---

## 🚀 Como Usar

### Pré-requisitos

Antes de implantar, você precisa ter:
1.  O ID da sua VPC padrão (ou qualquer VPC que você queira usar).
2.  Os IDs de pelo menos duas **Sub-redes Públicas** (em Zonas de Disponibilidade diferentes) dentro dessa VPC.

### 1. Implantação (Deploy) via Console AWS

1.  Faça login no **Console AWS** e vá para o serviço **CloudFormation**.
2.  Clique em **"Criar pilha"** (Create stack) e selecione **"Com novos recursos (padrão)"**.
3.  Em **"Especificar modelo"**, escolha **"Carregar um arquivo de modelo"** e suba o arquivo `.yaml` deste repositório.
4.  Dê um nome para a Pilha (ex: `meu-lab-web`).
5.  Preencha os **Parâmetros** solicitados:
    * `ProjectName`: O nome do seu projeto (ex: `Jessica-App`).
    * `VpcId`: O ID da sua VPC (ex: `vpc-06d4c1654675a2e03`).
    * `PublicSubnetIds`: As IDs das suas duas sub-redes, separadas por vírgula (ex: `subnet-0259f50ab1d5b2f7e,subnet-06c21fba49af32edc`).
    * `LatestAmiId`: Pode deixar o valor padrão (ele buscará a AMI mais recente).
6.  Clique em "Avançar", "Avançar" e, na última página, marque a caixa de confirmação de "Recursos IAM" (embora este template não crie recursos IAM, é uma boa prática).
7.  Clique em **"Enviar"**. A criação levará alguns minutos.

### 2. Testando a Aplicação

1.  Após o status da pilha mudar para **`CREATE_COMPLETE`**.
2.  Clique na pilha e vá para a aba **"Saídas" (Outputs)**.
3.  Copie o valor do `LoadBalancerDNSName`.
4.  Cole essa URL no seu navegador. Você verá a mensagem "Servidor Web - Instância: ip-XXX-XX-X-X".
5.  Ao atualizar a página (pode ser necessário `Ctrl+Shift+R`), você verá o nome do host mudar, provando que o Load Balancer está distribuindo o tráfego entre as duas instâncias.
<img width="1887" height="807" alt="Captura de tela 2025-10-31 083243" src="https://github.com/user-attachments/assets/30c98da7-8ef3-4d0a-839d-e1518f830416" />


### 3. Limpeza (Cleanup)

Para evitar cobranças, é crucial excluir todos os recursos criados.

1.  Vá para o console do **CloudFormation**.
2.  Selecione a pilha que você criou.
3.  Clique em **"Excluir" (Delete)**.

O CloudFormation irá encerrar e excluir automaticamente todos os recursos criados na ordem correta (ASG, ALB, Instâncias, Security Groups, etc.).
