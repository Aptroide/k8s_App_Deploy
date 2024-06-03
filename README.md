# Kubernetes Application Deployment

## Table of Contents

- [Objective](#objective)
- [Application Description](#application-description)
- [Containerization](#containerization)
- [Kubernetes Configuration](#kubernetes-configuration)
- [Scaling](#scaling)
- [Networking](#networking)
- [Monitoring](#monitoring)

## Objective

Apply concepts learned about pods, deployments, and services in Kubernetes to deploy a multi-tier application. 


## Application Description

To-Do list app:
 - **Fronted:** React for the user interface.
 - **Backend:** Node.js with Express.js for handling data processing. 
 - **Database:** MongoDB to store to-do tasks.

from: https://github.com/knaopel/docker-frontend-backend-db.git

## Containerization

Inside the `docker-frontend-backend-db` folder , the base application is divided into two folders:    

 - `frontend`: to containerize this module, we use `frontend/DockerFile`
    ```bash
    FROM node:14-alpine

    WORKDIR /app

    # add '/app/node_modules/.bin' to $PATH
    ENV PATH /app/node_modules/.bin:$PATH

    # install application dependencies
    COPY package*.json ./
    RUN npm install

    # copy app files
    COPY . .

    EXPOSE 3000
    CMD ["npm", "start"]
    ```

 - `backend`: to containerize this module, we use `backend/DockerFile`
    ```bash
    FROM node:10-alpine

    WORKDIR /usr/src/app

    # install application dependencies
    COPY package*.json ./
    RUN npm install

    # copy app files
    COPY . .

    EXPOSE 3000

    CMD ["node", "server.js"]
    ```
Once we have our Dockerfiles, we need to build them and then push them to Docker Hub so they can be downloaded inside each pod. To accomplish this, we use the following commands:
```bash
docker build . -t <docker hub username>/<image_name>:<tag>
docker push <docker hub username>/<image_name>:<tag>
```

To containerize the **database** we use the base image available on docker hub called `mongo`


## Kubernetes Configuration

Overview of the Kubernetes deployment and service configurations for the web application. The application is separated into three different deployments: frontend, backend, and MongoDB database.

#### MongoDB Deployment and Service

##### Deployment
- **Name:** db-k8s
- **Replicas:** 1
- **Container Image:** mongo
- **Resources:** 
  - Memory Limit: 128Mi
  - CPU Limit: 250m
- **Port:** 27017

##### Service
- **Name:** db-k8s
- **Type:** ClusterIP (default)
- **Port:** 27017

#### Backend Deployment and Service

##### Deployment
- **Name:** backend-k8s
- **Container Image:** aptroide/backend-k8s:V3
- **Resources:** 
  - Memory Limit: 128Mi
  - CPU Limit: 250m
- **Port:** 3000

##### Service
- **Name:** backend-k8s
- **Type:** NodePort
- **Port:** 3000
- **Target Port:** 3000

#### Frontend Deployment and Service

##### Deployment
- **Name:** frontend-k8s
- **Container Image:** aptroide/fronted-k8s:V3.1
- **Resources:** 
  - Memory Limit: 1024Mi
  - CPU Limit: 500m
- **Port:** 3000

##### Service
- **Name:** frontend-k8s
- **Type:** NodePort
- **Port:** 3000
- **Target Port:** 3000

#### Importance of Service Type

- **ClusterIP:** Used for internal communication within the Kubernetes cluster. This is used for **database service (MongoDB)** to ensure secure and efficient access within the cluster.

- **LoadBalancer:** Used on **Frontend and Backend services** Exposes the service on each node's IP at a static port. This is suitable because these services need to be accessible from outside the Kubernetes cluster, allowing external traffic to reach the application.

In the Frontend deployment, we allocate more resources compared to the backend deployment. This is necessary because React requires additional memory to compile its dependencies and render the graphical interface.

By separating the deployments and services, we achieve modularity, scalability, and ease of management, ensuring each component of the application can be independently managed and scaled as needed.

## Scaling

To Implement horizontal autoscaling (HPA) for the backend deployment based on CPU utilization, we first need to enable `metrics-server` addon of :
```bash
minikube addons enable metrics-server
```

Once we have this, we use `hpa_backend.yaml` to deploy the service:

```bash
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-k8s-pod-autoscaler
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend-k8s # name of the deployment where the HPA will be use
  minReplicas: 5
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 80 # max CPU utilization
```
Next, we apply this YAML file and check if the configuration is running correctly.

![start](/img/1.1.png)

We can see that it works by running

```bash
k get all
```
![start](/img/1.2.png)

to test this, we use the 

```bash
 while true; do wget -q -O- http://192.168.59.101:30303/api/todos; done
```
![start](/img/1.4.png)

This will increase CPU utilization. As depicted in the image below, the number of backend nodes increases because we set the minimum number of pods to 5. However, since they never reach the 80% CPU threshold specified in the YAML file, everything is functioning normally.

![start](/img/1.5.png)
## Networking

To test connectivity between pods, we first need to start minikube cluster using:
```bash
minikube start
```
![start](/img/1.png)

Setting alias to use kubectl:
```bash
alias k=kubectl
```

Next we apply all our configuration yaml files:
```bash
k apply -f backend_D_S.yaml -f mongo_D_S.yaml -f fronted_D_S.yaml
```
And check that all its working fine using:
```bash
k get pods
k get svc
k get deploy
```
![start](/img/2.png)

Using the names and IP addresses of the pods, we can check connectivity between them.

| NAME                          | READY   | STATUS   |   RESTARTS | AGE   | IP          | NODE     | NOMINATED NODE   | READINESS GATES   |
|:------------------------------|:--------|:---------|-----------:|:------|:------------|:---------|:-----------------|:------------------|
| backend-k8s-fc5756b7-bz98h    | 1/1     | Running  |          0 | 53m   | 10.244.0.51 | minikube | none           | none            |
| db-k8s-86f6b7c6-562wq         | 1/1     | Running  |          0 | 53m   | 10.244.0.52 | minikube | none           | none            |
| frontend-k8s-65cc48b7fc-lwkc6 | 1/1     | Running  |          0 | 53m   | 10.244.0.53 | minikube | none           | none            |

### Backend 
 - To fronted: 
    ```bash
    kubectl exec backend-k8s-fc5756b7-bz98h -- ping 10.244.0.53
    ```
 - To database: 
    ```bash
    kubectl exec backend-k8s-fc5756b7-bz98h -- ping 10.244.0.52
    ```
    ![start](/img/3.png)

### Fronted 
 - To backend: 
    ```bash
    kubectl exec frontend-k8s-65cc48b7fc-lwkc6 -- ping 10.244.0.51
    ```
 - To database: 
    ```bash
    kubectl exec frontend-k8s-65cc48b7fc-lwkc6 -- ping 10.244.0.52
    ```
    ![start](/img/4.png)

### Database 
 - To backend: 
    ```bash
    kubectl exec db-k8s-86f6b7c6-562wq -- ping 10.244.0.51
    ```
 - To fronted: 
    ```bash
    kubectl exec db-k8s-86f6b7c6-562wq -- ping 10.244.0.53
    ```
    ![start](/img/5.png)

We also can check that the app its running inside our minikube cluster:
![start](/img/app1.png)

API conexion response:
![start](/img/app2.png)

## Monitoring

To use prometheus for Monitoring, we first need to disable `metrics-server`

```bash
minikube addons disable metrics-server
```

Next we clone the repository
```bash
git clone https://github.com/prometheus-operator/kube-prometheus.git
```

and run `prometheus.sh` script to apply all the YAML files inside `manifests` 
```bash
#!/bin/bash
need to use kubectl create instead.
kubectl apply --server-side -f manifests/setup
kubectl wait \
	--for condition=Established \
	--all CustomResourceDefinition \
	--namespace=monitoring
kubectl apply -f manifests/
```

![start](/img/2.1.png)

This will create a new `namespace` called `monitoring`

![start](/img/2.2.png)

We will use `Grafana` pod, so we need to do a port-forward:

```bash
 k -n monitoring port-forward grafana-854bdcdf45-nmpnl 3000:3000
```
![start](/img/2.3.png)

Now we can log in to `Grafana` to monitor using default dashboards. (admin - admin are the inicial credentials)

![start](/img/2.4.png)
![start](/img/2.5.png)

