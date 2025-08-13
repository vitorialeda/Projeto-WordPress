# Projeto-WordPress

### Descrição do projeto:
Rodaremos uma aplicação web na nuvem AWS, utilizando serviços que garantirão desempenho, alta disponibilidade, escalabilidade e resiliência.

Os serviços utilizados são:
- Amazon Virtual Private Cloud (VPC);
- Amazon Elastic Compute Cloud (EC2);
- Amazon Relational Database Service (RDS);
- Amazon Elastic File System (EFS);
- Auto Scaling Group (ASG);
- Application Load Balancer (ALB).

![[diagrama-projeto.png]]
### Criando VPC:
- No painel de VPCs clique em **Criar VPC**
- Configure de forma que a VPC venha a ter:
	- 2 Subnets públicas (para o ALB)
	- 4 Subnets privadas (para EC2 e RDS)
	- Route Tables, IGW e NAT Gateway

![[criando-vpc-1-modelo-de-recursos.png]]

### Criando Security Group:
- No painel EC2, na aba Security Groups configure as seguintes **regras de entrada**:

|       Nome       |     Tipo     | Porta |         Origem          |
| :--------------: | :----------: | :---: | :---------------------: |
|      alb-sg      |     HTTP     |  80   |   Qualquer local-IPv4   |
|    bastion-sg    |     SSH      |  22   |        "Meu IP"         |
|    bastion-sg    |     HTTP     |  80   |   Qualquer local-IPv4   |
| wordpress-ec2-sg |     SSH      |  22   |       bastion-sg        |
| wordpress-ec2-sg |     HTTP     |  80   | bastion-sg (Provisório) |
|      rds-sg      | MYSQL/Aurora | 3306  |    wordpress-ec2-sg     |
|      efs-sg      |     NFS      | 2049  |    wordpress-ec2-sg     |
![[grupos-de-seguranca.png]]
### Criando RDS:

- No painel **Aurora and RDS**, clique em **Criar um banco de dados**
![[criando-rds-1-criar-banco-de-dados.png]]
##### Selecionando o Banco de Dados:
- Criação Padrão
- MySQL

![[criando-rds-2-metodo-e-mecanismo.png]]

##### Selecionando o modelo:
- Nível gratuito
![[criando-rds-3-selecionando-modelo.png]]

##### Definindo configurações básicas:
- Redefina os valores dessa aba conforme sua necessidade, utilizaremos os valores para nos conectarmos nossa aplicação ao banco de dados depois.
- Neste exemplo
	- Renomeei o **Identificador de instância de banco de dados**;
	- Selecionei a opção **Autogerenciada**
	- Marquei a opção **Gerar senha automaticamente**
![[criando-rds-4-config-basica.png]]

##### Configuração da instância:
- Mude para **db.t3.micro**
![[criando-rds-5-config-instancia.png]]

##### Definindo a conectividade:
- Marque "Não" em **Acesso público**;
- Marque "Selecionar existente" em **Grupo de segurança da VPC**;
- Selecione o grupo de segurança criado anteriormente referente ao RDS (rds-sg no caso desse exemplo);
![[criando-rds-6-conectividade.png]]

##### Configurações adicionais:
- Defina um nome para o seu banco em **Nome do banco de dados inicial**. (Também usaremos para nos conectar ao banco de dados posteriormente).
![[criando-rds-7-nome-bd.png]]

- Desça até o final da página e finalize a criação do Banco de dados.

##### Visualizar detalhes da conexão:
- Quando o banco de dados terminar de ser criado, clique em **Visualizar detalhes da conexão**
![[criando-rds-8-visualizar-detalhes.png]]

- Copie e **guarde** em um local seguro os valores de:
	- Nome do usuário principal;
	- Senha principal;
	- Endpoint;
	- Nome do banco de dados inicial.

### Criando EFS:
- No painel do Elastic File System, clique em **Criar sistema de arquivo**
![[criar-efs-1-criando-sistema-de-arquivo.png]]

##### Criando o sistemas de arquivo:
- Clique em **Personalizar**
![[criando-efs-2-personalizar.png]]

- No final da página, aperte em **Próximo**.

##### Acesso à rede:
- Em destino de montagem selecione os IDs de subnets *privadas* em **ID da subnet**. 
	- OBS: *No diagrama do projeto o Mount-Target está em subnets separadas das que estarão as EC2. Por isso, selecionei a subnet-private-3 e subnet-private-4.*
- Em **Grupos de segurança** selecione o grupo que criamos anteriormente referente à EFS.
![[criando-efs-3-acesso-a-rede.png]]

- Aperte em **Próximo** até concluir a criação da EFS

##### Guardando comando de montagem:
- No painel da EFS aperte no ID da EFS que criamos;
- Depois aperte em "Anexar";
![[criando-efs-4-anexar.png]]

- Copie a e **guarde** o código da segunda opção (*Usando o cliente do NFS*).
![[criando-efs-5-comando.png]]

### Criando EC2:
- No painel EC2, clique em **Executar instância**

##### Bastion:
- Selecione o par de chaves
- No bloco de **Configurações de rede**
	- Em **Sub-rede**, escolha uma sub-rede pública
	- Em **Atribuir IP público automaticamente** selecione a opção "Habilitar"
	- Marque a opção **Selecionar grupo de segurança existente** e selecione grupo que criamos referente ao bastion

![[criando-ec2-1-bastion.png]]
- Clique em **Executar instância**.

> O **Bastion** será a ferramenta que usaremos para fazer o **troubleshooting** (a solução de problemas) nas instâncias privadas onde o WordPress está em execução.
> - Você precisa acessar o **Bastion** a partir da sua máquina. Use o `SCP` para mover o arquivo de chaves de acesso da EC2 para lá e, em seguida, conecte-se à EC2 via `SSH`.

### Criando Launch Template:
- No painel EC2 clique em **Modelos de execução**, em seguida em **Criar modelo de execução**.

##### Nome e descrição do modelo de execução:
- Nomeie seu launch template;
- Marque a opção **Fornecer orientação para me ajudar a configurar um modelo que eu possa usar com o Auto Scaling do EC2**.
![[criando-launch-template-1.png]]

##### Imagens de aplicação e de sistema operacional:
- Selecione Início rápido e depois Amazon Linux.
![[criando-launch-template-2-imagem.png]]

##### Tipo de instância e Par de chaves:
- Selecione **t2.micro** como o **Tipo de instância**;
- Selecione o par de chaves que será utilizado.
![[criando-launch-template-3.png]]

##### Configurações de rede:
- Em sub-rede deixe selecionado **Não incluir no modelo de execução**;
- Selecione um grupo de segurança existente e marque apenas o grupo que criamos referente as instancias ec2 com WordPress.
![[criando-launch-template-4.png]]

##### Detalhes avançados:
- Em detalhes avançados cole o seguinte código em **Dados do usuário** (user data).

```bash
#!/bin/bash

## Monta o EFS
mkdir /home/ec2-user/efs

CODIGO_DE_MONTAGEM
# Ex: mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport DNS_EFS:/ efs

## Baixa as dependências
yum update -y
yum install -y httpd php php-mysqli

## Instala o docker e o docker-compose e altera permissões
yum install -y docker
service docker start
usermod -a -G docker ec2-user

curl -L https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m) -o /usr/local/bin/docker-compose  
chmod +x /usr/local/bin/docker-compose

## Cria arquivo do docker-compose e sobe o container
mkdir /home/ec2-user/docker-wordpress
cd /home/ec2-user/docker-wordpress

cat <<'EOF' > docker-compose.yml

services:
  wordpress:
    image: wordpress:latest
    ports:
      - 80:80
    environment:
      WORDPRESS_DB_HOST: ENDPOINT_RDS:3306
      WORDPRESS_DB_USER: USUARIO_PRINCIPAL
      WORDPRESS_DB_PASSWORD: SENHA_PRINCIPAL
      WORDPRESS_DB_NAME: NOME_BD_INICIAL
    volumes:
      - /efs:/var/www/html
EOF

docker-compose up -d
```

- **Mude as variáveis conforme seu ambiente**:
	- *CODIGO_DE_MONTAGEM*: Cole o comando copiado na seção: [[#Guardando comando de montagem]]
	- Utilize os dados copiados na seção [[#Visualizar detalhes da conexão]] para as seguintes variáveis:
		- *ENDPOINT_RDS*: Cole o endpoint do RDS (Lembre-se que após o endpoint vem a porta ":3306");
		- *USUARIO_PRINCIPAL*: Cole o nome de usuário principal do RDS;
		- *SENHA_PRINCIPAL*: Cole a senha principal do RDS;
		- *NOME_BD_INICIAL*: Cole o nome do banco de dados inicial.

Exemplo:
![[criando-launch-template-5.png]]


### Criando o Auto Scaling Group e Application Load Balancer:
- Aperte com botão direito no Launch Template que criamos e clique em **Criar Grupo de Auto Scaling**
![[criando-asg-1.png]]

##### Escolher o modelo de execução:
- Nomeie o grupo de Auto Scaling e garanta que selecionou a versão correta do template.
![[criando-asg-2.png]]

##### Rede:
- Em Rede selecione a VPC, as zonas de disponibilidades e sub-redes.
	- A sub-redes devem ser **privadas**
![[criando-asg-3.png]]

- Depois disso, no final da página clique em **Próximo** para avançar até a página de **Integrar com outros serviços**.

##### Criando o Application Load Balancer:
- Selecione a opção **Anexar a um novo balanceador de carga**
- Selecione a opção **Application Load Balancer**
- Nomeie o Load Balancer
- Selecione **Internet-facing** em **Esquema do balanceador de carga**
![[criando-lb-1.png]]

##### VPC:
- Em **Zonas de disponibilidade e sub-redes** selecione as sub-redes **públicas**
- Em **Listeners e roteamento** selecione a opção **Criar um grupo destino** nas opções de **Roteamento padrão (encaminhar para)** e dê um nome para esse grupo
![[criando-lb-2.png]]

##### Verificações de integridade:
- Em **Verificações de Integridade** marque a caixa: **Ative as verificações de integridade do Load Balancing**
- Clique em **Próximo** para avançar
![[criando-lb-3.png]]

#### Configurando o tamanho do Auto Scaling Group e o ajuste de escala:
- Em **Tamanho do grupo** digite a **Capacidade desejada**
![[configurando-asg-1.png]]

##### Escalabilidade:
- Em **Escalabilidade** digite os **Limites de ajuste de escala**
	- Para esse exemplo a quantidade mínima será de 1 e a máxima de 3.
- Selecione a opção de **Política de dimensionamento com monitoramento do objetivo** e ajuste conforme o necessário
	- Para esse exemplo a métrica foi: Uso médio da CPU em 50%
![[configurando-asg-2.png]]

##### Alterando verificação de integridade:
- Na aba **Grupos de destino**, clique com o botão direito no grupo de destino que criamos agora e aperte em **Editar configurações da verificação de integridade**.
![[alterando-integridade-1.png]]

- Em **Configurações avançadas de verificação de integridade**, altere **Códigos de sucesso** para: 200-399 e salve as alterações.
![[alterando-integridade-2.png]]

### Alternado o Security Group:
##### Load Balancer: 
- Na aba de **Load Balancers**, clique com o botão direito sobre o load balancer que criamos, em seguida clique em **Editar grupo de segurança**;
![[alterando-sg-1.png]]

- Selecione apenas o grupo que criamos previamente referente ao Application load balancer.
![[alterando-sg-2.png]]

##### EC2:
- Em Grupos de segurança, clique com o botão direito sobre o grupo de segurança da EC2, e depois em **Editar origem de entrada**;
![[alterandosg-3.png]]

- Altere a regra do tipo HTTP, mude a origem para o security group do Application Load Balancer
![[alterandosg-4.png]]

### Testando a aplicação:
- Na aba de Load Balancers copie o DNS do Load Balancer que criamos
![[testando-aplicacao-1.png]]

- Em seguida, cole o link no navegador.
![[testando-aplicacao-2.png]]