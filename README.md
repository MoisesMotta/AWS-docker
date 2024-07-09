# Configuração da Infraestrutura na AWS para Aplicação WordPress


## Passos de Configuração

### 1. Criando VPC

1. Acesse o console da AWS.
2. Vá para o serviço VPC.
3. Clique em "Create VPC".
   - **Nome**: my-vpc
   - **IPv4 CIDR block**: 10.0.0.0/16
   - **Tenancy**: Default
4. Clique em "Create VPC".

### 2. Criando Subnets

1. No console da VPC, clique em "Subnets".
2. Clique em "Create subnet".
   - **Nome**: my-public-subnet
   - **VPC**: my-vpc
   - **Availability Zone**: escolha uma AZ (por exemplo, us-east-1a)
   - **IPv4 CIDR block**: 10.0.1.0/24
3. Clique em "Create subnet".

4. Repita o processo para criar a subnet privada:
   - **Nome**: my-private-subnet
   - **VPC**: my-vpc
   - **Availability Zone**: escolha outra AZ (por exemplo, us-east-1b)
   - **IPv4 CIDR block**: 10.0.2.0/24
5. Anote os IDs das subnets criadas, pois serão usados em etapas futuras.

### 3. Criando NAT Gateway

1. No console da VPC, clique em "NAT Gateways".
2. Clique em "Create NAT Gateway".
   - **Subnet**: my-public-subnet
   - **Elastic IP Allocation**: clique em "Allocate Elastic IP" e selecione o IP alocado.
3. Clique em "Create NAT Gateway".

### 4. Configurando Route Tables

1. No console da VPC, clique em "Route Tables".
2. Selecione a Route Table associada à VPC.
3. Adicione uma rota para a subnet pública:
   - **Destino**: 0.0.0.0/0
   - **Target**: Internet Gateway (selecione o Internet Gateway associado à VPC)
4. Adicione uma rota para a subnet privada:
   - **Destino**: 0.0.0.0/0
   - **Target**: NAT Gateway (selecione o NAT Gateway criado na etapa anterior)

### 5. Criando Security Groups

1. No console da VPC, clique em "Security Groups".
2. Crie os seguintes Security Groups:
   - **SG-EC2**:
     - **Nome**: ec2-sg
     - **Descrição**: SG para EC2
     - **VPC**: my-vpc
     - **Regras de entrada**: HTTP (porta 80), SSH (porta 22)
   - **SG-EFS**:
     - **Nome**: efs-sg
     - **Descrição**: SG para EFS
     - **VPC**: my-vpc
     - **Regras de entrada**: NFS (porta 2049)
   - **SG-RDS**:
     - **Nome**: rds-sg
     - **Descrição**: SG para RDS
     - **VPC**: my-vpc
     - **Regras de entrada**: MySQL/Aurora (porta 3306)
   - **SG-LB**:
     - **Nome**: lb-sg
     - **Descrição**: SG para Load Balancer
     - **VPC**: my-vpc
     - **Regras de entrada**: HTTP (porta 80)

### 6. Criando EC2 Connect Endpoint

1. No console da VPC, clique em "Endpoints".
2. Clique em "Create endpoint".
   - **Service name**: com.amazonaws.vpce.us-east-1.ec2
   - **VPC**: my-vpc
   - **Subnets**: my-private-subnet
   - **Security Group**: ec2-sg

### 7. Criando Elastic File System (EFS)

1. No console da AWS, acesse o serviço EFS.
2. Clique em "Create file system".
   - **Tipo**: Regional
   - **VPC**: my-vpc
   - **Security Group**: efs-sg
   - **Subnets**: my-private-subnet
3. Clique em "Create".

4. Anote o DNS Name do EFS criado, que será usado na configuração do Docker Compose.

### 8. Criando Relational Database Service (RDS)

1. No console da AWS, acesse o serviço RDS.
2. Clique em "Create database".
   - **Engine type**: MySQL
   - **Version**: 8.0.35
   - **Template**: Free Tier
   - **Instance type**: db.t3.micro
   - **Storage**: gp3, 20GB
   - **VPC**: my-vpc
   - **Public access**: No
   - **VPC Security Group**: rds-sg
   - **Initial database name**: escolha um nome
3. Clique em "Create database".

4. Anote o endpoint do RDS criado, que será usado na configuração do Docker Compose.

### 9. Criando Launch Template

1. No console da AWS, acesse o serviço EC2.
2. Clique em "Launch Templates".
3. Clique em "Create launch template".
   - **Name**: my-launch-template
   - **AMI**: Amazon Linux 2023
   - **Instance type**: t3.small
   - **Key pair**: selecione ou crie um par de chaves
   - **Network settings**:
     - **VPC**: my-vpc
     - **Security Group**: ec2-sg
   - **Storage**: 20GB gp3
   - **Advanced details**: adicione o seguinte script em User data:
     ```bash
     #!/bin/bash
     # Atualizar pacotes e instalar Docker
     sudo yum update -y
     sudo amazon-linux-extras install docker -y
     sudo service docker start
     sudo usermod -a -G docker ec2-user

     # Instalar Docker Compose
     sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
     sudo chmod +x /usr/local/bin/docker-compose

     # Instalar NFS client
     sudo yum install -y nfs-utils

     # Criar diretório para EFS
     sudo mkdir -p /mnt/efs

     # Montar EFS
     sudo mount -t nfs4 -o nfsvers=4.1 <EFS_DNS_NAME>:/ /mnt/efs

     # Adicionar ao /etc/fstab para montagem persistente
     echo "<EFS_DNS_NAME>:/ /mnt/efs nfs4 defaults,_netdev 0 0" | sudo tee -a /etc/fstab

     # Criar Docker Compose file
     cat <<EOF > /home/ec2-user/docker-compose.yml
     version: '3'
     services:
       wordpress:
         image: wordpress:latest
         ports:
           - "80:80"
         environment:
           WORDPRESS_DB_HOST: <RDS_ENDPOINT>:3306
           WORDPRESS_DB_USER: <DB_USER>
           WORDPRESS_DB_PASSWORD: <DB_PASSWORD>
           WORDPRESS_DB_NAME: <DB_NAME>
         volumes:
           - /mnt/efs:/var/www/html
     EOF

     # Iniciar Docker Compose
     cd /home/ec2-user
     sudo docker-compose up -d
     ```
4. Clique em "Create launch template".

5. Anote o ID do launch template criado para usar em etapas posteriores.

### 10. Criando Auto Scaling Group

1. No console da AWS, acesse o serviço EC2.
2. Clique em "Auto Scaling Groups".
3. Clique em "Create Auto Scaling Group".
   - **Group name**: my-auto-scaling-group
   - **Launch template**: selecione o template criado anteriormente (my-launch-template)
   - **Network**: selecione a VPC (my-vpc) e as subnets desejadas (my-private-subnet)
   - **Load balancing**: configure conforme abaixo
     - **Load Balancer**: selecione o load balancer criado anteriormente (my-load-balancer)
     - **Health check type**: ELB
   - **Group size**: configure conforme necessário
   - **Scaling policies**: defina políticas de escalabilidade conforme necessário
   - **Notifications**: configure notificações conforme necessário
4. Clique em "Create Auto Scaling Group".

### 11. Configurando Load Balancer

1. No console da AWS, acesse o serviço EC2.
2. Clique em "Load Balancers".
3. Clique em "Create Load Balancer".
   - **Tipo**: Classic Load Balancer
   - **Nome**: my-load-balancer
   - **VPC**: my-vpc
   - **Listeners**: configure conforme necessário (por exemplo, HTTP na porta 80)
   - **Availability Zones**: selecione as subnets públicas criadas anteriormente (my-public-subnet)
   - **Security Settings**: selecione o security group lb-sg
   - **Health Check**: configure conforme necessário
4. Clique em "Create".

5. Anote o DNS Name do Load Balancer criado para acessar a aplicação WordPress.

---
