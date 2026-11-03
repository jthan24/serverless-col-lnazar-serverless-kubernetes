# serverless-col-lnazar-serverless-kubernetes

## Instalacion de k9s (opcional)
```bash
wget https://github.com/derailed/k9s/releases/download/v0.51.0/k9s_linux_amd64.deb
dpkg -i k9s_linux_amd64.deb
```

## Instalacion de keda
```bash

# agregar el repositorio
helm repo add kedacore https://kedacore.github.io/charts  
helm repo update

# Instalacion del CRD y del operador
helm install keda kedacore/keda --namespace keda --create-namespace

```

### Validacion de la instalacion
```bash
kubectl get pods -n keda -w
```

## Despliegue de una APP

### Despliegue de la infra requerida
`mongo-infra.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongo
spec:
  ports:
    - port: 27017
      targetPort: 27017
  type: LoadBalancer
  selector:
    app: mongo
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongo-pv-claim
  labels:
    app: mongo
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi

---
apiVersion: v1
kind: Secret
metadata:
  name: mongo-secret
type: Opaque
data:
  mongo-root-password: YWRtaW4xMjM=
  mongo-root-username: YWRtaW4=
  
---
apiVersion: apps/v1 
kind: Deployment
metadata:
  name: mongo
  labels:
    app: mongo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongo
  template:
    metadata:
      labels:
        app: mongo
    spec:
      dnsPolicy: ClusterFirst
      containers:
      - image: mongo
        name: mongo
        ports:
        - containerPort: 27017
        volumeMounts:
        - name: mongo-persistent-storage
          mountPath: /data/db/mongo
        env:
        - name: MONGO_INITDB_ROOT_USERNAME
          valueFrom:
            secretKeyRef:
              name: mongo-secret
              key: mongo-root-username
        - name: MONGO_INITDB_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mongo-secret
              key: mongo-root-password
      volumes:
      - name: mongo-persistent-storage
        persistentVolumeClaim:
          claimName: mongo-pv-claim 
```

Aplicar la infra
`kubectl apply -f mongo-infra.yaml`

### Despliegue de la app
`mongo-app.yaml`
```yaml
apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
kind: Deployment
metadata:
  name: dummy-mongo
  labels:
    app: dummy-mongo
spec:
  replicas: 0
  selector:
    matchLabels:
      app: dummy
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: dummy
    spec:
      terminationGracePeriodSeconds: 5
      containers:
        - name: dummy
          image: docker.io/karthik3030/kedadummyserver:latest
          imagePullPolicy: IfNotPresent
          lifecycle:
            preStop:
              exec:
                command:
                  - pkill -f "sleep"
```

Aplicar la app
`kubectl apply -f mongo-app.yaml`

### desplegar un proxy con la bd
`mongo-proxy.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: server
  labels:
    app: server
spec:
  ports:
    - port: 3232
  selector:
    app: server
    tier: frontend
  type: LoadBalancer
---
apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
kind: Deployment
metadata:
  name: server
  labels:
    app: server
spec:
  selector:
    matchLabels:
      app: server
      tier: frontend
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: server
        tier: frontend
    spec:
      containers:
        - name: master
          image:  docker.io/karthik3030/mongoscaleserver:latest2
          imagePullPolicy: Always
          args:
            - app
          ports:
            - containerPort: 3232
              name: server
          resources:
            limits:
              cpu: "0.3"
```
Aplicar el proxy
`kubectl apply -f mongo-proxy.yaml`


## Desplegar el objeto de keda
`mongo-keda.yaml`
```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: mongo-scaledobject
  namespace: default
  labels:
    deploymentName: dummy-mongo
spec:
  maxReplicaCount: 5
  pollingInterval: 5  # Optional. Default: 30 seconds
  cooldownPeriod:  5 # Optional. Default: 300 seconds
  scaleTargetRef:
    name: dummy-mongo
  triggers:
    - type: mongodb
      metadata:
        host: "10.1.1.1" #expects some number needs to be fixed..
        port: "27017"
        dbName: "admin"
        collection: "scalecollection"
        query: '{"name":"test"}'
        queryValue: "1"
        activationQueryValue: "0"
      authenticationRef:
        name: mongodb-local-trigger

---

apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: mongodb-local-trigger
spec:
  secretTargetRef:
    - parameter: connectionString
      name: mongodb-local-secret
      key: connect

---

apiVersion: v1
kind: Secret
metadata:
  name: mongodb-local-secret
type: Opaque
data:
  connect: bW9uZ29kYjovL2FkbWluOmFkbWluMTIzQG1vbmdvLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWw6MjcwMTcvYWRtaW4=
```

## Validacion del serverless
```bash

kubectl get pods -w
```

> con k9s
```bash
k9s
```

### Agergar eventos a la BD de mongo
```bash
kubectl exec $(kubectl get pods | grep "server" | cut -f 1 -d " ") -- keda-talk mongo insert
```


### Eliminar los eventos de la BD
```bash
kubectl exec $(kubectl get pods | grep "server" | cut -f 1 -d " ") -- keda-talk mongo delete
```

### Validar los logs del operador
```bash
kubectl logs -n keda -l app=keda-operator
```


## Referencias
https://github.com/kedacore/samples