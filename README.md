# ANÁLISE TÉCNICA DO CÓDIGO

Este código Terraform é feito para provisionar e configurar diversos recursos AWS.
Ele cria recursos necessários para configurar uma VPC - Rede Privada Virtual - além de configurar uma instância EC2 com acesso SSH.

## Configuração do provedor AWS

```hcl
provider "aws" {
  region = "us-east-1"
}
```

- **Provedor AWS**: Configura o provedor para interagir com a API na região `us-east-1

## Variáveis

```hcl
variable "projeto" {
  description = "Rede Privada Virtual"
  type        = string
  default     = "VExpenses"
}

variable "candidato" {
  description = "Daniel Torres"
  type        = string
  default     = "Daniel"
}
```

- **`projeto`**: Define o nome do projeto, tendo como padrão: `VExpenses`
- Essas variáveis são usadas para nomear recursos AWS

## Criação de uma chave SSH

```hcl
resource "tls_private_key" "ec2_key" {
  algorithm = "RSA"
  rsa_bits  = 2048
}

resource "aws_key_pair" "ec2_key_pair" {
  key_name   = "${var.projeto}-${var.candidato}-key"
  public_key = tls_private_key.ec2_key.public_key_openssh
}
```

- **tls_private_key** : Cria uma chave privada que será utilizada para acessar a EC2 via SSH
- **aws_key_pair** : Gera um par de chaves na AWS com os nomes providenciados pelas variáveis. Este par de chaves são usadas para acessar as intâncias EC2

## Criação da VPC

```hcl
resource "aws_vpc" "main_vpc" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "${var.projeto}-${var.candidato}-vpc"
  }
}
```

- **AWS VPC** : Cria a VPC com um bloco CIDR de classe A e máscara /16, `10.0.0.0/16` e suporte DNS

## Criação de uma sub-rede

```hcl
resource "aws_subnet" "main_subnet" {
  vpc_id            = aws_vpc.main_vpc.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-east-1a"

  tags = {
    Name = "${var.projeto}-${var.candidato}-subnet"
  }
}
```

- Gera uma sub-rede dentro da VPC com o bloco CIDR `10.0.1.0/24, permitindo até 256 endereços de IP's válidos
- A sub-rede é criada na zona `us-east-1a`

## Internet Gateway

```hcl
resource "aws_internet_gateway" "main_igw" {
  vpc_id = aws_vpc.main_vpc.id

  tags = {
    Name = "${var.projeto}-${var.candidato}-igw"
  }
}
```

- Vinculando `Internet Gateway` à VPC, permite que as instâncias presentas na VPC tenham acesso a internet

## Tabela de rotas

```hcl
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

  tags = {
    Name = "${var.projeto}-${var.candidato}-route_table_association"
  }
}
```

- **Tabela de Rotas** : Cria uma tabela para a VPC, com uma rota que direciona todo tráfego(0.0.0.0/0) para internet gateway. Isso permite que as instânicas na sub-rede tenham acesso a internet

```hcl
resource "aws_route_table_association" "main_association" {
  subnet_id      = aws_subnet.main_subnet.id
  route_table_id = aws_route_table.main_route_table.id

  tags = {
    Name = "${var.projeto}-${var.candidato}-route_table_association"
  }
}
```

- Esta parte é responsável pela associação da tabela. Assim, é garantido que a sub-rede use as rotas definidas na tabela para encaminhar tráfego

### OBS.:

- O recurso `aws_route_table_association` não aceita o atributo tags. Para associar tags a uma tabela de rotas, pode-se utilizar o recurso `aws_route_table` para definir as tags e depois associá-las. Como já está sendo feito nesta parte:

```hcl
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
```

## Grupo de Segurança

```hcl
resource "aws_security_group" "main_sg" {
  name        = "${var.projeto}-${var.candidato}-sg"
  description = "Permitir SSH de qualquer lugar e todo o tráfego de saída"
  vpc_id      = aws_vpc.main_vpc.id

  ingress {
    description      = "Allow SSH from anywhere"
    from_port        = 22
    to_port          = 22
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }

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
```

- O **Grupo de Segurança** define regras para controlar o tráfego de entrada e saída das instâncias dentro da VPC:
    - Ingress (entrada): Permite conexões SSH (porta 22, protocolo TCP) de qualquer lugar (0.0.0.0/0 e ::/0 para IPv6).
    - Egress (saída): Permite todo o tráfego de saída, sem restrições.

## Amazon Machine Image (AMI)

```hcl
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
```

- Realiza uma busca pela AMI mais recente para Debian 12, utilizando filtros para garantir que a imagem seja HVM de 64 bits. Sendo a AMI providenciada por uma conta da AWS `679593333241`

## Criação da EC2

```hcl
resource "aws_instance" "debian_ec2" {
  ami             = data.aws_ami.debian12.id
  instance_type   = "t2.micro"
  subnet_id       = aws_subnet.main_subnet.id
  key_name        = aws_key_pair.ec2_key_pair.key_name
  security_groups = [aws_security_group.main_sg.name]

  associate_public_ip_address = true

  root_block_device {
    volume_size           = 20
    volume_type           = "gp2"
    delete_on_termination = true
  }

  user_data = <<-EOF
              #!/bin/bash
              apt-get update -y
              apt-get upgrade -y
              EOF

  tags = {
    Name = "${var.projeto}-${var.candidato}-ec2"
  }
}
```

- Cria uma instância EC2 usando a AMI mais recente do Debian 12
- **EC2** : Tem o tipo `t2.micro` e está associada à sub-rede e ao grupo de segurança
- Configura um armazenamento de 20gb do sistema operacional
- Atualizações do OS são feitas a partir de `user_data`

## Outputs

```hcl
output "private_key" {
  description = "Chave privada para acessar a instância EC2"
  value       = tls_private_key.ec2_key.private_key_pem
  sensitive   = true
}

output "ec2_public_ip" {
  description = "Endereço IP público da instância EC2"
  value       = aws_instance.debian_ec2.public_ip
}
```

- `private_key_pem`: Exibe a chave privada gerada para acesso à instância EC2. Este valor é marcado como sensível para evitar exposição acidental.
- `ec2_public_ip`: Exibe o endereço IP público da instância EC2, permitindo que o usuário se conecte à instância via SSH.

# Alterações

## Alteração do grupo de segurança 

```hcl
resource "aws_instance" "debian_ec2" {
  ami                    = data.aws_ami.debian12.id
  instance_type         = "t2.micro"
  subnet_id             = aws_subnet.main_subnet.id
  key_name              = aws_key_pair.ec2_key_pair.key_name
  vpc_security_group_ids = [aws_security_group.main_sg.id]  # Alterado para vpc_security_group_ids
  ...
```

- **De** : `security_group = [aws_security_group.main_sg.name]`
- **Para** : `vpc_security_group_ids = [aws_security_group.main_sg.id]`
- Isso permite que o código ligue diversos grupos de segurança à instância. Assim também pode ser mais apropriado para uma VPC

## Criptografia do volume de armazenamento

```hcl
root_block_device {
  volume_size           = 20
  volume_type           = "gp2"
  delete_on_termination = true
  encrypted             = true 
}
```

- Agora, o volume de armazenamento é criptografado, aumentando a segurança dos dados

## Nginx

```hcl
user_data = <<-EOF
            #!/bin/bash
            apt-get update -y
            apt-get upgrade -y
            apt-get install -y nginx
            systemctl start nginx
            systemctl enable nginx
            EOF
```

- Depois da instância `debian_ec2` ser criada, é executado um script de inicialização. Neste script o `Nginx` é instalado ( `apt-get install -y nginx` ), é inciado ( `systemctl start nginx` ) e é configurado para toda vez que o sistema seja reiniciado, ele seja iniciado automaticamente ( `systemctl enabled nginx` )

## Regras de segurança

```hcl
ingress {
  description      = "Allow HTTP traffic from anywhere"
  from_port        = 80
  to_port          = 80
  protocol         = "tcp"
  cidr_blocks      = ["0.0.0.0/0"]
  ipv6_cidr_blocks = ["::/0"]
}

ingress {
  description      = "Allow HTTPS traffic from anywhere"
  from_port        = 443
  to_port          = 443
  protocol         = "tcp"
  cidr_blocks      = ["0.0.0.0/0"]
  ipv6_cidr_blocks = ["::/0"]
}
```

- Para que seja permitido o tráfego HTTP e HTPPS, as regras de segurança foram alteradas
- As portas 80 e 443 permitem o tráfego HTTP e HTPPS, o que é necessário para servir páginas web a partir do Nginx

## Lifecycle

```hcl
lifecycle {
   prevent_destroy = true
   create_before_destroy = true
 }
```

- Lifecycle são argumentos de segurança que pode ser adicionado para ajudar a controlar o fluxo do código terraform a partir da criação de regras personalizadas na instância
- `prevent_destroy` : Esta opção impede que o Terraform de remover recursos cruciais acidentalmente
- `create_before_destroy` : Para alterações que possam complicar o comando `terraform apply`, esta opção faz com que um novo recurso seja criado antes da remoção ou destruição


# Passo a Passo

## Pré-requisitos

- **Terraform** : Terraform precisa estar instalado. Você pode baixar a versão mais recente no site original ou instalar o gerenciador de versões Tfenv: https://github.com/tfutils/tfenv
- **AWS CLI** : Instale a AWS CLI e configure suas credenciais AWS - `aws configure`- para que o terraform tenha acesso à sua conta AWS
- **Debian** : Tenha certeza de ter a máquina virtual referente ao Debian 12 ativada na sua conta AWS. Você pode conseguir na AWS Marketplace: https://aws.amazon.com/marketplace/pp/prodview-g5rooj5oqzrw4

## Diretório

- Clone o arquivo deste repositório na sua máquina

```git
git clone https://github.com/daniellucena1/VPC-VEx.git
```

- Ou baixe o .zip

## Terraform

- Dentro do diretório, execute os seguintes comandos:

```hcl
terraform init
```

- Para inicializar o ambiente terraform e baixar os pluggins necessários 

```hcl
terraform plan
```

- Para vizualizar tudo que será aplicado em sequência na infraestrutura

```hcl
terraform apply
```

- Para aplicar todas as alterações em seu infraestrutura
- Será solicitado que você digite `yes` para confirmas as alterações

## Vefirificar resultados

- Após a conclusão, a chave privada e o Ip serão mostrados no console.
- Para Verificar a criação dos recursos pode ser utilizado o AWS Console, ou a AWS CLI
