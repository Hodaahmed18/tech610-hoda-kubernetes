# Kubernetes, intro session

## Biggest challenges facing enterprises

- Modernise legacy applications
- Migrating to the cloud, or back to on-prem
- Costs of cloud and AI use
- Security and compliance, more services means a bigger attack surface, AI is also being used to assist with hacks, and there is a need to protect data as well as applications and services
- Understanding how to make best use of AI
- Containerisation, leading to Kubernetes

## Why Kubernetes is needed

Managing containers, especially at scale. Docker Compose showed how to run a couple of containers together on one machine, using one file to describe them. Kubernetes is the same underlying idea, taken much further, coordinating potentially hundreds or thousands of containers, across many different physical machines, deciding where each one runs, restarting anything that fails, and scaling automatically based on real demand.

## Benefits of Kubernetes

- Orchestrates, schedules, and manages containers at scale
- Open source
- Can run anywhere, cloud agnostic, the same concepts work on AWS, Azure, GCP, or on-premises
- Self-healing, automatically detects and replaces failed containers or nodes
- Auto scaling
- Load balancing
- Rolling updates and rollbacks, updates can be applied gradually with zero downtime, and reversed instantly if something goes wrong
- Declarative, the same underlying principle as Terraform, describe the desired state and Kubernetes works to keep reality matching it, rather than manually running commands step by step
- Designed for production with no single point of failure

## Success stories

- Spotify, runs its backend infrastructure on Kubernetes, managing thousands of microservices at scale
- Airbnb, migrated much of its infrastructure to Kubernetes to handle unpredictable, highly variable booking traffic
- Pinterest, moved to Kubernetes to improve deployment speed and reduce infrastructure costs
- The New York Times, migrated to modernise legacy infrastructure and speed up feature deployment
- CERN, uses Kubernetes to manage compute infrastructure processing enormous data volumes from the Large Hadron Collider
- Pokemon GO (2015), ran on Kubernetes on Google's infrastructure, exceeded the load expected by 50 times, and was still able to handle it

Every one of these adopted Kubernetes to directly solve one of the challenges listed at the top, unpredictable scaling, managing many microservices, or modernising legacy systems, not simply because it is popular.

## Kubernetes architecture

A cluster is a set of machines, called nodes, that run container apps. A cluster is made up of two kinds of node, working together.

**Control plane, has the master node(s), makes decisions, never runs application containers directly:**
- API server, the central hub for all communication, every request, whether from kubectl or anything else, goes through here
- Controller manager, monitors for changes needed in the cluster, if a change is needed, it goes through the API server to actually make it happen, rather than acting directly itself
- Scheduler, creates worker nodes based on resource availability, and decides which worker node a new pod should run on
- etcd, a database that keeps track of everything happening in the cluster, using key-value pairs. If etcd is lost, the cluster loses its memory of itself entirely

**Data plane, has the worker node(s), where containers actually run:**
- kubelet, connects directly to and receives instructions from the API server on the control plane, making sure the containers that should be running on this node genuinely are
- kube-proxy, controls network routing, making sure traffic reaches the correct pod
- Container engine, the actual software, such as Docker, that runs the containers
- Pods, the containers themselves, running inside

The core relationship to remember, the control plane decides and instructs, the data plane, the worker nodes, actually does the work.

```
                Control plane (master node)
   +---------------------------------------------------+
   |  API server   |  Scheduler  |  Controller manager  |
   |                     etcd                           |
   +---------------------------------------------------+
                     |                    |
       (schedules workloads onto worker nodes)
                     |                    |
     +----------------------+   +----------------------+
     |     Worker node 1    |   |     Worker node 2     |
     |   [Pod]      [Pod]   |   |   [Pod]      [Pod]    |
     |  kubelet, kube-proxy |   |  kubelet, kube-proxy  |
     +----------------------+   +----------------------+
```

## The cluster setup

- A cluster is made up of at least one master node
- Must have at least one worker node
- Production wants a multi-node setup, multiple workers, so no single machine failing takes the whole system down
- Dev and testing can use a single-node setup, one worker node, fine for learning but genuinely fragile

## Kubernetes objects

- **Deployment**, contains a ReplicaSet and Pods, this is usually the object actually interacted with directly, rather than managing Pods or ReplicaSets by hand. Handles rolling updates and rollbacks.
- **ReplicaSet**, can contain pods, and can replicate them, ensuring a specified number of identical pods are always running, if one dies, it gets replaced automatically.
- **Pod**, the smallest component of Kubernetes. A container is not the same thing as a pod, a pod is the wrapper around one or more containers. Usually a pod holds just one container, but it can hold more than one, for example a second container acting as a supporting service like logging, running alongside the main application container. A pod does not need to be inside a ReplicaSet at all, it can exist standalone, a ReplicaSet is only needed if duplication or redundancy is wanted. Pods are ephemeral.
- **Service**, used to either expose pods to the outside world, or to expose pods for communication between other pods inside the cluster.
- **Volume**, used for persistent storage.

## What "ephemeral" means for a pod

A pod is disposable and temporary by design. It can be destroyed and recreated at any time, and when that happens it gets a new identity entirely, a new IP address, a fresh internal state. Nothing about a pod is meant to be permanent. This connects directly to the volumes lesson from Docker Compose, if data genuinely needs to survive, it has to live outside the pod itself, in a separate, persistent storage layer, the exact same reason MongoDB's own data lived in a Docker volume rather than inside the container. A Kubernetes Volume object solves this same problem, at the cluster level.

## How to mitigate security concerns with containers

- Use maintained container images
- Use automatic vulnerability scanning on the container registry
- Use your own security scanning tool on container images
- Never run containers with root privileges
- Monitor and/or log container activity

## Maintained images

Official, actively maintained base images, like node:20 or mongo:8.0, both used in earlier Docker work, receive regular security patches and updates from a trusted source, rather than being built from scratch or left to grow outdated.

- Benefits: stronger security, more reliability, less ongoing maintenance burden
- Tradeoffs: less control over exactly what is included inside the image, and potential extra size if the maintained image contains things not actually needed

## Commands

```bash
kubectl get all
```
Shows everything currently running in the cluster. Can also be narrowed to kubectl get deployments or kubectl get pods to see just one specific object type.

```bash
kubectl create -f <filename>
```
Creates a new deployment, or other object, from a YAML file. -f means from this file.

```bash
kubectl apply -f <filename>
```
Updates an already-existing deployment to match the current contents of a file, this is what is actually used for changes to something already created, rather than create.

```bash
kubectl delete -f <filename>
```
Removes the deployment, or other object, defined in that file.

## Code-along, deploying and exposing nginx

**Folder structure used:** ~/k8s/local-nginx-deploy/

**Deployment file, nginx-deploy.yml:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: daraymonsta/nginx-257:dreamteam
        ports:
        - containerPort: 80
```

apiVersion: apps/v1 is the API version Deployments use. kind: Deployment is the object type. replicas: 3 tells Kubernetes to always keep exactly 3 identical pods running. selector: matchLabels: app: nginx is how the Deployment identifies which pods belong to it, by label, not by name. The template section is the blueprint each individual pod is stamped out from, its own labels: app: nginx is what actually ties each created pod back to the selector above. image and ports inside the container spec work the same as they would in a plain docker run command or Dockerfile.

Applied with:
```bash
kubectl apply -f nginx-deploy.yml
```

Confirmed with kubectl get pods, all three pods showed 1/1 Running once the image finished being pulled.

**Service file, nginx-svc.yml, needed separately to make the pods actually reachable:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
  namespace: default
spec:
  ports:
  - nodePort: 30001
    port: 80
    targetPort: 80
  selector:
    app: nginx
  type: NodePort
```

A Deployment alone only creates running pods, it does not make them reachable from outside the cluster. A Service is a separate object specifically for exposure and networking. selector: app: nginx connects this Service to the exact same pods created by the Deployment, using the same label-matching mechanism. type: NodePort exposes the service on a specific port on the node itself, the alternative, LoadBalancer, is for genuine external cloud-provider-backed access, and ClusterIP, the default, only allows access from inside the cluster. nodePort: 30001 must fall in the 30000-32768 range, and is the actual port reachable from outside. port: 80 is what the Service listens on internally, targetPort: 80 is the port on the pod itself that traffic actually gets forwarded to, matching nginx's own listening port.

Applied with:
```bash
kubectl apply -f nginx-svc.yml
```

Confirmed working with:
```bash
minikube ip
curl http://<minikube-ip>:30001
```
Returned the actual nginx homepage HTML, confirming deployment, replica management, and networking all working correctly together.

## Changing replica count while a cluster is live, three ways

Three different ways to change how many replicas are running, without deleting and recreating anything from scratch, taught in this session:

**1. Edit the live deployment directly:**
```bash
kubectl edit deploy nginx-deployment
```
Opens the deployment's current configuration in an editor. Saving the file applies the change immediately.

**2. Scale via a direct command:**
```bash
kubectl scale --current-replicas=3 --replicas=4 deployment/nginx-deployment
```
`--current-replicas` acts as a safety check, the command only proceeds if the deployment's actual current count matches what is specified, preventing an accidental scale based on stale information. `--replicas` sets the new target count.

**3. Edit the YAML file itself, then reapply:**
Change `replicas:` directly inside `nginx-deploy.yml`, then run:
```bash
kubectl apply -f nginx-deploy.yml
```
Kubernetes compares the file's declared state against the cluster's actual current state and adjusts accordingly, the same declarative principle used everywhere else in Kubernetes.

## Rolling updates

Changing the container image inside `nginx-deploy.yml` to a different image or tag, then reapplying the file, triggers a rolling update. Running `kubectl get all` immediately afterward during this kind of change shows two separate ReplicaSets existing at the same time briefly, the original one and a new one matching the updated image, with new pods coming up as `Running` or `Pending` while old pods show `Terminating`.

This is what a rolling update looks like in practice, Kubernetes does not delete every old pod and then start new ones, it brings new pods up gradually, only phasing out old ones once replacements are ready, meaning the application stays available throughout the entire update rather than having any period of total downtime. This is the direct, hands-on version of the rolling updates and rollbacks benefit listed earlier
