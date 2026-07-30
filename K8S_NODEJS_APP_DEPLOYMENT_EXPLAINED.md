# Kubernetes deployment of the NodeJS TTT app

## Aim

Deploy the tictactoe app (already pushed to Docker Hub) using Kubernetes, first on its own, then again alongside a MongoDB database, comparing to how the same architecture was previously deployed manually, then with Terraform, then with Docker Compose.

## Prerequisites confirmed

- NodeJS app image already on Docker Hub: `hodaahmed18/tech610-tttapp:1.2.0-amd64`
- MongoDB maintained image: `mongo:8.0`, the same official image used in the Docker Compose task

## Part 1, app only, no database

**Folder:** `k8-yaml-definitions/local-nodejs20-app-deploy/`

**Deployment file, `nodejs-deploy.yml`:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nodejs-deployment
spec:
  selector:
    matchLabels:
      app: nodejs-tttapp
  replicas: 3
  template:
    metadata:
      labels:
        app: nodejs-tttapp
    spec:
      containers:
      - name: nodejs-tttapp
        image: hodaahmed18/tech610-tttapp:1.2.0-amd64
        ports:
        - containerPort: 3000
```

Adapted directly from my existing nginx deployment file, the structure stayed identical, only the name, label, image, and port changed to match this app specifically. `containerPort: 3000` matches where this app listens, rather than nginx's port 80.

**Service file, `nodejs-service.yml`:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nodejs-svc
  namespace: default
spec:
  ports:
  - nodePort: 30002
    port: 3000
    targetPort: 3000
  selector:
    app: nodejs-tttapp
  type: NodePort
```

`nodePort: 30002` chose it deliberately different from the existing nginx service's `30001`, so that both could run side by side without a port clash. `selector: app: nodejs-tttapp` connects this service to the pods created by the deployment above by label, the same mechanism used throughout every Kubernetes object.

**Applied with:**
```bash
kubectl apply -f nodejs-deploy.yml
kubectl apply -f nodejs-service.yml
```

**Confirmed working:**
```bash
kubectl get pods
```
All three pods reached `1/1 Running`.

```bash
curl http://$(ip):30002
```

## Part 2, app and database together

**Folder:** `k8-yaml-definitions/local-nodejs20-mongo-deploy/`

**App deployment file, `nodejs-deploy.yml`, same as Part 1 but with the database connection details added as environment variables:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nodejs-deployment
spec:
  selector:
    matchLabels:
      app: nodejs-tttapp
  replicas: 3
  template:
    metadata:
      labels:
        app: nodejs-tttapp
    spec:
      containers:
      - name: nodejs-tttapp
        image: hodaahmed18/tech610-tttapp:1.2.0-amd64
        ports:
        - containerPort: 3000
        env:
        - name: MONGODB_URI
          value: mongodb://mongo-svc:27017/tictactoe
        - name: STATEFUL_MODE
          value: server
```

`mongo-svc` in the connection string is not a real external hostname, it is the name of the MongoDB Service defined below. Kubernetes Services get their own internal DNS name inside the cluster, the same principle as Docker Compose's service-name networking, meaning there is no need to look up or hardcode a real IP address anywhere.

**MongoDB deployment file, `mongo-deploy.yml`:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongo-deployment
spec:
  selector:
    matchLabels:
      app: mongo
  replicas: 1
  template:
    metadata:
      labels:
        app: mongo
    spec:
      containers:
      - name: mongo
        image: mongo:8.0
        ports:
        - containerPort: 27017
```

Only 1 replica, ffrom the task requirement, a single database instance, unlike the app which needs multiple for availability.

**MongoDB service file, `mongo-service.yml`:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongo-svc
  namespace: default
spec:
  ports:
  - port: 27017
    targetPort: 27017
  selector:
    app: mongo
  type: ClusterIP
```

`type: ClusterIP` rather than `NodePort` here, MongoDB only ever needs to be reached by the app itself, from inside the cluster, it has no reason to be reachable from outside at all. `ClusterIP` is the default, internal-only type, and is the more secure, correct choice for a database that should never be directly exposed.

**App's own service file, `nodejs-service.yml`, unchanged from Part 1.**

**Applied in order, database first:**
```bash
kubectl apply -f mongo-deploy.yml
kubectl apply -f mongo-service.yml
kubectl apply -f nodejs-deploy.yml
kubectl apply -f nodejs-service.yml
```

Since the app's deployment file changed (because i added the new environment variables), applying it triggered a rolling update automatically, the existing 3 running pods were replaced gradually with new ones carrying the updated configuration, with zero downtime throughout, super cool Kubernetes thing.

## Confirmed working, app and database together

```bash
curl http://$(ip):30002
```
Footer now showed `Global Scoreboard` and `Mode: Persistent with Mongo DB`, confirming the app successfully connected to MongoDB purely through the Kubernetes Service's internal DNS name, with 3 app replicas and 1 database replica all running correctly together.

## Kubernetes architecture for this deployment

```
              Control plane (master node)
   +---------------------------------------------------+
   |  API server   |  Scheduler  |  Controller manager  |
   |                     etcd                           |
   +---------------------------------------------------+
                          |
                          v
   +-------------------------------------------------------+
   |                    Worker node                        |
   |                                                        |
   |   nodejs-tttapp Pod x3        mongo Pod x1             |
   |   (port 3000)                 (port 27017)             |
   |         ^                          ^                   |
   |         |                          |                   |
   |    nodejs-svc                 mongo-svc                |
   |    NodePort 30002              ClusterIP               |
   |    (external access)        (internal only)            |
   |         |                          |                   |
   |         +---- app reaches DB via mongo-svc:27017 ------+
   +---------------------------------------------------+
```

The nodejs app's pods are reachable from outside the cluster via the NodePort service. The MongoDB pod is only reachable from inside the cluster, via the ClusterIP service, and the app reaches it purely by the service's name, `mongo-svc`, never by IP address.

