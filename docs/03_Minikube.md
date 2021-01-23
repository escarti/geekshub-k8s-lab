# Minikube

## Instalar Minikube

Instalar Minikube 1.6.2 con Kubernetes 1.17.0 y Docker 19.03.5

Seguiremos las instrucciones oficiales detalladas [aquí](https://kubernetes.io/es/docs/tasks/tools/install-minikube/)


En otro terminal ejecutamos
```
minikube start --vm-driver=virtualbox
```

Verificamos que estamos en el contexto de minikube:
```
kubectl config current-context
```

Si la consola no nos devuelve `minikube` deberemos cambiar el contexto de kubectl a minikube
```                     
kubectl config use-context minikube 
```

Más información con:
```
kubectl config get-contexts                          
kubectl config current-context  
```

Desplegamos hello-minikube
```
kubectl create deployment hello-minikube --image=k8s.gcr.io/echoserver:1.10
```

Ahora exponemos el puerto 8080 de hello-minikube al exterior y obtenemos la url
```
kubectl expose deployment hello-minikube --type=NodePort --port=8080
```

Deberemos esperar a que esté desplegado 
```
kubectl get pods
```

cuando lo esté ya podremos obtener su URL
```
minikube service hello-minikube --url
```

Si todo está en orden borraremos el servicio y el despliegue
```
kubectl delete services hello-minikube
kubectl delete deployment hello-minikube
```

Y

```
kubectl get all
```

Debería devolver algo así:
```
NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   221d
```