# azure-nodejs-api-deploy

Este repositório contém um script automatizado para realizar o deploy de uma API Node.js na infraestrutura do Azure, configurando uma máquina virtual (VM) Ubuntu, instalando o Nginx como proxy reverso e executando a API na porta 3000.

## Estrutura do Projeto

api-deploy-azure/ ├── app.js ├── package.json ├── README.md ├── nginx.conf ├── deploy.sh ├── .gitignore

php
Copiar

## Arquivos

### 1. `deploy.sh`

Este script automatiza o processo de criação da VM no Azure, configuração do Nginx e deploy de uma API Node.js. Ele cria a infraestrutura necessária no Azure, configura o Nginx e executa a API.

```bash
#!/bin/bash

# Definindo variáveis
RESOURCE_GROUP="rg-nodejs-api"
LOCATION="East US"
VM_NAME="nodejs-api-vm"
VM_USER="azureuser"
VM_PASSWORD="YourPassword123!"  # Altere para uma senha mais forte
PUBLIC_IP_NAME="nodejs-api-ip"
VNET_NAME="nodejs-vnet"
SUBNET_NAME="nodejs-subnet"
NIC_NAME="nodejs-nic"
VM_SIZE="Standard_B1ms"
IMAGE="UbuntuLTS"

# Criação do grupo de recursos
echo "Criando grupo de recursos..."
az group create --name $RESOURCE_GROUP --location $LOCATION

# Criando rede virtual e sub-rede
echo "Criando rede virtual e sub-rede..."
az network vnet create --resource-group $RESOURCE_GROUP --location $LOCATION --name $VNET_NAME --address-prefix 10.0.0.0/16 --subnet-name $SUBNET_NAME --subnet-prefix 10.0.1.0/24

# Criando IP público
echo "Criando IP público..."
az network public-ip create --resource-group $RESOURCE_GROUP --name $PUBLIC_IP_NAME --allocation-method Dynamic

# Criando interface de rede
echo "Criando interface de rede..."
az network nic create --resource-group $RESOURCE_GROUP --name $NIC_NAME --vnet-name $VNET_NAME --subnet $SUBNET_NAME --public-ip-address $PUBLIC_IP_NAME

# Criando a máquina virtual
echo "Criando máquina virtual..."
az vm create --resource-group $RESOURCE_GROUP --name $VM_NAME --size $VM_SIZE --image $IMAGE --admin-username $VM_USER --admin-password $VM_PASSWORD --nics $NIC_NAME --public-ip-address $PUBLIC_IP_NAME --output none

# Abrindo a porta 80 para HTTP
echo "Abrindo a porta 80 no firewall..."
az vm open-port --resource-group $RESOURCE_GROUP --name $VM_NAME --port 80

# Exibindo o IP público da VM
PUBLIC_IP=$(az vm show --resource-group $RESOURCE_GROUP --name $VM_NAME --show-details --query publicIps -o tsv)
echo "Acesse a API usando o IP público: $PUBLIC_IP"

# Conectando na VM para instalar dependências e configurar a aplicação
echo "Conectando à VM para configurar a aplicação..."
sshpass -p $VM_PASSWORD ssh -o StrictHostKeyChecking=no $VM_USER@$PUBLIC_IP << EOF
  # Atualizando pacotes
  sudo apt update
  sudo apt upgrade -y

  # Instalando Node.js e Nginx
  sudo apt install -y nodejs npm nginx

  # Criando diretório para a API
  mkdir ~/api-deploy
  cd ~/api-deploy

  # Criando arquivo package.json
  echo '{
    "name": "api-deploy-azure",
    "version": "1.0.0",
    "main": "app.js",
    "scripts": {
      "start": "node app.js"
    },
    "dependencies": {
      "express": "^4.18.2"
    }
  }' > package.json

  # Instalando dependências
  npm install express

  # Criando o código da API (app.js)
  echo 'const express = require("express");
  const app = express();
  const port = 3000;

  app.get("/", (req, res) => {
    res.send("API funcionando no servidor Linux com Nginx!");
  });

  app.listen(port, () => {
    console.log(\`API rodando em http://localhost:\${port}\`);
  });' > app.js

  # Configurando o Nginx como proxy reverso
  echo 'server {
    listen 80;

    server_name _;

    location / {
      proxy_pass http://localhost:3000;
      proxy_http_version 1.1;
      proxy_set_header Upgrade \$http_upgrade;
      proxy_set_header Connection "upgrade";
      proxy_set_header Host \$host;
      proxy_cache_bypass \$http_upgrade;
    }
  }' | sudo tee /etc/nginx/sites-available/default > /dev/null

  # Reiniciando o Nginx
  sudo systemctl restart nginx

  # Rodando a API
  nohup node app.js &

  # Finalizando
  echo "API instalada e em execução!"


echo "Deploy concluído!"

2. nginx.conf
Este arquivo contém a configuração do Nginx para redirecionar o tráfego da porta 80 para a aplicação Node.js, que roda na porta 3000.

nginx

server {
    listen 80;

    server_name _;

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
3. package.json
Este arquivo define as dependências do projeto e o script de inicialização da API.

json

{
  "name": "api-deploy-azure",
  "version": "1.0.0",
  "main": "app.js",
  "scripts": {
    "start": "node app.js"
  },
  "dependencies": {
    "express": "^4.18.2"
  }
}

4. .gitignore
Este arquivo é utilizado para evitar o envio de arquivos desnecessários ao GitHub, como dependências do Node.js.

gitignore

node_modules/
Passo a Passo para Executar o Script

##Clone o repositório:

bash

git clone https://github.com/seu-usuario/api-deploy-azure.git
cd api-deploy-azure
Instalar o sshpass (caso necessário):

bash

sudo apt-get install sshpass
Torne o script executável:

bash

chmod +x deploy.sh
Execute o script:

bash

./deploy.sh
Acesse a API: Após a execução do script, o IP público da máquina será exibido no terminal. Acesse a API via navegador ou usando o curl:

bash

http://<PUBLIC_IP>
A resposta será:

bash

API funcionando no servidor Linux com Nginx!
Licença
Este projeto está licenciado sob a licença MIT.

## Conclusão
Com este processo automatizado via Azure CLI, você consegue criar uma máquina virtual Linux, instalar as dependências necessárias, configurar o Nginx como proxy reverso e fazer o deploy de uma API Node.js de forma totalmente automatizada. Esse procedimento pode ser útil tanto para ambientes de desenvolvimento quanto de produção, proporcionando uma forma rápida e consistente de realizar deploys na Azure.
