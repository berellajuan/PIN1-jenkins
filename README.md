PIN1 MundosE - Grupo 13
===

## Objetivos
* Instalar docker
* Lanzar un contenedor custom de Jenkins
* Correr un pipeline con ciertas customizaciones
* Crear una imagen y enviarla a nuestro registry propio
* Scanear esta imagen

## Desarollo
La idea es realizar una prueba utilizando la imagen de NGINX para cumplir los objetivos en los siguientes steps:
* __Clone__ : utilizaremos un repositorio de github privado donde tendremos el Pipleline y un DockerFile.
* __Build__: realizaremos el build del DockerFile el mismo tiene la imagen de nginx
* __Security Scan__: realizaremos un scan de vulnerabilidades de la imagen que construimos
* __Test:__ probaremos que la imagen este fucionando correctamente pidiendo la version de la misma dentro del contenedor
* __Deploy__: desplegaremos la imagen a nuestro repositorio local de NEXUS previamente configurado para recibir imagenes.

## NEXUS REPOSITORY IMAGE
* Crear un repositorio de tipo hosted dandole un puerto para poder conectarnos
* Realms: configurarlo para que se pueda logear con docker seleccionando Docker Bearer Token

### Comandos para levantar un contenedor de Nexus
* Agregar credeciales de GITHUB y NEXUS REPOSITORY o DockerHub
* Antes de levantar este contenedor creamos un red llamada _i_cd_network_
#### Build NetWork
```sh
docker network create i_cd_network
```
#### __Run Nexus Imagen__

```sh
docker run --network=ci_cd_network  -d -p 8081:8081 -p 8082:8082 -p 8083:8083 --name nexus-repository -v nexus-data:/nexus-data sonatype/nexus3
```
#### Configurara Nexus para ir por http
```sh
echo '{ "insecure-registries": [ "IP_DE_NEXUS:8082","IP_DE_NEXUS:8082" ] }' > /etc/docker/deamon.json | systemctl reaload docker
```
## JENKINS IMAGE
* Agregar el Plugin de Docker a Jenkins
* Reutilizamos la red __i_cd_network__ previamente creada para conectar el contenedor de Jenkins con el de Nexus

#### __Run Jenkins Imagen__

```sh
docker container run --network=ci_cd_network -u $(stat -c '%u:%g' /var/run/docker.sock) -d -p 8080:8080 -v jenkinsvol:/var/jenkins_home -v /var/run/docker.sock:/var/run/docker.sock --name jenkins-local jenkins/jenkins:lts
```

| Comando                                 | Descripcion                                                                                                                                                                                                                   |
|----------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| --network=ci_cd_network                | Connects the container to an existing Docker network named ci_cd_network. This allows the container to communicate with other containers on the same network.                                                                 |
| -u $(stat -c '%u:%g' /var/run/docker.sock) | Sets the user and group of the container to the same UID and GID as the /var/run/docker.sock file on the host. This is useful to allow the container to interact with the Docker daemon on the host with the same permissions as the docker.sock file. |
| -d                                     | Runs the container in detached mode, meaning the container runs in the background.                                                                                                                                           |
| -p 8080:8080                           | Maps port 8080 of the container to port 8080 of the host. This allows accessing Jenkins from a web browser using the host's port 8080.                                                                                      |
| -v jenkinsvol:/var/jenkins_home        | Mounts a Docker volume named jenkinsvol to /var/jenkins_home inside the container. /var/jenkins_home is the directory where Jenkins stores its configuration and data, so mounting a volume here allows those data to persist between container restarts. |
| -v /var/run/docker.sock:/var/run/docker.sock | Mounts the host's Docker socket inside the container. This allows the container to execute Docker commands, effectively controlling the host's Docker daemon. It is useful for creating and managing containers from within the Jenkins container. |
| --name jenkins-local                    | Assigns the name jenkins-local to the container. This is useful for identifying and managing the container with Docker commands.                                                                                              |
| jenkins/jenkins:lts                    | Specifies the Docker image to use, in this case, the Long Term Support (LTS) version of Jenkins.                                                                                                                            |

### Pasos de configuracion previos a ejecutar los Piplelines
- Para poder correr piplelines que ejecuten comandos de docker debemos una vez levantado y configurado nuestro jenkins dentro del contenedor ejecutar estos comando para que pueda ver el docker local nuestro

#### Instalacion Docker en Contenedor
```sh
apt-get update && \
apt-get -y install apt-transport-https \
     ca-certificates \
     curl \
     gnupg2 \
     software-properties-common && \
curl -fsSL https://download.docker.com/linux/$(. /etc/os-release; echo "$ID")/gpg > /tmp/dkey; apt-key add /tmp/dkey && \
add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/$(. /etc/os-release; echo "$ID") \
   $(lsb_release -cs) \
   stable" && \
apt-get update && \
apt-get -y install docker-ce


```
#### Dar Permisos a jenkins
```sh
usermod -aG docker jenkins
```
#### En caso de necesitar Compose en nuestro Jenkins
```sh
curl -L "https://github.com/docker/compose/releases/download/1.27.4/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose && chmod +x /usr/local/bin/docker-compose
```