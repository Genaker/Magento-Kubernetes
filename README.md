# Magento-Kubernetes
Magento 2 Kubernetes Configuration 

This repo showcases the full power of Kubernetes clusters and shows how can we deploy the world's most popular eCommerce framework Magento 2 on top of world's most popular container orchestration platform Kubernetes. We provide a full roadmap for hosting Magento on a Kubernetes Cluster. Each component runs in a separate container or group of containers.

Magento 2 represents a typical multi-tier app and each component will have its own container(s). The Magento 2 containers will be the frontend tier and the MySQL, Redis, elastic search container will be the database/backend tier for Magento.

## This scenario provides instructions for the following tasks:

Create local persistent volumes to define persistent disks.
Create and deploy the Magento frontend with one or more pods.
Create and deploy the MySQL database (either in a container or using AWS RDS Aurora Mysql as a backend).

## Create Local Persistent Volumes
To save your data beyond the lifecycle of a Kubernetes pod, you will want to create persistent volumes for your MySQL and MAgento applications to attach to.

Create the local persistent volumes manually by running
```
kubectl create -f local-volumes.yaml
```

## Create Services and deployments for Magento and MySQL

Install persistent volume on your cluster's local storage. Then, create the secret and services for MySQL and Magento.
```
kubectl create -f mysql-deployment.yaml
kubectl create -f magento-deployment.yaml
```
When all your pods are running, run the following commands to check your pod names.

kubectl get pods
This should return a list of pods from the kubernetes cluster.
```
NAME                               READY     STATUS    RESTARTS   AGE
magento-43534534543-54mmd         1/1       Running   0          45m
magento-mysql-2569670970-bd07b    1/1       Running   0          53m
```
