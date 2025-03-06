# Descrição Técnica do Código Terraform

O código fornecido é responsável por provisionar uma infraestrutura na AWS usando Terraform. Ele cria uma série de recursos, como uma VPC, sub-rede, gateway de internet, tabela de rotas, grupo de segurança, instância EC2 com Debian 12, e uma chave SSH. O código também gera uma chave privada para acesso seguro à instância EC2 e configura um script de inicialização para a instalação do Debian.

1. Provider AWS
provider "aws": Define a configuração do provedor AWS. A região us-east-1 (Norte da Virgínia) é escolhida para provisionar os recursos.
2. Variáveis
variable "projeto" e variable "candidato": Definem duas variáveis de entrada para o código: projeto e candidato. Esses valores são usados ao longo de toda a configuração para personalizar os nomes dos recursos criados, como VPC, instâncias e grupos de segurança.
3. Chave Privada (TLS)
resource "tls_private_key" "ec2_key": Cria uma chave privada usando o algoritmo RSA de 2048 bits. Essa chave será usada para criar um par de chaves SSH, permitindo acesso seguro à instância EC2.
4. Key Pair EC2
resource "aws_key_pair" "ec2_key_pair": Cria um par de chaves SSH na AWS usando a chave pública gerada no recurso anterior (tls_private_key). O nome da chave SSH é configurado dinamicamente com base nas variáveis projeto e candidato.
5. VPC (Virtual Private Cloud)
resource "aws_vpc" "main_vpc": Cria uma VPC com o bloco CIDR 10.0.0.0/16, permitindo um grande intervalo de endereços IP privados. A VPC tem suporte a DNS e nomes de host habilitados, o que facilita a comunicação entre instâncias e serviços na rede.
6. Sub-rede
resource "aws_subnet" "main_subnet": Cria uma sub-rede dentro da VPC com o bloco CIDR 10.0.1.0/24 na zona de disponibilidade us-east-1a. Isso limita o alcance da sub-rede, permitindo até 256 endereços IP.
7. Gateway de Internet
resource "aws_internet_gateway" "main_igw": Cria um gateway de internet, permitindo que a VPC se conecte à internet. Este gateway é associado à VPC criada anteriormente.
8. Tabela de Roteamento
resource "aws_route_table" "main_route_table": Cria uma tabela de rotas para a VPC. A tabela define uma rota para o bloco CIDR 0.0.0.0/0 (toda a internet) através do gateway de internet aws_internet_gateway.main_igw. Isso permite que as instâncias na VPC acessem a internet.
9. Associação da Tabela de Roteamento
resource "aws_route_table_association" "main_association": Associa a tabela de rotas à sub-rede criada anteriormente (aws_subnet.main_subnet). Isso garante que o tráfego da sub-rede será roteado conforme definido na tabela de rotas.
10. Grupo de Segurança
resource "aws_security_group" "main_sg": Cria um grupo de segurança para a VPC. Este grupo de segurança permite tráfego de entrada na porta 22 (SSH) de qualquer IP (0.0.0.0/0), permitindo acesso remoto à instância EC2. Também permite tráfego de saída ilimitado, sem restrições.
11. AMI (Amazon Machine Image) - Debian 12
data "aws_ami" "debian12": Busca a imagem mais recente do Debian 12 na AWS usando filtros baseados no nome e tipo de virtualização. A imagem é usada para criar a instância EC2.
12. Instância EC2
resource "aws_instance" "debian_ec2": Provisiona uma instância EC2 usando a AMI Debian 12 mais recente. A instância será criada com:
Tipo t2.micro (adequado para uso de baixo custo e baixo tráfego).
Sub-rede aws_subnet.main_subnet.id e um par de chaves aws_key_pair.ec2_key_pair.key_name para permitir acesso SSH.
Um endereço IP público será associado à instância para acesso externo.
O dispositivo de bloco raíz terá um tamanho de 20 GB e será configurado para ser excluído quando a instância for terminada.
user_data é usado para automatizar a atualização e instalação do sistema operacional, com o comando apt-get update e apt-get upgrade executados ao iniciar a instância.
13. Outputs
output "private_key": Exibe a chave privada gerada no recurso tls_private_key. Essa chave privada será usada para acessar a instância EC2 via SSH.
output "ec2_public_ip": Exibe o endereço IP público da instância EC2, permitindo que o usuário acesse a instância de fora da AWS.

Como os recursos interagem:
A VPC é criada para fornecer a rede isolada dentro da AWS.
A sub-rede é criada dentro dessa VPC, e a tabela de rotas garante que a sub-rede tenha acesso à internet por meio do gateway de internet.
Um grupo de segurança permite acesso remoto via SSH à instância EC2, configurando as permissões de tráfego.
A instância EC2 é provisionada com a AMI Debian 12, usando a chave SSH para acesso seguro.
O script user_data garante que o sistema operacional da instância EC2 seja atualizado assim que ela for inicializada.
As chaves privadas e o IP público da instância são retornados como outputs, fornecendo os dados necessários para acessar a instância remotamente.

Arquivo main.tf Modificado: # Código Terraform modificado

provider "aws" {
  region = "us-east-1"
}

variable "projeto" {
  description = "Nome do projeto"
  type        = string
  default     = "VExpenses"
}

variable "candidato" {
  description = "Nome do candidato"
  type        = string
  default     = "SeuNome"
}

resource "tls_private_key" "ec2_key" {
  algorithm = "RSA"
  rsa_bits  = 2048
}

resource "aws_key_pair" "ec2_key_pair" {
  key_name   = "${var.projeto}-${var.candidato}-key"
  public_key = tls_private_key.ec2_key.public_key_openssh
}

resource "aws_vpc" "main_vpc" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "${var.projeto}-${var.candidato}-vpc"
  }
}

resource "aws_subnet" "main_subnet" {
  vpc_id            = aws_vpc.main_vpc.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-east-1a"

  tags = {
    Name = "${var.projeto}-${var.candidato}-subnet"
  }
}

resource "aws_internet_gateway" "main_igw" {
  vpc_id = aws_vpc.main_vpc.id

  tags = {
    Name = "${var.projeto}-${var.candidato}-igw"
  }
}

resource "aws_route_table" "main_route_table" {
  vpc_id = aws_vpc.main_vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main_igw.id
  }

  tags = {
    Name = "${var.projeto}-${var.candidato}-route_table"
  }
}

resource "aws_route_table_association" "main_association" {
  subnet_id      = aws_subnet.main_subnet.id
  route_table_id = aws_route_table.main_route_table.id
}


resource "aws_security_group" "main_sg" {
  name        = "${var.projeto}-${var.candidato}-sg"
  description = "Permitir SSH de IPs confiaveis e todo o trafego de saida"
  vpc_id      = aws_vpc.main_vpc.id

 Regras de entrada (ajuste o CIDR para permitir SSH de um IP confiável)
  ingress {
    description      = "Allow SSH from trusted IP only"
    from_port        = 22
    to_port          = 22
    protocol         = "tcp"
    cidr_blocks      = ["177.37.170.177/32"]  
    ipv6_cidr_blocks = ["::/0"]
  }

  Regras de saída
  egress {
    description      = "Allow all outbound traffic"
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }

  tags = {
    Name = "${var.projeto}-${var.candidato}-sg"
  }
}

data "aws_ami" "debian12" {
  most_recent = true

  filter {
    name   = "name"
    values = ["debian-12-amd64-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }

  owners = ["679593333241"]
}

resource "aws_instance" "debian_ec2" {
  ami             = data.aws_ami.debian12.id
  instance_type   = "t2.micro"
  subnet_id       = aws_subnet.main_subnet.id
  key_name        = aws_key_pair.ec2_key_pair.key_name
  security_groups = [aws_security_group.main_sg.id]

  associate_public_ip_address = true

  root_block_device {
    volume_size           = 20
    volume_type           = "gp2"
    delete_on_termination = true
  }

  Automação para instalar o Nginx
  user_data = <<-EOF
              #!/bin/bash
              apt-get update -y
              apt-get upgrade -y
              apt-get install -y nginx
              systemctl start nginx
              systemctl enable nginx
              EOF

  tags = {
    Name        = "${var.projeto}-${var.candidato}-ec2"
    Environment = "Production"
    Owner       = var.candidato
  }
  depends_on = [aws_security_group.main_sg]  # Garante que o Security Group será criado primeiro
}

output "private_key" {
  description = "Chave privada para acessar a instancia EC2"
  value       = tls_private_key.ec2_key.private_key_pem
  sensitive   = true
}

output "ec2_public_ip" {
  description = "Endereço IP público da instancia EC2"
  value       = aws_instance.debian_ec2.public_ip
}


Descrição Técnica das Alterações
1. Melhorias de Segurança
Reforço na segurança do grupo de segurança (Security Group):
Atualmente, o código permite SSH de qualquer lugar (0.0.0.0/0), o que pode ser um risco. Podemos melhorar isso ao restringir o acesso SSH para um IP específico (ou um intervalo de IPs confiáveis).
Outra melhoria seria adicionar regras para limitar o tráfego de entrada e saída, caso necessário.
Alteração proposta:
ingress {
  description      = "Allow SSH from trusted IP only"
  from_port        = 22
  to_port          = 22
  protocol         = "tcp"
  cidr_blocks      = ["198.51.100.0/24"] 
  ipv6_cidr_blocks = ["::/0"]
}

Isso permite que apenas um IP ou rede específica acesse a instância via SSH, garantindo um nível adicional de segurança.

2. Automação da Instalação do Nginx:
User Data: A principal melhoria foi a automação da instalação do Nginx no script user_data. Assim que a instância EC2 for criada, o Nginx será automaticamente instalado, iniciado e configurado para iniciar na inicialização.
O comando apt-get update e apt-get upgrade garante que o sistema esteja atualizado antes de instalar o Nginx.
O comando apt-get install -y nginx instala o Nginx.
systemctl start nginx garante que o Nginx seja iniciado imediatamente após a instalação.
systemctl enable nginx configura o Nginx para iniciar automaticamente a cada inicialização da instância.

3. Outras Melhorias:
Chave Privada Sensível: O campo private_key foi marcado como sensitive = true. Isso evita que a chave privada seja exibida no output de execução do Terraform, aumentando a segurança.
Recurso root_block_device com Volume de 20 GB: O volume de armazenamento foi mantido com 20 GB, que é uma quantidade mínima e suficiente para uma instalação do Nginx. Para produção, recomenda-se ajustar o tamanho do volume conforme a necessidade da aplicação.
Melhoria sugerida:
Backup de Dados: Para uma infraestrutura de produção, pode ser necessário configurar backups automáticos ou snapshots para o volume root_block_device, garantindo a recuperação de dados em caso de falhas.

Instruções de Uso
Pré-requisitos:
Terraform instalado em sua máquina local.
Conta AWS configurada com as credenciais necessárias.
Chave de Acesso (AWS Access Key) configurada no seu ambiente.
Passos para Inicializar e Aplicar a Configuração:
Baixar o código Terraform:

Faça o download do arquivo main.tf ou clone o repositório onde o código está armazenado.
Configuração de Variáveis:
Altere as variáveis no arquivo main.tf (como projeto e candidato) para personalizar o nome da infraestrutura.
Inicializar o Terraform:

No diretório onde o arquivo main.tf está localizado, execute o seguinte comando para inicializar o Terraform e baixar os provedores necessários:
terraform init

Executar o Plano do Terraform:
Execute o comando para visualizar o que será criado:
terraform plan

Aplicar a Configuração:
Após verificar o plano, execute o comando para aplicar a configuração e criar a infraestrutura na AWS:
terraform apply

Acessar a Instância EC2:
Após a criação, o Terraform exibirá o IP público da instância EC2. Use a chave privada gerada para acessar a instância via SSH:
ssh -i /caminho/para/sua/chave.pem ubuntu@<ec2_public_ip>
