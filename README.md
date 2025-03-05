Descrição Técnica do Código Terraform

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

Descrição Técnica das Alterações
1. Melhorias de Segurança:
Restrição de Acesso SSH: Embora o código original permita o acesso SSH de qualquer lugar (0.0.0.0/0), para aumentar a segurança, seria interessante restringir o acesso SSH para IPs específicos ou um intervalo de IPs conhecidos, em vez de permitir acesso irrestrito. Isso reduz as chances de ataques de força bruta.
Alteração proposta:
hcl
CopiarEditar
ingress {
  description      = "Allow SSH from a specific IP"
  from_port        = 22
  to_port          = 22
  protocol         = "tcp"
  cidr_blocks      = ["YOUR_IP/32"]
  ipv6_cidr_blocks = ["YOUR_IPV6/128"]
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

