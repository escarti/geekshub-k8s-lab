# Jenkins (Part 2)

## Docker-Hub registry

Antes de empezar vamos a necesitar un repo en docker-hub al que subir nuestra imágenes.

Vamos a [Docker Hub](https://hub.docker.com/repository/create) y creamos un repo de nombre `geekshub-django`

## Crear MultiBranch Pipeline

Vamos a nuestro Jenkins (probablemente [http://localhost:8080](http://localhost:8080)) y creamos "Nueva Tarea"

Ponemos un nombre (p.ej. "Geekshubs Django") y seleccionamos una "MultiBranch Pipeline"

1. En Branch sources seleccionamos GitHub y añadimos nuestra URL y nuestros credenciales
2. En Declarative Pipeline (Docker) introducimos la dirección de docker registry `https://registry.hub.docker.com` y seleccionamos nuestros credenciales en el dropdown
3. Guardamos y le damos a "Scan Repository Now"

Ahora nos vamos al repo donde tenemos nuestro backend de Django y creamos un archivo de tipo Jenkinsfile
```
touch Jenkinsfile
```

```
pipeline {
    agent any
    triggers {
        /* Vamos a pullear el repo activamente porque en nuestra instalación local sería complicado usar Webhooks */
        pollSCM('* * * * */1')
    }

}
```

Cargamos las variables de entorno de Docker añadiendo debajo de triggers
```
    environment {
        registry = "escarti/geekshub-django" /* Usad vuestro docker-hub registry */
        registryCredential = 'Docker'
        imageTag = "${env.GIT_BRANCH + '_' + env.BUILD_NUMBER}"

    }
```

Añadimos una primera fase de tests "dummy"
```
        stages {
                stage('Test') {
                    steps {
                        echo 'Testing..'
                    }
                }
        }
```

Ahora queremos que Jenkins se encarge de construir las imágenes de Docker y subirlas a nuestro registry
```
        stage('Build image') {
            steps {
                script {
                    dockerImage = docker.build(registry + ":$imageTag", "--cache-from $registry:latest --network host .")
                }
            }
        }
        stage('Deploy Image') {
            steps{
                script {
                    docker.withRegistry( 'https://registry.hub.docker.com', registryCredential ) {
                        dockerImage.push()
                        dockerImage.push('latest')
                    }
                }
            }
        }
```

También nos gustaría que nuestro Jenkins pudiera desplegar en nuestro cluster de Kubernetes, pero aún nos queda un poco para eso primero hay que configurar...

## Acceso externo al cluster

Creamos una ServiceAccount con nombre `jenkins-robot` en nuestro namespace
```
NAMESPACE=default
kubectl -n $NAMESPACE create serviceaccount jenkins-robot
```

Ahora le damos al robot de jenkins permiso para administrar el cluster
```
kubectl -n $NAMESPACE create rolebinding jenkins-robot-binding --clusterrole=cluster-admin --serviceaccount=$NAMESPACE:jenkins-robot
```

Mostrar el token para usarlo en Jenkins
```
TOKEN=$(kubectl -n $NAMESPACE get secrets $(kubectl -n $NAMESPACE get serviceaccount jenkins-robot -o go-template --template='{{range .secrets}}{{.name}}{{"\n"}}{{end}}') -o go-template --template '{{index .data "token"}}' | base64 -d)
```

Mostrar la dirección de nuestro API server de minikube
```
APISERVER=$(kubectl config view --minify | grep server | cut -f 2- -d ":" | tr -d " ")
```

### Crear los credenciales del token en Jenkins

Vamos a Jenkins > Credentials > System > Global credentials (unrestricted) y creamos un credencial de tipo "Secret Text" con ID `minikube-auth-token` y pegamos el resultado de `echo $TOKEN`.

## Instalar kubectl en el contenedor de Jenkins

entrar al contenedor en modo root
```
docker exec -u root -it 19ec2796397e bash
```

Navegar a /usr/local/bin e instalar kubectl
{
cd /usr/local/bin
curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.17.0/bin/linux/amd64/kubectl
chmod +x ./kubectl
}

Para verificar ejecutar
```
kubectl version
```

> BONUS -> Hacer esto mismo modificando el docker compose de jenkins para que se instale automáticamente kubectl. Solucíon [aquí](99_Solutions.md).

## Añadir despliegue en K8s

Para despleguar haremos uso del plugin de Kubernetes que hemos instalado y añadimos esto a nuestro Jenkinsfile.

Primero introduciremos las variables en la sección correspondiente:
```
    environment {
        registry = "escarti/geekshub-django"
        registryCredential = 'docker-registry'
        apiServer = "https://192.168.99.101:8443"
        devNamespace = "default"
        minikubeCredential = 'minikube-auth-token'
        imageTag = "${env.GIT_BRANCH + '_' + env.BUILD_NUMBER}"
    }
```

Y añadimos
```
        stage('Deploy to K8s') {
            steps{
                withKubeConfig([credentialsId: minikubeCredential,
                                serverUrl: apiServer,
                                namespace: devNamespace
                               ]) {
                    sh 'kubectl set image deployment/django django="$registry:$imageTag" --record'
                }
            }
        }
```

> BONUS introducir la dirección de API-SERVER como un credencial de Jenkins sin necesidad de "hard-codearlo" en el archivo

El problema es que esto nos hará un deploy en el cluster independientemente de la rama y nosotros probablemente no queramos eso. Por eso añadimos una condición "when" a nuestra fase:
```
            when {
                expression { env.GIT_BRANCH == 'develop' }
            }
```

El archivo final debería quedar así:
```
pipeline {
    agent any
    triggers {
        pollSCM('')
    }
    environment {
        registry = "escarti/geekshub-django"
        registryCredential = 'docker-registry'
        apiServer = "https://192.168.99.101:8443"
        devNamespace = "default"
        minikubeCredential = 'minikube-auth-token'
        imageTag = "${env.GIT_BRANCH + '_' + env.BUILD_NUMBER}"
    }
    stages {
        stage('Test') {
            steps {
                echo 'Testing..'
            }
        }
        stage('Build image') {
            steps {
                script {
                    dockerImage = docker.build(registry + ":$imageTag", "--cache-from $registry:latest --network host .")
                }
            }
        }
        stage('Upload to registry') {
            steps{
                script {
                    docker.withRegistry( '', registryCredential ) {
                        dockerImage.push()                                                dockerImage.push('latest')
                    }
                }
            }
        }
        stage('Deploy to K8s') {
            when {
                expression { env.GIT_BRANCH == 'develop' }
            }
            steps{
                withKubeConfig([credentialsId: minikubeCredential,
                                serverUrl: apiServer,
                                namespace: devNamespace
                               ]) {
                    sh 'kubectl set image deployment/django django="$registry:$imageTag" --record'
                }
            }
        }
    }
}
```

## Lanzamiento automático de los trabajos

En una situación normal, el plugin de Git que hemos instalado se encarga de configurar los webhooks necesarios para que se lancen los trabajos, sin embargo en nuestro caso la instalación es local y tenemos que hacer un pequeño "hack"

Modificamos la condición de Poll a
```
    triggers {
        pollSCM('* * * * */1')
    }
```

Esto preguntará al repo cada minuto si hay cambios y lanzará el trabajo en caso afirmativo.

> NO HACER ESTO EN CASA ;)

## Funcionamiento

Ahora que ya está todo configurado podemos modificar el valor de la versión en nuestro archivo de settings y ver cómo se lanza.

En el repo del backend > ir a geekshub/settings.py y cambiar la variable "VERSION" por el número que queramos.

## Deployment patch

Ahora vemos que podemos cambiar las versiones de nuestro contenedor en el pod del deployment pero no es muy elegante:

1. No sabemos si el update ha funcionado
2. No sabemos cuándo se han cambiado las versiones
3. No sabemos quién ha cambiado las versiones

Para resolver todos estos problemas usaremos un deployement patch.

### Repositorio

Creamos un repositorio para guardar nuestro archivo de deployment de develop en git y le ponemos un nombre parecido a geekshub-django-deployment.

Ahora en Jenkins descargaremos el archivo de deployment.

> IMPORTANTE: Añadir nombre e email para GIT en Jenkins. Para ello navegamos a Administrar Jenkins > Configurar > Scroll down hasta 'Git plugin' e introducimos `Global Config user.name Value` y `Global Config user.email Value`
> Si vuestra contraseña contiene carácteres especiales deberéis crear un credencial nuevo con vuestra contraseña en URLENCODED. En mi caso la he llamado `Git-Encoded`. La docu la dejo con el ID `GitHub`original

Añadimos esta fase:

```
stage('Update deployment file') {
            when {
                expression { env.GIT_BRANCH == 'develop' }
            }
            steps{
                script {
                    withCredentials([usernamePassword(credentialsId: 'GitHub', usernameVariable: 'username', passwordVariable: 'password')]){
                        sh "rm -rf geekshub-django-deployment"
                        sh "git clone https://$username:$password@github.com/escarti/geekshub-django-deployment.git"
                        dir("geekshub-django-deployment") {
                            sh "echo \"spec:\n  template:\n    spec:\n      containers:\n        - name: django\n          image: ${registry}:$imageTag\" > patch.yaml"
                            sh "kubectl patch --local -o yaml -f django-deployment.yaml -p \"\$(cat patch.yaml)\" > new-deploy.yaml"
                            sh "mv new-deploy.yaml django-deployment.yaml"
                            sh "rm patch.yaml"
                            sh "git add ."
                            sh "git commit -m\"Patched deployment for $imageTag\""
                            sh "git push https://$username:$password@github.com/escarti/geekshub-django-deployment.git"
                        }
                    }
                }
            }
        }
```

> MUCHO OJO con usar el plugin de "git" con varios repos dentro de una misma PIPELINE, Jenkins escaneará todos los repos que encuentre

Y modificamos la fase de despliegue:

```
        stage('Deploy to K8s') {
            when {
                expression { env.GIT_BRANCH == 'develop' }
            }
            steps{
                withKubeConfig([credentialsId: minikubeCredential,
                                serverUrl: apiServer,
                                namespace: devNamespace
                               ]) {
                    sh 'kubectl apply -f geekshub-django-deployment/django-deployment.yaml'
                }
            }
        }
```

Podemos comprobar que todo haya ido bien usando el comando

```
kubectl rollout status deployments/django
```

> EJERCICIO 1: Añadir una fase de Jenkins que compruebe que el rollout ha acabao con éxito y si no ha sido así, que falle el JOB de Jenkins
