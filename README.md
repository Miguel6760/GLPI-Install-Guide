# GLPI-Install-Guide
Guía para la instalacion de un servidor GLPI y un agente simple.

*** Instalación y configuracion docker ***

Primero instalaremos docker-engine y docker-compose. Para ello, primero añadimos el repositorio oficial de docker mediante este script:

```nano
#!bin/bash

# Add Docker's official GPG key:
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
```

Después, instalamos docker con el siguiente comando:

```bash
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Ahora añadimos el usuario principal al grupo de docker para poder ejecutar docker sin sudo:

```bash
sudo usermod -aG docker TUUSUARIO
```
Hay que reiniciar la maquina para guardar los cambios.

*** Descarga de imagenes ***

Ahora procedemos a la descarga de las imagenes de GLPI y MariaDB:

```bash
docker pull glpi/glpi
```
