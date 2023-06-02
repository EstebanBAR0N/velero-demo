# Velero demo

This project aims to explore and test the Velero tool for backing up and restoring Kubernetes clusters.  
Velero is a powerful open-source tool that facilitates the backup, migration and restoration of Kubernetes resources.

## Prerequisites

- k3d: https://k3d.io/#installation
- Docker: https://docs.docker.com/get-docker/
- Docker Compose: https://docs.docker.com/compose/install/
- Velero CLI: https://velero.io/docs/v1.6/basic-install/

## Example 1 : Backup and Restore

In this example, we'll back up and restore part of our cluster using Velero.  
First, we'll create a backup from a selector. Next, we'll simulate a disaster (deletion of our namespace). And finally, we'll restore it.

### Setup k3d cluster

```bash

# Create a k3d cluster

k3d cluster create velero-demo

```

### Setup minio

```bash

# setup minio

cd minio && docker-compose up -d && cd -

```

Create a bucket for velero :

Go to http://localhost:9001, login with creds in the docker compose file and create a bucket named "velero"

Create credentials for velero :

Go to http://localhost:9001 and create a new access key, copy the credentials and paste them in velero/credentials-velero file

### Setup Velero

```bash

# get your ip address

ip a

# Install Velero

velero install \
    --provider aws \
    --plugins velero/velero-plugin-for-aws:v1.1.0 \
    --bucket velero \
    --secret-file velero/credentials-velero \
    --use-volume-snapshots=false \
    --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://<your-ip-address>:9000 # replace <your-ip-address> with your ip address

# check logs

kubectl logs deployment/velero -n velero

```

### Deploy the app

```bash

# Create a namespace and deploy the app for the demo

kubectl apply -f ./app/nginx-demo-example.yaml

# Check the app

kubectl get all -n nginx-example

```

### Backup

```bash

# Create a backup for nginx app

velero backup create nginx-backup

# Verify the backup

velero backup describe nginx-backup # it should be completed

# in case of failure, check the logs

velero backup logs nginx-backup

```

### Restore

```bash

# Simulate a disaster

kubectl delete namespace nginx-example

# Restore the backup

velero restore create --from-backup nginx-backup

# Verify the restore

velero restore get

velero restore describe nginx-backup-xxxx # it should be completed

# You can now check your nginx app

kubectl get all -n nginx-example

```

## Example 2 : Selective Backup and Restore

With the selective backup and restore feature, we have the ability to choose and backup a specific part of the cluster.

The only part that undergoes a change is the backup process.

### Backup

```bash

# Create a backup for nginx app

velero backup create nginx-backup-2 --selector app=nginx

# Verify the backup

velero backup describe nginx-backup-2 # it should be completed

# in case of failure, check the logs

velero backup logs nginx-backup-2

```

> For more information on the command

```bash

velero backup create -h

```

## Example 3 : Cluster migration

Migration from one cluster to another with Velero is simple. Simply connect Velero to your second cluster and clone the first cluster from your backup created earlier.

### Setup the 2nd k3d cluster

```bash

k3d cluster create velero-demo-2

```

### Setup Velero on the 2nd cluster

```bash

# get your ip address

ip a

# Install Velero

velero install \
    --provider aws \
    --plugins velero/velero-plugin-for-aws:v1.1.0 \
    --bucket velero \
    --secret-file velero/credentials-velero \
    --use-volume-snapshots=false \
    --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://<your-ip-address>:9000 # replace <your-ip-address> with your ip address

# check logs

kubectl logs deployment/velero -n velero

```

### Clone 1st cluster from Velero backup

```bash

velero restore create --from-backup nginx-backup

# You can now check your nginx app

kubectl get all -n nginx-example

```

# Test with a full project

In order to test with a full project we will use the project from the following repository:
https://github.com/Roxxas96/gitops-demo

## Clone the project

first you can clone the project:

```bash
git@github.com:Roxxas96/gitops-demo.git
```

## Setup

For the setup you can do it the same way than in previous example.
So you just need to create a cluster and install velero on it.
You will also need to setup minio and create a bucket for velero.

## Deploy the app

```bash

cd gitops-demo/infrastructure/helm/postgresql
helm install postgres ./chart -f ../../../../velero-demo-gitops/postgres-values.yaml
cd -
```

```bash

cd gitops-demo/word-service/containers/helm
helm install word-api ./chart -f ../../../../velero-demo-gitops/back-values.yaml
cd -
```

```bash
cd gitops-demo/frontend/containers/helm
helm install word-api ./chart -f ../../../../velero-demo-gitops/front-values.yaml
cd -
```

## Backup

```bash
velero backup create gitops --default-volumes-to-fs-backup
velero backup logs test
helm uninstall postgres
velero restore create --from-backup gitops
```
