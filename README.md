# Servidor Wordpress via Docker em AWS
![Screenshot_5](https://github.com/vitortoniolo/PB-AtividadeDocker/assets/133904035/cf3155bd-4aed-4331-83d2-bee70cf990ef)
## Sumário
wip

## 1. Requisitos
wip

## 2. Configurações 
### 2.x VPC e Subnets
Para VPC, podemos manter a padrão, porém iremos criar 3 subnets adicionais.
Navegue para o dashboard de VPC e selecione a aba de subnets para criar mais.

- Subnet privada 1
  - Nome: `sub-private01`
  - AZ: `us-east-1a`
  - IPv4 CIDR: `172.31.0.0/24` (ou o equivalente para sua VPC)
- Subnet privada 2
  - Nome: `sub-private02`
  - AZ: `us-east-1b`
  - IPv4 CIDR: `172.31.1.0/24`
- Subnet pública
  - Nome: `sub-public01`
  - AZ: `us-east-1a`
  - IPv4 CIDR: `172.31.2.0/24`
  - Após criar, selecione as configurações e cheque a caixa de `Enable auto-assign public IPv4 address`

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

 
