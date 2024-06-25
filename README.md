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

## JENKINS IMAGE

* Antes de levantar este contenedor creamos un red llamada _i_cd_network_
#### Build NetWork
```sh
docker network create i_cd_network
```

#### __Run Jenkins Imagen__

```sh
docker container run --network=ci_cd_network -u $(stat -c '%u:%g' /var/run/docker.sock) -d -p 8080:8080 -v jenkinsvol:/var/jenkins_home -v /var/run/docker.sock:/var/run/docker.sock --name jenkins-local jenkins/jenkins:lts
```

- --network=ci_cd_network: Conecta el contenedor a una red Docker existente llamada ci_cd_network. Esto permite que el contenedor se comunique con otros contenedores en la misma red.

- -u $(stat -c '%u:%g' /var/run/docker.sock): Establece el usuario y grupo del contenedor al mismo UID y GID que el archivo /var/run/docker.sock en el host. Esto es útil para permitir que el contenedor interactúe con el Docker daemon del host con los mismos permisos que el archivo docker.sock.

- -d: Ejecuta el contenedor en modo "detached" (desconectado), lo que significa que el contenedor se ejecuta en segundo plano.

- -p 8080:8080: Mapea el puerto 8080 del contenedor al puerto 8080 del host. Esto permite acceder a Jenkins desde el navegador web usando el puerto 8080 del host.

- -v jenkinsvol:/var/jenkins_home: Monta un volumen Docker llamado jenkinsvol en /var/jenkins_home dentro del contenedor. /var/jenkins_home es el directorio donde Jenkins almacena su configuración y datos, por lo que montar un volumen aquí permite que esos datos persistan entre reinicios del contenedor.

- -v /var/run/docker.sock:/var/run/docker.sock: Monta el socket de Docker del host dentro del contenedor. Esto permite que el contenedor ejecute comandos Docker, efectivamente controlando el Docker daemon del host. Es útil para crear y gestionar contenedores desde dentro del contenedor de Jenkins.

- --name jenkins-local: Asigna el nombre jenkins-local al contenedor. Esto es útil para identificar y gestionar el contenedor con comandos Docker.

- jenkins/jenkins:lts: Especifica la imagen Docker a usar, en este caso, la versión "Long Term Support" (LTS) de Jenkins

### Pasos de configuracion previos a ejecutar los Piplelines
- Para poder correr piplelines que ejecuten comandos de docker debemos una vez levantado y configurado nuestro jenkins dentro del contenedor ejecutar estos comando para que pueda ver el docker local nuestro

### Instalacion Docker en Contenedor
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
### Dar Permisos a jenkins
```sh
usermod -aG docker jenkins
```
### En caso de necesitar Compose en nuestro jenkins
```sh
curl -L "https://github.com/docker/compose/releases/download/1.27.4/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose && chmod +x /usr/local/bin/docker-compose
```