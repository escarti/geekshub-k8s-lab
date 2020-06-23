# Base de datos

Para poder usar base de datos necesitaremos tener la capacidad de persistir datos.
Para ello usaremos los [PersistentVolumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) de Kubernetes.

## Secrets

El objeto 'Secrets' nor permite guardar información sensible de manera segura en el cluster de Kubernetes.

Primero generemos el archivo db_secrets.yaml

```
{
DB_USER=db_user
DB_PASSWORD=MyStrongPass123!

cat > ./secrets.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: postgres-credentials
type: Opaque
data:
  user: $(echo -n "$DB_USER" | base64)
  password: $(echo -n "$DB_PASSWORD" | base64)
EOF
}
```

Deplegamos los secrets en el cluster

```
kubectl apply -f secrets.yml
```

> IMPORTANTE: Estos credenciales no están encriptados y no deberían subirse a un repositorio sin encriptar previamente. Se puede usar un gestor de secretos como Vault o git-crypt.

Primero creamos el servicio para exponer postgresql-service en nuestro cluster

```
apiVersion: v1
kind: Service
metadata:
  name: postgresql-service
  labels:
    app: postgresql
spec:
  ports:
    - protocol: TCP
      port: 5432
      targetPort: 5432
```

Ahora necesitamos reservar el espacio y reclamarlo para nuestro despliegue.

Crearemos un archivo 'postgres-deployment.yaml' y compiaremos
```
kind: PersistentVolume
apiVersion: v1
metadata:
  name: postgres-pv
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /data/postgres-pv
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgresql-pv-claim
  labels:
    app: postgresql
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
  volumeName: postgres-pv
```

Llegados a este punto podemos hacer un despliegue y verificar que todo esté en orden.

```
kubectl apply -f postgresql-deployment.yaml
```

Si hubiera algún error siempre podemos usar el comando
```
kubectl describe pod/service/pvc
```
para aversiguar lo que ocurre.

Si todo ha ido bien deberíamos poder ejecutar

```
kubectl get pv
```

Y debería devolvernos algo así
```
NAME          CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                         STORAGECLASS   REASON   AGE
postgres-pv   2Gi        RWO            Retain           Bound    default/postgresql-pv-claim   manual                  16m
```

```
kubectl get pvc
```
Nos devolverá
```
NAME                  STATUS   VOLUME        CAPACITY   ACCESS MODES   STORAGECLASS   AGE
postgresql-pv-claim   Bound    postgres-pv   2Gi        RWO            manual         13m
```

y

```
kubectl get svc
```
Devolverá
```
postgresql-service   ClusterIP   10.96.42.177   <none>        5432/TCP         31m
```

> Como curiosidad podemos ver que en la máquina virtual de minikube se ha creado el directorio /data/postgres-pv

```
minikube ssh
```

Y una vez dentro
```
cd /data/postgres-pv/
```

Finalmente añadimos la información del propio contenedor de postgres a nuestro archivo de despliegue
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgresql-deployment
  labels:
    app: postgresql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgresql
      tier: backend
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: postgresql
        tier: backend
    spec:
      containers:
      - image: postgres:12.1-alpine
        name: postgresql
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

        - name: POSTGRES_DB
          value: kubernetes_django

        ports:
          - containerPort: 5432
        volumeMounts:
          - name: postgresql-persistent-storage
            mountPath: /var/lib/postgresql/data

      volumes:
      - name: postgresql-persistent-storage
        persistentVolumeClaim:
          claimName: postgresql-pv-claim
```

> BONUS

Si queremos acceder a la base de datos con alguna aplicación del host, podemos usar port-forwarding.

Con el siguiente comando enviaremos toda la información del puerto local 25432 a deployment/postgresql-deployment en el puerto 5432

```
kubectl port-forward deployment/postgresql-deployment 25432:5432
```

> ENHORABUENA hemos desplegado nuestra base de datos en el cluster de Kubernetes con datos persistentes