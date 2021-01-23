# Jenkins

## Instalar Jenkins

Seguiremos las instrucción detalladas en la página ofical de [Jenkins](https://jenkins.io/doc/book/installing/)

Creamos una red para que los dos contenederos que usamos se puedan comunicar
```
docker network create jenkins
```

Crear volúmenes para compartir datos y guardarlos en el "host"
```
docker volume create jenkins-docker-certs
docker volume create jenkins-data
```

Para poder ejecutar docker dentro de Jenkins (dockerizado) instalamos docker:dind (Docker in Docker)
```
docker container run \
  --name jenkins-docker \
  --rm \
  --detach \
  --privileged \
  --network jenkins \
  --network-alias docker \
  --env DOCKER_TLS_CERTDIR=/certs \
  --volume jenkins-docker-certs:/certs/client \
  --volume jenkins-data:/var/jenkins_home \
  --publish 2376:2376 \
  docker:dind
```

Ahora lanzamos jenkins/jenkins:lts-alpine
```
docker container run \
  --name jenkins \
  --rm \
  --detach \
  --network jenkins \
  --dns 8.8.8.8 \
  --env DOCKER_HOST=tcp://docker:2376 \
  --env DOCKER_CERT_PATH=/certs/client \
  --env DOCKER_TLS_VERIFY=1 \
  --env JAVA_OPTS="-Xmx2048m -Djava.awt.headless=true" \
  --publish 8080:8080 \
  --publish 50000:50000 \
  --volume jenkins-data:/var/jenkins_home \
  --volume jenkins-docker-certs:/certs/client:ro \
  jenkins/jenkins:lts-alpine
```

> BONUS crear un archivo de docker compose para automatizar este proceso

Todos estos pasos se pueden automatizar mediante docker-compose. Ver [jenkins.yml](../compose/jenkins.yml)

Navegar al directorio "compose" y ejecutar
```
docker-compose -f jenkins_build.yml up -d
```

> El docker-compose jenkins_build.yml ya instala docker y kubectl en el contenedor por lo que nos ahoramos hacerlo manualmente

Si no hemos destruido o parado los contenedores anteriores, el sistema nos alertará de un conflicto con los puertos
```
WARNING: Host is already in use by another container

ERROR: for compose_jenkins-docker_1  Cannot start service jenkins-docker: driver failed programming external connectivity on endpoint compose_jenkins-docker_1 (f1f6a58539792074c55ec31787226ab836406cd8c2dca0144dd8799303d644e3): Bind for 0.0.0.0:2376 failed: port is already allocated
Creating compose_jenkins_1 ... error

ERROR: for compose_jenkins_1  Cannot start service jenkins: driver failed programming external connectivity on endpoint compose_jenkins_1 (12886eed248bb4cb9305dd46911aa3ca3178f7798c4732423cfa90c8e652bd61): Bind for 0.0.0.0:50000 failed: port is already allocated
```

Deberemos parar los contenedores existente volver a ejecutar
```
docker kill CONTAINER-ID
docker-compose -f compose/jenkins_build.yml up -d
```

Podremos ver los volúmenes creados usando
```
docker volume ls
```

Veremos que tenemos dos versiones de los volúmenes.
```
local     compose_jenkins-data
local     compose_jenkins-docker-certs
local     jenkins-data
local     jenkins-docker-certs
```

Ahora borraremos las que no vayamos a usar con el comando
```
docker volume rm jenkins-data
docker volume rm jenkins-docker-certs
```

## Configuración inicial

Para listar los contenedores activos ejecutamos
```
docker ps
```

Deberíamos ver algo así
```
CONTAINER ID        IMAGE                 COMMAND                  CREATED             STATUS              PORTS                                              NAMES
60f9136cfb3e        jenkins/jenkins:lts-alpine   "/sbin/tini -- /usr/…"   9 minutes ago       Up 8 seconds        0.0.0.0:8080->8080/tcp, 0.0.0.0:50000->50000/tcp   compose_jenkins_1
a134bd24d67c        docker:dind           "dockerd-entrypoint.…"   9 minutes ago       Up 8 seconds        2375/tcp, 0.0.0.0:2376->2376/tcp                   compose_jenkins-docker_1
```

Para instalar jenkins necesitamos una clave que se encuentra dentro del contenedor así que nos conectamos al contenedor de jenkins:lts-alpine
```
docker exec -it 60f9136cfb3e bash
```

Una vez dentro del contenedor obtenemos la clave que necesitamos para completar la instalación
```
cat /var/jenkins_home/secrets/initialAdminPassword
```

Instalamos los plugins más comunes.

Para el usuario admin usaremos:

```
user = admin
password = admin123
```

Aceptar todo hasta llegar a la pantalla de Jenkins.

Ahora pararemos los contenedores y los volveremos a iniciar para asegurarnos de que nuestros datos persisten:
```
docker-compose -f compose/jenkins.yml down
```

y después
```
docker-compose -f compose/jenkins.yml up -d
```

Deberíamos poder hacer login con admin:admin123

> ¡ENHORABUENA! Hemos instalado Jenkins con éxito.

## Instalar docker en Jenkins Alpine

Ahora debemos instalar docker dentro del contenedor de docker-alpine de Jenkins.

Hacemos `docker ps` y obtenemos el HASH del contenedor de Jenkins y ejecutamos:

```
docker exec -u root -it 194fdd323437 bash
```

Una vez dentro del contenedor ejecutamos los siguientes comandos:
```
apk add --update docker openrc
rc-update add docker boot
```

Hecho esto el comando `docker ps` dentro del contenedor debería devolver:

```
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
```

## Plugins

Nos vamos a Jenkins > Administrar jenkins e instalamos los plugins de

1. GitHub Integration
2. GitHub
3. Kubernetes
4. Docker
5. Docker Pipeline
4. Kubernetes CLI

y reiniciamos el servidor

Ahora generamos un Token de GitHub para darle accesso a Jenkins. Visitaremos [github/settings/tokens](https://github.com/settings/tokens)

y marcamos:

- repo (todo)
- admin:repo_hook (todo)

Con ese Token nos vamos a Credentials:

Al pasar el ratón sobre el enlace a Jenkins en el listado, aparece un pequeño triángulo apuntando hacia abajo. Pulsando sobre él se mostrará un menú desplegable con la opción "Add domain".

> **Nota:** En versiones más antiguas de Jenkins la opción "Add domain" está en Credentials > System

Seleccionamos "Add domain" e introducimos `api.github.com`

Ahora navegamos dentro del dominio y seleccionamos "Add Credentials" > Secret Text

Y en ID ponemos "GitHubToken" y en Secret Text introducimos nuestro token.

> Alternativa

También podemos hacerlo directamente en Jenkins a través de

Administrar Jenkins > Configurar > Scroll down hasta 'GitHub Servers' > Add GitHub Server > Avanzado >

Manage Additional actions > Convert login and password to token

# Configure GitHub Server

Navegamos a Administrar Jenkins > Configurar > Scroll down hasta 'GiHub' -> 'GitHub Servers' > Add GitHub Server > Seleccionamos nuestras credenciales de Token y le damos a "Test connection"

Le damos a "Guardar"

# Configure Global Git

Navegamos a Administrar Jenkins > Configurar > Scroll down hasta 'GitHub plugin' e introducimos `Global Config user.name Value` y `Global Config user.email Value`

# Add Git credentials

Nos vamos a Credentials > Sytem > Global credentials > Add credentials

1. Seleccionamos username with password
2. Introducimos datos y como ID usamos GitHub (o Git o lo que nos parezca :) )
3. Damos a Ok

# Add Docker credentials

Nos vamos a Credentials > Sytem > Global credentials > Add credentials

1. Seleccionamos username with password
2. Introducimos datos y como ID usamos Docker
3. Damos a Ok