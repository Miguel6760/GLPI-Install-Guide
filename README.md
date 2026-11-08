# GLPI-Install-Guide
Guía para la instalacion de un servidor GLPI y un agente simple en Ubuntu Server.

## ÍNDICE
- [Instalación y configuración docker](#instalación-y-configuracion-docker)
- [Descarga de imagenes](#descarga-de-imagenes)
- [Configuración de los contenedores](#configuración-de-los-contenedores)
- [Inicio de los contenedores](#inicio-de-los-contenedores)

## Instalación y configuracion docker

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

## Descarga de imagenes

Ahora procedemos a la descarga de las imágenes de GLPI y MariaDB:

```bash
docker pull glpi/glpi
docker pull mariadb:lts-ubi10
```

Para comprobar que se han descargado, ejecutamos el siguiente comando:

```bash
asrc@ServerGLPI:~$ docker images
                                                                                                                           
IMAGE               ID             DISK USAGE   CONTENT SIZE   EXTRA
glpi/glpi:latest    f1a08a72afa5       1.69GB          376MB    U
mariadb:lts-ubi10   3b6b6ef7565b        723MB          171MB    U
```

## Configuración de los contenedores

Ahora que tenemos las imágenes, vamos a configurarlas para que se comuniquen entre si.

Creamos un directorio de trabajo para el contenedor ( en mi caso GLPI ):

```bash
mkdir GLPI
```
Dentro del directorio creamos 2 archivos, .venv y docker-compose.yml:

```bash
cd GLPI
touch .env
touch docker-compose.yml
```

- .venv --> Sera el archivo que contenga las variables para acceder tanto a GPLI como a la BBD. ( host, puerto, contraseña,nombre de la BBDD, contraseña root de la BBDD...) El contenido sería este:

```nano
GLPI_DB_HOST=mariadb
GLPI_DB_PORT=3306
GLPI_DB_NAME=glpi
GLPI_DB_USER=glpi
GLPI_DB_PASSWORD=glpi
GLPI_DB_ROOT_PASSWORD=mariadb
# Usar contraseñas robustas
```

Cambiamos los permisos para que solo el propietario pueda acceder, ya que tiene contraseñas:

```bash
chmod 600 .env
```

- docker-compose.yml --> Este archivo contendrá las configuraciones del servidor GLPI y la BBDD. El contenido sería este:

```yaml
name: glpi

services:
  glpi:
    image: "glpi/glpi:latest"
    restart: "unless-stopped"
    container_name: glpi
    hostname: glpi
    volumes:
      - data:/var/glpi                  # Directorio donde se guardará los archivos de glpi
    env_file: .env                      # el fichero creado anteriormente con las varibles
    depends_on:
      - mariadb
    ports:
      - "8080:80"                       #  Puerto externo ( el que escuchará la maquina host ) : Puerto interno que utilzará docker

  mariadb:
    image: "mariadb:lts-ubi10"
    restart: "unless-stopped"
    container_name: mariadb
    hostname: mariadb
    volumes:                            # Directorio donde se guardará la BBDD
      - mariadb_data:/var/lib/mysql     
    environment:                        # Variables de la BBDD usando las declaradas en el archivo .env
      MARIADB_ROOT_PASSWORD: ${GLPI_DB_ROOT_PASSWORD}
      MARIADB_DATABASE: ${GLPI_DB_NAME}
      MARIADB_USER: ${GLPI_DB_USER}
      MARIADB_PASSWORD: ${GLPI_DB_PASSWORD}

volumes:
  glpi_data:
  mariadb_data:
```

Fisicamente en el sistema se encuentran en **/var/lib/docker/volumes/**.

## Inicio de los contenedores

Ya con las imágenes y los archivos de configuración, simplemente ejecutamos el siguiente comando en el directorio de trabajo:

```bash
docker compose up -d
```
Deberia imprimir esta salida:

```bash
[+] up 2/2
 ✔ Container mariadb Started                                                                                                                0.2s
 ✔ Container glpi    Started                                                                                                                0.4s
```

Además, si ejecutamos el siguiente comando debería aparacer esto:

```bash
asrc@ServerGLPI:~/GLPI$ docker compose ps
NAME      IMAGE               COMMAND                  SERVICE   CREATED        STATUS          PORTS
glpi      glpi/glpi:latest    "/opt/glpi/entrypoin…"   glpi      45 hours ago   Up 13 seconds   0.0.0.0:8080->80/tcp, [::]:8080->80/tcp
mariadb   mariadb:lts-ubi10   "docker-entrypoint.s…"   mariadb   45 hours ago   Up 13 seconds   3306/tcp
```



