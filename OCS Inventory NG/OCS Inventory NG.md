# Tutorial Técnico: Instalação e Operacionalização do OCS Inventory NG em Rede Híbrida com Servidor em VirtualBox ou Docker

## Índice

1. [Introdução ao OCS Inventory NG](#1-introdução-ao-ocs-inventory-ng)  
2. [Arquitetura do Sistema](#2-arquitetura-do-sistema)  
3. [Requisitos de Hardware](#3-requisitos-de-hardware)  
4. [Instalação do Servidor](#4-instalação-do-servidor)  
   - [4.1 Preparação da Máquina Virtual no VirtualBox](#41-preparação-da-máquina-virtual-no-virtualbox)  
   - [4.2 Instalação no Ubuntu/Debian](#42-instalação-no-ubuntudebian)  
   - [4.3 Instalação no CentOS/RHEL](#43-instalação-no-centosrhel)  
   - [4.4 Instalação e Configuração do Servidor em Container Docker](#44-instalação-e-configuração-do-servidor-em-container-docker)  
5. [Instalação dos Agentes](#5-instalação-dos-agentes)  
   - [5.1 Agente Windows](#51-agente-windows)  
   - [5.2 Agente Linux](#52-agente-linux)  
   - [5.3 Agente macOS](#53-agente-macos)  
6. [Configuração de Rede](#6-configuração-de-rede)  
7. [Solução de Problemas](#7-solução-de-problemas)  

---

## 1. Introdução ao OCS Inventory NG

O **OCS Inventory NG** (Open Computers and Software Inventory Next Generation) é uma solução open source para gerenciamento de ativos de TI que permite:

- Inventário automático de hardware e software  
- Descoberta de rede via SNMP  
- Deploy de pacotes  
- Visualização via console web  
- Suporte multiplataforma (Windows, Linux, macOS)  

### Principais Componentes

- **Servidor de Banco de Dados**: MySQL/MariaDB para armazenamento  
- **Servidor de Comunicação**: Apache com mod_perl para HTTP/HTTPS  
- **Console de Administração**: Interface web em PHP  
- **Servidor de Deploy**: Para distribuição de pacotes  
- **Agentes**: Software instalado nas estações clientes  

---

## 2. Arquitetura do Sistema

A arquitetura do OCS Inventory é baseada no modelo cliente-servidor:

[Clientes com Agentes] ←→ [Servidor Apache] ←→ [Banco MySQL] ←→ [Console Web]
↓
[Servidor Deploy]

text

### Fluxo de Comunicação

1. Agentes coletam dados dos clientes  
2. Dados são enviados via XML comprimido (HTTP/HTTPS)  
3. Servidor processa e armazena no banco  
4. Console permite visualização e gerenciamento  
5. Deploy distribui pacotes para os clientes  

---

## 3. Requisitos de Hardware

### Hardware Mínimo

| Nº Máquinas       | CPU         | RAM     | Disco      | Rede      |
|-------------------|-------------|---------|------------|-----------|
| Até 100           | 2 vCPUs     | 2 GB    | 30 GB      | 100 Mbps  |
| Até 1.000         | 4 vCPUs     | 4 GB    | 100 GB     | 1 Gbps    |
| Até 10.000        | 8 vCPUs     | 8 GB    | 500 GB     | 1 Gbps    |

*Para grandes ambientes, recomenda-se separar banco de dados e servidor de aplicação.*

### Software Base Necessário

- Sistema Operacional: Linux (Ubuntu 18.04+, CentOS 7+, Debian 9+)  
- Apache Web Server 2.2.x ou superior  
- MySQL / MariaDB 8.0+ / 10.3+  
- PHP 7.0+ com extensões ZIP, GD, cURL, mbstring, XML, SOAP  
- Perl 5.6+ com módulos específicos (XML::Simple, Compress::Zlib, DBI, DBD::mysql, SOAP::Lite, etc.)

### Portas usadas na rede

- TCP 80 (HTTP) - Comunicação dos agentes  
- TCP 443 (HTTPS) - Deploy e comunicação segura  
- TCP 3306 (MySQL) - Acesso ao banco (local)

---

## 4. Instalação do Servidor

### 4.1 Preparação da Máquina Virtual no VirtualBox

1. **Criar VM Linux no VirtualBox**

- Sistema Operacional sugerido: Ubuntu 22.04 LTS (ou Debian/CentOS conforme preferência)  
- Recursos mínimos para até 100 hosts:  
  - CPU: 2 vCPUs  
  - RAM: 2 GB  
  - Disco: 30 GB (dinâmico ou fixo)  

2. **Configurar Placa de Rede**

- Configure a placa de rede da VM em **Modo Bridge** para permitir que a VM obtenha IP na mesma rede física, facilitando acesso dos agentes ao servidor.  

3. **Instalar o Sistema Operacional**

- Use a ISO do Ubuntu (ou SO escolhido) para instalação da VM seguindo o processo padrão.

4. **Dicas na VM**

- Após a instalação do SO, faça o update completo e instale os pacotes base necessários conforme os próximos passos do tutorial.
- Use `ip a` ou `ifconfig` para obter o IP da VM, que será utilizado para configurar os agentes.
- Recomenda-se configurar reserva de IP via DHCP no roteador para que a VM mantenha o mesmo IP.
- Crie snapshots da VM após configurações importantes para fácil rollback.

---

### 4.2 Instalação no Ubuntu/Debian

#### Passo 1: Atualizar sistema

sudo apt update && sudo apt upgrade -y

text

#### Passo 2: Instalar LAMP Stack (Apache, MariaDB, PHP)

sudo apt install apache2 mariadb-server mariadb-client php php-mysql php-gd php-curl php-xml php-mbstring php-zip php-soap libapache2-mod-php -y
sudo systemctl enable apache2 mariadb
sudo systemctl start apache2 mariadb
sudo mysql_secure_installation

text

#### Passo 3: Instalar módulos Perl

sudo apt install libperl-dev perl-modules-5.32 libxml-simple-perl libcompress-zlib-perl libdbi-perl libdbd-mysql-perl libapache-dbi-perl libnet-ip-perl libsoap-lite-perl make -y

text

#### Passo 4: Configurar banco de dados

sudo mysql -u root -p
CREATE DATABASE ocsweb;
CREATE USER 'ocs'@'localhost' IDENTIFIED BY 'SuaSenhaSegura123';
GRANT ALL PRIVILEGES ON ocsweb.* TO 'ocs'@'localhost' WITH GRANT OPTION;
FLUSH PRIVILEGES;
EXIT;

text

#### Passo 5: Baixar e instalar OCS Server

Baixe a última versão no GitHub:  
[https://github.com/OCSInventory-NG/OCSInventory-Server/releases/latest](https://github.com/OCSInventory-NG/OCSInventory-Server/releases/latest)

Exemplo:

cd /tmp
wget https://github.com/OCSInventory-NG/OCSInventory-Server/releases/download/2.12.3/OCSNG_UNIX_SERVER-2.12.3.tar.gz
tar -xzf OCSNG_UNIX_SERVER-2.12.3.tar.gz
cd OCSNG_UNIX_SERVER-2.12.3
sudo sh setup.sh

text

#### Passo 6: Configurar Apache

sudo a2enmod perl
sudo a2enconf ocsinventory-server
sudo a2enconf ocsinventory-reports
sudo systemctl restart apache2

text

---

### 4.3 Instalação no CentOS/RHEL

#### Passo 1: Atualizar sistema e instalar EPEL

sudo yum update -y
sudo yum install epel-release -y
sudo yum install wget vim -y

text

#### Passo 2: Instalar LAMP Stack

sudo yum install httpd mariadb mariadb-server php php-mysql php-gd php-xml php-mbstring php-zip php-curl php-soap -y
sudo systemctl enable httpd mariadb
sudo systemctl start httpd mariadb
sudo mysql_secure_installation

text

#### Passo 3: Instalar módulos Perl

sudo yum install perl-XML-Simple perl-Compress-Zlib perl-Net-IP perl-Digest-MD5 perl-Net-SSLeay perl-Data-UUID dmidecode perl-devel make gcc -y

text

#### Passo 4: Configurar firewall

sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload

text

#### Passo 5: Instalar OCS Server  
Siga o mesmo procedimento do Ubuntu/Debian na seção 4.2 Passo 5.

---

### 4.4 Instalação e Configuração do Servidor em Container Docker

Este método permite executar o servidor OCS Inventory NG isoladamente via container Docker, facilitando deploy rápido e portabilidade.

#### Pré-requisitos

- Sistema com Docker instalado (versão recomendada 20.10+)
- Conhecimento básico em Docker e linha de comando

#### Passo 1: Obter Imagem Docker

Use imagem confiável mantida na comunidade, por exemplo `jc21/ocsinventory`:

docker pull jc21/ocsinventory

text

#### Passo 2: Criar Rede Docker (opcional)

docker network create ocsnet

text

#### Passo 3: Rodar Container MariaDB

docker run -d
--network ocsnet
--name ocsdb
-e MYSQL_ROOT_PASSWORD=SUA_SENHA_ROOT
-e MYSQL_DATABASE=ocsweb
-e MYSQL_USER=ocs
-e MYSQL_PASSWORD=SUA_SENHA_OCS
mariadb:10.7

text

#### Passo 4: Rodar Container OCS Inventory Server

docker run -d
--network ocsnet
--name ocsserver
-p 80:80
-e DB_HOST=ocsdb
-e DB_NAME=ocsweb
-e DB_USER=ocs
-e DB_PASS=SUA_SENHA_OCS
jc21/ocsinventory

text

#### Passo 5: Acessar o Console Web

Abra: `http://localhost/` (se Docker local) ou `http://IP_do_servidor/`

#### Persistência de Dados

Monitore volumes Docker para MariaDB e OCS Inventory para manter dados estáveis entre reinícios.

---

## 5. Instalação dos Agentes

### 5.1 Agente Windows

- Download oficial:  
[OCS Inventory Windows Agent](https://lupahosting.com.br/publico/programas/OCS/Agents/OCS%20Inventory%20downloads.htm)

#### Instalação Manual
1. Baixe `OCS-NG-Windows-Agent-Setup.exe`  
2. Execute como administrador  
3. Use URL do servidor (ex: `http://<IP-do-servidor>/ocsinventory`)  
4. Configure opções conforme necessidade

#### Instalação Silenciosa

OCS-NG-Windows-Agent-Setup.exe /S /SERVER=http://<IP-do-servidor>/ocsinventory /NOW

text

#### Deploy via GPO

Configure política de instalação com pacote MSI.

---

### 5.2 Agente Linux

- Download oficial:  
[Unix Agent Releases](https://github.com/OCSInventory-NG/UnixAgent/releases/latest)

#### Pré-requisitos e instalação  
Segue conforme tutorial principal, apontando para URL do servidor.

---

### 5.3 Agente macOS

- Download oficial:  
[OCS Inventory macOS Agent](https://lupahosting.com.br/publico/programas/OCS/Agents/OCS%20Inventory%20downloads.htm)

#### Instalação e configuração conforme tutorial principal.

---

## 6. Configuração de Rede

- Assegure acesso nas portas TCP 80, 443 e 3306 conforme servidor escolhido (VM ou Docker).  
- Configure firewall e proxies conforme ambiente.

---

## 7. Solução de Problemas

Consulte logs, ajuste configurações e diagnósticos na documentação e seções anteriores para resolver falhas comuns.

---

Este tutorial completo permite a instalação flexível e gerenciamento do OCS Inventory NG em ambientes híbridos, com alternativas em VM VirtualBox ou via container Docker.

---