---
date: "2026-09-15"
title: "K8s cronjob for MongoDB backups"
tags: ["MongoDB", "Kubernetes"]
---

A simple approach to periodically backup a MongoDB instance running in Kubernetes is by using a `CronJob` executing the `mongodump` command. Here is an example implementation that uploads the dumps to an S3 bucket:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: mongodb-backup
spec:
  schedule: "0 3 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: Never
          containers:
            - name: mongodb-backup
              image: mongo:8.0 # ships mongodump
              command:
                - /bin/bash
                - -c
                - |
                  set -euo pipefail

                  TIMESTAMP=$(date +%Y-%m-%dT%H-%M-%S)
                  ARCHIVE="/tmp/backup-${TIMESTAMP}.archive.gz"

                  echo "Running mongodump..."
                  mongodump --uri="${MONGODB_URI}" --archive="${ARCHIVE}" --gzip

                  echo "Installing curl and unzip..."
                  apt-get update -qq
                  DEBIAN_FRONTEND=noninteractive apt-get install -y -qq curl unzip

                  echo "Installing AWS CLI v2 (official installer)..."
                  curl -sSL "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o /tmp/awscliv2.zip
                  unzip -q /tmp/awscliv2.zip -d /tmp
                  /tmp/aws/install

                  echo "Uploading to Linode..."
                  aws s3 cp "${ARCHIVE}" \
                    "s3://<your-s3-bucket>/backups/${TIMESTAMP}.archive.gz" \
                    --endpoint-url https://<your-s3-endpoint>

                  echo "Done."

              envFrom:
                - secretRef:
                    name: mongodb-backups
```

The necesary environment variables are `MONGODB_URI`, `AWS_ACCESS_KEY_ID` and `ACCESS_SECRET_KEY`, stored in a dedicated secret.

## Dedicated container

To simplify the Cronjob, I made a dedicated container which comes with the AWS CLI preinstalled and runs the backup script at runtime.
The container image is available on [Docker Hub](https://hub.docker.com/repository/docker/moreillon/mongodump-s3/general) while the source code is on [GitHub](https://github.com/maximemoreillon/mongodump-s3).

With this container, the CronJob can be simplified as follows:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: mongodb-backup
spec:
  schedule: "0 3 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: Never
          containers:
            - name: mongodb-backup
              image: moreillon/mongodump-s3:8.0
              env:
                - name: S3_ENDPOINT_URL
                  value: <your endpoint>
                - name: S3_BUCKET
                  value: <your bucket>
                - name: S3_PREFIX
                  value: backups
              envFrom:
                - secretRef:
                    name: mongodb-backups
```
