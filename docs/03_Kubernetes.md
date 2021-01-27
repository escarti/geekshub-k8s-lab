# Kubernetes Cluster

Inicialmente la práctica estaba pensada para Minikube pero dado que kubectl abstrae la información del clúster también se provee una alternativa con un clúster EKS en AWS.


## 1. Minikube

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

## 2. EKS

Deberemos seguir las instrucciones detalladas [aquí](https://learn.hashicorp.com/tutorials/terraform/eks)

Pero para simplificar las podéis seguir en este mismo readme.

> IMPORTANTE: Si no usáis el perfil de ayer "docker-swarm-aws" deberéis cambiarlo en los archivos de terraform 

Vamos al directorio ./EKS y desplegamos la infraestructura con:

```
cd EKS
terraform init
terraform apply
```

Una vez hecho esto: 

```
aws eks --region $(terraform output -raw region) update-kubeconfig --name $(terraform output -raw cluster_name) --profile docker-swarm-aws
```

Si queréis una DASHBOARD local y el servidor de métricas ejecutad:
```
wget -O v0.3.6.tar.gz https://codeload.github.com/kubernetes-sigs/metrics-server/tar.gz/v0.3.6 && tar -xzf v0.3.6.tar.gz
kubectl apply -f metrics-server-0.3.6/deploy/1.8+/
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta8/aio/deploy/recommended.yaml
kubectl proxy
```

Para ir a la dashboard:
```
http://127.0.0.1:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
```

Para Autenticar la Dashboard:
```
kubectl apply -f https://raw.githubusercontent.com/hashicorp/learn-terraform-provision-eks-cluster/master/kubernetes-dashboard-admin.rbac.yaml
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep service-controller-token | awk '{print $1}')
```
