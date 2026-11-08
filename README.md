# GLPI-Install-Guide
Guía para la instalacion de un servidor GLPI y un agente simple en Ubuntu Server.

## ÍNDICE
- [Primeros pasos](#primeros-pasos)
- [Instalación y configuración docker](#instalación-y-configuracion-docker)
- [Descarga de imagenes](#descarga-de-imagenes)
- [Configuración de los contenedores](#configuración-de-los-contenedores)
- [Inicio de los contenedores](#inicio-de-los-contenedores)
- [Instalación de un agente glpi en otro servidor](#instalación-de-un-agente-glpi-en-otro-servidor)
- [Pasos finales para el servidor](#pasos-finales-para-el-servidor)

## Primeros pasos

Para que no haya fallos en la conexión de la máquina, recomiendo permitir los siguiente puertos en el firewall:

```bash
sudo ufw allow 3306/tcp comment "MariaDB"
sudo ufw allow 8080/tcp comment "GLPI Server"
sudo ufw allow 2222/tcp comment "SSH ( OPCIONAL )"

sudo ufw reload
```

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

Con esto debería estar funcionando correctamente GLPI. Desde otra maquina en la misma red, entra en un navegador con la siguiente url:

```nano
http://IPDETUSERVIDOR:8080/
# El puerto puedes cambiarlo en el archivo docker-compose.yml.
```

Debería aparecer la web de glpi. El usuario y contraseña por defecto son:

```nano
usuario: glpi
contraseña: glpi
```

## Pasos finales para el servidor

Para que los clientes puedan eviar su inventario al servidor, vamos a ***'Administration -> Inventory'*** y  habilitaremos la opción 'inventory enabled' en el servidor. Además, para aumentar la seguridad, habilitaremos la autenticación por clave:

![Pasos finales](./img/glpi_server1.PNG)

Después, iremos a ***'Setup -> OAuth clients'*** y creamos tantas claves como clientes tengamos. El procedimiento es el siguiente:

* Añadimos un nuevo cliente.

![Pasos finales](./img/glpi1.PNG)

* Elegimos un nombre . En el apartado **'scopes'** seleccionamos inventory. En el apartado **'Grants'** seleccionamos client credentials. Debería quedar así:

![Pasos finales](./img/glpi2.PNG)

* Le damos a **'+ Add'**. Si volvemos a OAuth clients y entramos en que acabamos de crear, veremos el **ID** y el **Secret** del cliente. Los utilizaremos más tarde para la creación del agente.

## Instalación de un agente glpi en otro servidor

### Instalación

Es recomendable instalarlo mediante perl. Para ello, descargamos el instalador con el siguiente enlace: 

<https://github.com/glpi-project/glpi-agent/releases/download/1.21/glpi-agent-1.21-linux-installer.pl>

Para instalarlo introducimos el siguiente comando:

```bash
# Si no tienes instalado pearl
sudo apt install pearl

perl glpi-agent-1.21-linux-installer.pl
```

### Configuración

El archivo de configuración se encuentra en la siguiente ruta:

```nano
/etc/glpi-agent/conf.d/00-install.cfg
```

Un contenido simple sería este:

```nano
server = http://IPSERVIDOR:8080
tag = cliente_A                    # Etiqueta para diferenciar entre distintos clientes ( OPCIONAL )
task =                             # Tareas a realizar

# Autenticación por clave
oauth-client-id = 
oauth-client-secret = 
```

Las tareas puden ser de distintos tipos:

| Valor | Descripción / Función |
| :--- | :--- |
| `all` | *(Por defecto)* Ejecuta todas las tareas disponibles asignadas por el servidor GLPI |
| `inventory` | Realiza el inventario local de hardware, software y sistema operativo |
| `deploy` | Permite el despliegue de paquetes de software y ejecución de scripts remotos |
| `network` | Escaneo de red (SNMP / NetDiscovery / NetInventory) para impresoras, switches, etc. |
| `collect` | Extrae archivos o claves de registro específicos del equipo |
| `esx` | Realiza el inventario de hipervisores VMware ESX/ESXi mediante API |
| `wakeonlan` | Envía paquetes Magic Packet (WOL) para encender otros equipos de la red |
| `none` | Desactiva las tareas automáticas (útil para ejecuciones manuales o pruebas) |


Múltiples tareas se separan por comas.

