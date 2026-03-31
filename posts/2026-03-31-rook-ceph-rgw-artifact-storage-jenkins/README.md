# How to Use Ceph RGW as Artifact Storage for Jenkins

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Jenkins, CI/CD, Artifact, S3, Storage

Description: Configure Jenkins to store and retrieve build artifacts on Ceph RGW S3-compatible storage, replacing local disk storage with a scalable, durable object store.

---

## Overview

Jenkins builds produce artifacts like JARs, binaries, test reports, and Docker tarballs. By default these are stored on the Jenkins controller disk, which limits scalability. Ceph RGW provides an S3-compatible endpoint that Jenkins can use via the S3 Artifact Manager plugin for durable, scalable artifact storage.

## Prepare Ceph RGW Bucket

Create a dedicated bucket and user:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin user create \
  --uid=jenkins \
  --display-name="Jenkins CI" \
  --access-key=jenkinsakey \
  --secret-key=jenkinsskey

aws s3 mb s3://jenkins-artifacts \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80
```

## Install the S3 Artifact Manager Plugin

In Jenkins: Manage Jenkins > Plugins > Available > search "S3 Artifact Manager" > Install.

## Configure Jenkins S3 Provider

In Jenkins: Manage Jenkins > System > Artifact Management for Builds:

```text
S3 provider:
  S3 bucket: jenkins-artifacts
  Endpoint override: http://rook-ceph-rgw-my-store.rook-ceph:80
  Bucket region: us-east-1
  Path style: enabled
  Access key: jenkinsakey
  Secret key: jenkinsskey
```

Or configure via Groovy init script:

```groovy
import io.jenkins.plugins.artifact_manager_jclouds.s3.S3BlobStoreConfig

def config = S3BlobStoreConfig.get()
config.setContainer("jenkins-artifacts")
config.setPrefix("builds/")
config.setEndpointUrl("http://rook-ceph-rgw-my-store.rook-ceph:80")
config.setPathStyleAccess(true)
config.setUseHttp(true)
config.save()
```

## Use Artifacts in a Jenkinsfile

```groovy
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }
        stage('Archive') {
            steps {
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            }
        }
    }
    post {
        always {
            junit 'target/surefire-reports/*.xml'
        }
    }
}
```

## Verify Artifacts in Ceph

```bash
aws s3 ls s3://jenkins-artifacts/builds/ \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
  --recursive | head -20
```

## Download an Artifact Directly

```bash
aws s3 cp s3://jenkins-artifacts/builds/job-name/1/target/app-1.0.jar \
  /tmp/app.jar \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80
```

## Summary

Using Ceph RGW as Jenkins artifact storage removes disk capacity constraints from the Jenkins controller and centralizes artifact management in a durable object store. The S3 Artifact Manager plugin handles all upload and retrieval transparently, and Rook manages the Ceph backend lifecycle.
