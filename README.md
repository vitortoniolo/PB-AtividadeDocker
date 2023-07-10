# Servidor Wordpress via Docker em AWS
![Screenshot_5](https://github.com/vitortoniolo/PB-AtividadeDocker/assets/133904035/cf3155bd-4aed-4331-83d2-bee70cf990ef)

## Requisitos
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

## Integrantes
- [Pedro Liu](https://github.com/PedroTxfl/Projeto_docker_wordpress) 
- [Bruno Marques](https://github.com/BrunoMarques1/Atividade_DOCKER/tree/main)
- [José Toniolo](https://github.com/vitortoniolo/PB-AtividadeDocker)
  
## Configurações 
### VPC e Subnets
Crie uma nova VPC, nomeando-a (ex: Wordpress) e decidindo o bloco IPv4 CIDR (ex: 172.28.0.0/16)

Agora navegue para subnets e crie conforme o seguinte (o que não for específicado deve ser mantido padrão):
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
- Script user data (mude o ip):
```bash
#!/bin/bash
sudo yum update -y
sudo yum install nfs-utils -y
sudo mkdir -p /mnt/nfs
sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport <ip do efs>:/ /mnt/nfs
sudo echo "<ip do efs>:/    /mnt/nfs         nfs    defaults          0   0 " >> /etc/fstab
sudo yum install docker -y
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -a -G docker ec2-user
sudo curl -L https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m) -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

  

### Configurando a instância para acesso:
Na sua máquina, abra o terminal e execute: `ssh-add <suachave.pem>`.

Agora você deve acessar o seu bastion host com: `ssh -A -i <suachave.pem> ec2-user@<IP público da instância>`

E a instância wordpress com: `ssh ec2-user@<IP privado da instância>`

Alternativamente, você pode copiar a chave para dentro de seu bastion host com o comando: `scp -i <suachave.pem> ec2-user@<ip público do bastion>:/home/ec2-user`

### Subindo o container Wordpress:
Acessando a instância via o bastion host, crie um arquivo `docker-compose.yaml` com o seguinte código:
```yaml
version: '3'
services:
  wordpress:
    image: wordpress:latest
    ports:
      - "80:80"
    restart: always
    environment:
      WORDPRESS_DB_HOST: <Endpoint do banco de dados criado>
      WORDPRESS_DB_USER: <Usuário criado>
      WORDPRESS_DB_PASSWORD: <Senha criada>
      WORDPRESS_DB_NAME: <Banco de dados inicial criado>
    volumes:
      - /mnt/nfs/wordpress:/var/www/html
```
Execute `docker-compose up -d` para subir o container.

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
 
### Auto Scaling
No console EC2, selecione a nossa instância Wordpress e vá em `Images and templates > Create Image`. Escolha um nome e mantenha as configurações padrões.

Agora abra `Launch Templates` e crie um modelo novo com as seguintes configurações:
- Imagem: `Selecione a imagem criada anteriormente`
- Tipo de instância: `t2.micro`
- Key pair: `Selecione uma existente ou crie uma nova`
- Security group: `Selecione o mesmo da instância Wordpress`

Podemos agora criar o Auto Scaling Group, seguindo estes parâmetros:
- Nome: `ASG-WP`
- Launch template: `Selecione a template criada aneriormente`
- VPC: `wordpress`
- AZs: `Selecione as duas subnets privadas`
- Load balancing: `Selecione o nosso grupo de LB`
- Health checks: `Ative o Elastic Load Balancing health checks`
- Group size:
  - Desired capacity: `2`
  - Minimum capacity: `2`
  - Maximum capacity: `4` 
- Scaling policies: `Target tracking scaling policy`
  - Target value: 70
 
  

