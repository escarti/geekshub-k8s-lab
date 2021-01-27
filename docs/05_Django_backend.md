# Django backend

## Repo

En primer lugar bajamos un nivel el directorio y clonamos el repo de django en su versión v1 para poder trabajar

```
{
DIR=$PWD
cd .. 
git clone https://github.com/escarti/geekshub-django.git 
cd geekshub-django 
git checkout v1
cd $DIR 
}
```
### Crear repo nuevo

Vamos a nuestro GitHub y creamos nuevo repo fresco.

Borramos la carpeta .git del repo actual
```
rm -rf .git
```

Y subimos los archivos a nuestro nuevo repo.

Creamos la rama "develop"
```
git checkout -b develop
```

## Despliegue en K8s

Para hacer nuestro despliegue necesitaremos varios elementos.

Crearemos un archivo 'django-deployment.yaml' para agrupar todos los elementos necesarios.

### Servicio 

Para poder acceder a nuestro Django en el cluster de Kubernetes crearemos un servicio que exponga el puerto 8000 de todas las pods con el label "pod: django"

```
apiVersion: v1
kind: Service
metadata:
  name: django-service
  labels:
    app: django
spec:
  selector:
    pod: django
  ports:
    - protocol: TCP
      port: 8000
      targetPort: 8000
  type: NodePort
```

Y ahora configuramos nuestro despliegue usando los secrets que configuramos en el anterior apartado

```
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: django
  labels:
    deployment: django
spec:
  replicas: 1
  selector:
    matchLabels:
      pod: django
  template:
    metadata:
      labels:
        pod: django
    spec:
      containers:
        - name: django
          image: escarti/geekshub-django:v1.0.0
          ports:
            - containerPort: 8000
          env:
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: postgres-credentials
                  key: user

            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-credentials
                  key: password

            - name: POSTGRES_HOST
              value: postgresql-service
```

```
kubectl apply -f django-deployment.yaml
```

Para ver cuando están disponibles las pods usaremos
```
kubectl get pods
```
hasta ver algo así
```
NAME                                     READY   STATUS    RESTARTS   AGE
django-6ccb67cbb9-f87tv                  1/1     Running   0          24s
postgresql-deployment-59df5657fb-qlpfk   1/1     Running   2          22h
```

### Minikube 

Ahora obtendremos la url con 
```
minikube service django-service --url
```
y deberíamos leer el siguiente mensaje
```
This is our super cool Django Deployment
You are currently seeing:

version 1
Enjoy!
```

### EKS

Ejecutamos:
```
kubectl expose deployment django --type=LoadBalancer --name=django-service-alb
```

Para la URL pública vamos a la dashboard vamos a:
```
http://127.0.0.1:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
```
Y de ahí obtendremos la URL para nuestro servicio externo

## Realizar un update

Una de las ventajas de K8s es la capacidad de realizar updates sin downtime.

> BONUS los que quieran practicar pueden editar el repo
En el repo de django

1. Modificar el archivo settings.py y cambiar "VERSION" a "2"

Construir imagen
```
docker build -t ${your_user}/${my_coolname}:v2.0.0 .
```
Para subirla deberéis tener usuario de DockerHub. Sino, simplemente usad escarti/geekshub-django:v2.0.0
```
docker push ${your_user}/${my_coolname}:v2.0.0
```

Editamos el archivo django-deployment.yaml y ponemos la nueva imagen de referencia y applicamos los cambios con
```
kubectl apply -f django-deployment.yaml
```

Y anotamos los cambios
```
kubectl annotate deployment/django kubernetes.io/change-cause="image version 2.0.0"
```

Podremos ver la historia de cambios mediante:
```
kubectl rollout history deployments/django
```

Y en caso de necesidad voler a una versión anterio mediante
```
kubectl rollout undo deployment/django --to-revision=1
```

