# Servidor Wordpress via Docker em AWS
![Screenshot_5](https://github.com/vitortoniolo/PB-AtividadeDocker/assets/133904035/cf3155bd-4aed-4331-83d2-bee70cf990ef)
## Sumário
wip

## 1. Requisitos
- Instalação e configuração do DOCKER ou CONTAINERD no
host EC2
  - Ponto adicional para o trabalho utilizar a instalação via script de
Start Instance (user_data.sh)
- Efetuar Deploy de uma aplicação Wordpress com:
  - Container de aplicação
  - RDS database Mysql
- Configuração da utilização do serviço EFS AWS para estáticos
do container de aplicação Wordpress
- Configuração do serviço de Load Balancer AWS para a aplicação
Wordpress
- Configurar Auto Scaling Group para as duas instâncias.

## Configurações 
### VPC e Subnets
Crie uma nova VPC, nomeando-a (ex: Wordpress) e decidindo o bloco IPv4 CIDR (ex: 172.28.0.0/16)

Agora navegue para subnets e crie conforme o seguinte:
- Subnet pública
  - Nome: `sub-public01`
  - AZ: `us-east-1a`
  - IPv4 CIDR: `172.28.0.0/24`
  - Após criar, selecione as configurações e cheque a caixa de `Enable auto-assign public IPv4 address`
- Subnet pública 2
  - Nome: `sub-public02`
  - AZ: `us-east-1b`
  - IPv4 CIDR: `172.28.1.0/24`
  - Após criar, selecione as configurações e cheque a caixa de `Enable auto-assign public IPv4 address`
- Subnet privada 1
  - Nome: `sub-private01`
  - AZ: `us-east-1a`
  - IPv4 CIDR: `172.28.2.0/24` (ou o equivalente para sua VPC)
- Subnet privada 2
  - Nome: `sub-private02`
  - AZ: `us-east-1b`
  - IPv4 CIDR: `172.28.3.0/24`
  
Agora vá para a aba NAT gateways e crie um novo gateway:
- Nome: `NAT01`
- Subnet: `sub-public01`
- Connectivity type: `Public`
- Ellastic IP Allocation ID: `Allocate Elastic IP`
  
Navegue para a aba de route tables e crie uma route table com nome `rt-public` e outra `rt-private`

- Selecione a `rt-private` e clique em `Subnet associations > Edit Subnet associations`:
  - Associe com as subnets privadas. Faça o mesmo mas com a `rt-public` (associando com a subnet publica)
- Selecione `rt-public` e vá em `Routes > Edit Routes`. Adicione uma nova rota:
  - Destination: `0.0.0.0/0` 
  - Target: `Internet Gateway`
- Selecione `rt-private` e vá em `Routes > Edit Routes`. Adicione uma nova rota:
  - Destination: `0.0.0.0/0` 
  - Target: `NAT Gateway` (NAT01)

### Bastion Host
Crie uma instância com as seguintes configurações:
- Imagem: `Amazon Linux 2`
- Tipo: `t2.micro`
- VPC: `Wordpress`
- Subnet: `sub-public01`
- Auto-associação de endereço IP: `Ativo`
- Security group:
  Tipo | Protocolo | Intervalo de portas | Origem
  ------------- | ------------- | ------------- | -------------
  SSH | TCP | 22 | `Seu IP`

### Instância privada para o Wordpress
Crie uma instância com as seguintes configurações:
- Imagem: `Amazon Linux 2`
- Tipo: `t3.small`
- VPC: `Wordpress`
- Subnet: `sub-private01`
- Armazenamento: `20 GB gp3`
- Security group:
  Tipo | Protocolo | Intervalo de portas | Origem
  ------------- | ------------- | ------------- | -------------
  SSH | TCP | 22 | `IP privado do seu Bastion host`
  HTTP | TCP | 80 | 0.0.0.0/0
  HTTPS | TCP | 443 | 0.0.0.0/0
  NFS | TCP | 2049 | `Bloco CIDR do EFS`

###Configurando a instância para acesso:



### EFS
Crie um novo EFS, selecionando a VPC criada anteriormente.

Crie um security group para o EFS:
  Tipo | Protocolo | Intervalo de portas | Origem
  ------------- | ------------- | ------------- | -------------
  NFS | TCP | 2049 | `Security group da instância WP`

Abra as configurações do EFS e vá para a aba `Network`. Mude os security groups para o recém criado.

Acessando a instância pelo Bastion Host, execute `sudo mkdir /mnt/nfs`. Para montar o NFS, clique em `Attach` na tela do mesmo e copie o comando de `Mount via IP`, certifique-se de mudar o caminho de montagem.

### RDS
Primeiramente, crie um novo security group para o RDS:
  Tipo | Protocolo | Intervalo de portas | Origem
  ------------- | ------------- | ------------- | -------------
  MYSQL | TCP | 3306 | `Bloco CIDR do VPC`

Crie um novo rds, com as seguintes configurações:
- Engine type: `MySQL`
- Templates: `Free tier`
- DB instance identifier: `wordpressDB`
- VPC: `Wordpress`
- Public Access: `Yes`
- VPC Security Group: `SG criado anteriormente`
- Informações de login: `Seu critério`
- Initial databasename: `wpDB`

### Load Balancer
- Primeiro, crie um target group:
  - Tipo: `Instâncias`
  - VPC: `wordpress`
  - Registre a instância do wordpress
- Agora um security group:
   Tipo | Protocolo | Intervalo de portas | Origem
  ------------- | ------------- | ------------- | -------------
  HTTP | TCP | 80 | `0.0.0.0/0`
- E finalmente, o load balancer:
  - Tipo: `Application Load Balancer`
  - Scheme: `Internet-facing`
  - VPC: `wordpress`
  - Mappings: `Marque as duas zonas e selecione as subnets publicas`
  - Security groups: `Selecione o grupo criado para o LB`
  - Listener: `Selecione o target group criado`
  

