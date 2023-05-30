# Velero demo

This project aims to explore and test the Velero tool for backing up and restoring Kubernetes clusters.  
Velero is a powerful open-source tool that facilitates the backup, migration and restoration of Kubernetes resources.

## Prerequisites

- k3d: https://k3d.io/#installation
- Docker: https://docs.docker.com/get-docker/
- Docker Compose: https://docs.docker.com/compose/install/
- Velero CLI: https://velero.io/docs/v1.6/basic-install/

## Example 1 : Backup and Restore

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

velero backup create nginx-backup --selector app=nginx

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
