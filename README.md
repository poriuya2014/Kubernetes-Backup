# Kubernetes-Backup


## velero + MinIO + Velero UI Deployment with NodePort in Kubernetes

this guide covers the installation, configuration, backup, and restoration of Kubernetes resources using Velero and MinIO. Finally, we will set up Velero UI and expose it via NodePort for easy access.


# 1.Prerequisites
 Requirements:
A running Kubernetes cluster (version 1.16+)
Docker installed on your system
Basic understanding of Kubernetes concepts
MinIO as an S3-compatible storage backend
Installed CLI tools: velero and mc (MinIO Client)

# 2. Install Velero and MinIO Client

## 2.1 Install Velero CLI
```bash
wget https://github.com/vmware-tanzu/velero/releases/download/v1.14.0/velero-v1.14.0-linux-amd64.tar.gz
tar -xvf velero-v1.14.0-linux-amd64.tar.gz
sudo mv velero-v1.14.0-linux-amd64/velero /usr/local/bin/
velero version
```
## 2.2 Install MinIO Client (mc)

```bash
wget https://dl.min.io/client/mc/release/linux-amd64/mc
chmod +x mc
sudo mv mc /usr/local/bin/

```

# 3. Deploy MinIO in docker

```bash
mkdir -p ${HOME}/minio/data
```
Start the MinIO container
```bash
docker run -d \
   -p 9000:9000 \
   -p 9001:9001 \
   --user $(id -u):$(id -g) \
   --name minio1 \
   --restart unless-stopped \
   -e "MINIO_ROOT_USER=ROOTUSER" \
   -e "MINIO_ROOT_PASSWORD=CHANGEME123" \
   -v ${HOME}/minio/data:/data \
   quay.io/minio/minio server /data --console-address ":9001"

```
# 3.1Create a bucket for Velero backups:

Open your browser and navigate to the MinIO Web UI:
```bash
http://<NODE_IP>:9001
```

3.2. Create a New Bucket for Velero
Once logged into the MinIO Web UI:

1. Click on "Buckets" in the left sidebar.
2. Click "Create Bucket" (usually a + button on the top right).
3. Enter a Bucket Name (e.g., velero-backups).
4. Set the bucket policies (e.g., keep it private unless required otherwise).
5. Click "Create" to finalize the bucket creation.

 Now the velero-bucket is created and ready for use.

You can even do this by command:
# Using mc (MinIO client)

```bash
mc alias set local http://192.168.140.27:9000 puria 1q2w3e
mc mb local/velero
```



3.3.Obtain an Access Token for Authentication
Instead of using static credentials (minioadmin and minioadmin), you can generate a secure Access Token for Velero.

1. Navigate to "Access Keys" or "Identity Management" in MinIO Web UI.
2. Create a new Access Key by clicking "Create New Key".
3. Save the generated Access Key and Secret Key.
4. Ensure that the user has the appropriate permissions for Velero to store backups.

If needed, attach the Read and Write policy to the generated credentials.

# 4. Install Velero in Kubernetes

## 4.1 Create a credentials file for Velero
```bash
cat <<EOF > credentials-velero
[default]
aws_access_key_id=minioadmin
aws_secret_access_key=minioadmin
EOF
```

## 4.2 Install Velero with MinIO

```bash

velero install \
  --provider aws \
  --image velero/velero:v1.14.0 \
  --plugins velero/velero-plugin-for-aws:v1.10.0 \
  --bucket velero-backups \
  --secret-file ./credentials-velero \
  --use-node-agent \
  --use-volume-snapshots=false \
  --uploader-type kopia \
  --default-volumes-to-fs-backup \
  --namespace velero \
  --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://<minio_ip>::9000
```
 Check Velero Pods:

```bash
kubectl get pods -n velero

```


## 4.3 Create and Restore Backups

```bash
# Set the backup name based on current date and time
BACKUP_NAME="full-cluster-backup-$(date +%Y.%m.%d-%H.%M)"
# Create a Velero backup with the specified name and wait for completion
velero backup create $BACKUP_NAME --wait

# Describe the details of the created backup
velero backup describe $BACKUP_NAME
# View the logs of the backup creation process
velero backup logs $BACKUP_NAME

# List all Velero backups
velero get backup

```
Restore from the backup:
```bash
# Restore from a Velero backup using the specified backup name
velero restore create $BACKUP_NAME --from-backup $BACKUP_NAME --wait

# Describe the details of the restore operation
velero restore describe $BACKUP_NAME
# View the logs of the restore process
velero restore logs $BACKUP_NAME
```

# 5. Install Velero UI and Set Up NodePort
## 5.1 Install Velero UI using Helm
Deploying Velero UI chart
To install the velero-ui chart in the velero-ui namespace:

```bash
kubectl create namespace velero-ui
```
```bash
helm repo add otwld https://helm.otwld.com/
helm repo update
helm install velero-ui otwld/velero-ui --namespace velero-ui

```
Upgrading Velero UI chart:
Make adjustments to your values as needed, then run helm upgrade:
```bash
# -- This pulls the latest version of the velero-ui chart from the repo.
helm repo update
helm upgrade velero-ui olwld/velero-ui --namespace velero-ui --values values.yaml

```
## 5.2 Expose Velero UI using NodePort:

```bash

kubectl patch svc velero-ui -n velero-ui -p '{
  "spec": {
    "type": "NodePort",
    "ports": [{
      "name": "ui-port",
      "port": 9099,
      "protocol": "TCP",
      "targetPort": 3000,
      "nodePort": 30999
    }]
  }
}'

```
Check Velero UI Service:
```bash
kubectl get svc -n velero-ui
```
Velero UI will be accessible at:
```bash

http://<NODE_IP>:30999

```


```
ctr image pull docker.io/velero/velero:v1.14.0
ctr image pull docker.io/otwld/velero-ui:0.10.0
```




