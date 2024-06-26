pipeline {
    environment {
        /* Definimos las variables de entorno */
        IMAGEN = "nginx" /* Cambiamos el nombre de la imagen de Docker a nginx */
        USUARIO = 'NEXUS_CREDENTIAL' /* Nombre de usuario de Docker Hub */
        NEXUS_GROUP= 'grupo13/pin1' /* Grupo de PIN */
    }
    agent any /* Indicamos que el agente puede ser cualquiera de los disponibles, en este caso el que tenga Docker instalado */
    stages { /* Definimos las etapas del pipeline */
        stage('Clone') { /* Etapa de clonación del repositorio */
            steps {
                git branch: "main", url: 'https://github.com/berellajuan/pipileline-deploy-nexus-repository.git'
            }
        }
        stage('Build') { /* Etapa de construcción de la imagen de Docker */
            steps {
                script {
                    newApp = docker.build "$NEXUS_GROUP/$IMAGEN:$BUILD_NUMBER" /* Construimos la imagen de Docker con un nombre único basado en el número de compilación */
                }
            }
        }
        stage('Security Scan') { /* Etapa de escaneo de seguridad */
            steps {
                script { /* Ejecutamos un contenedor de Trivy para escanear la imagen de Docker */
                    sh '''
                    docker run --rm -v /var/run/docker.sock:/var/run/docker.sock -v $HOME/.cache:/root/.cache/ aquasec/trivy:latest image $NEXUS_GROUP/$IMAGEN:$BUILD_NUMBER
                    ''' 
                }
            }
        }
        stage('Test') { /* Etapa de pruebas */
            steps {
                script {
                    // Iniciamos el contenedor mapeando el puerto 80 del contenedor al puerto 8080 del host
                    sh "docker run -d -p 8080:80 --name test_container $NEXUS_GROUP/$IMAGEN:$BUILD_NUMBER nginx -g 'daemon off;'"
                    // Esperamos un poco para asegurarnos de que el contenedor esté completamente arriba y corriendo
                    sh "sleep 5"
                    // Verificamos la versión de nginx dentro del contenedor
                    sh "docker exec test_container nginx -v"
                    // Realizamos una petición HTTP al contenedor
                    sh "curl -I http://localhost:8080"
                    // Detenemos y eliminamos el contenedor después de las pruebas
                    sh "docker stop test_container"
                    sh "docker rm test_container"
                }
            }
        }
                
        stage('Deploy') {
            steps {
                script {
                    docker.withRegistry( 'http://127.0.0.1:8083', USUARIO ) { /* Autenticamos con Nexus Local utilizando el nombre de usuario */
                        newApp.push() /* Subimos la imagen de Docker a Nexus -> docker-hosted */
                    }
                }
            }
        }
        
        stage('Clean Up') { /* Etapa de limpieza */
            steps {
                sh "docker rmi -f $NEXUS_GROUP/$IMAGEN:$BUILD_NUMBER" /* Eliminamos la imagen de Docker localmente */
            }
        }
    }
}