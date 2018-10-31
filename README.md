# k8s-cockroachdb-backup

A simple and highly configurable cron job for cockroachdb backup to a GCP bucket

## Motivation

The design of this cron job has been mainly taken from <https://github.com/kminehart/cockroachdb-backup> but with some minor changes. Because the docker image
of the kminehart/cockroachdb-backup:v2.0.0 sets the version of cockroach db, it means that in order to use the correct version you'd have to have multiple of those docker images. This repo moves the execution script and the installation script into the container command. That way things such as cockroach db version can increment as the actual database increments.

## Setup & Configuration

| Environment Variable | Default Value                                                 | Notes                |
| -------------------- | ------------------------------------------------------------- | -------------------- |
| COCKROACH_URL        | postgresql://root@db-cockroachdb-public:26257?sslmode=disable |                      |
| COCKROACH_DATABASE   | database                                                      | name of the database |
| CLOUD_PROVIDER       | gcp                                                           |                      |
| GCP_BUCKET_NAME      | my-gcp-bucket                                                 |                      |
| GCP_SA_USER          | $COCKROACHDB_BACKUP_EMAIl                                     | see below            |

### Create a ServiceAccount

This service account will have access to the Storage services in your project

    gcloud iam service-accounts create cockroach-backup-sa --display-name "CockroachDB Backup Account"

### Create the key for this service account

This key will be given to the Kubernetes job to authenticate to your gcloud account

    export COCKROACHDB_BACKUP_EMAIl=`gcloud iam service-accounts list --filter="displayName='CockroachDB Backup Account'" --format="value(email.scope())" | head -n 1`
    gcloud iam service-accounts keys create key.json --iam-account $COCKROACHDB_BACKUP_EMAIl

### Grant the proper roles (roles/storage.objectCreator)

Now grant the backup service account the ability to add items to Storage

    export PROJECT=$(gcloud info --format='value(config.project)')

    gcloud projects add-iam-policy-binding $PROJECT \
      --member serviceAccount:$COCKROACHDB_BACKUP_EMAIl \
      --role roles/storage.objectCreator

### Now turn it into a Kubernetes secret

    kubectl create secret generic cockroach-backup-sa --from-file=./key.json

### Add the cronjob

    kubectl create -f cronjob.yml

it will run at midnight UTC every day
