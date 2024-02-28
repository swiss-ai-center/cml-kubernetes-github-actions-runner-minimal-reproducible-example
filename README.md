# CML and Kubernetes using a GitHub Actions Runner minimal reproducible example

This repository contains a minimal reproducible example to illustrate a bug
occuring with CML and Kubernetes using a GitHub Actions Runner as described in
[CML Issue #1415](https://github.com/iterative/cml/issues/1415).

## Prerequisites

In order to run this example, you will need the following:

- A Google Cloud Platform account (GCP)
- A valid billing account on GCP
- [gcloud CLI](https://cloud.google.com/sdk/docs/install-sdk) installed and
  configured

## Tools versions

At the time of writing, the following versions were used:

```sh
# Display gcloud version
$ gcloud --version
Google Cloud SDK 466.0.0
alpha 2024.02.26
beta 2024.02.26
bq 2.0.101
bundled-python3-unix 3.11.7
core 2024.02.26
gcloud-crc32c 1.0.0
gke-gcloud-auth-plugin 0.5.8
gsutil 5.27
kubectl 1.26.13

# Display Kubernetes cluster version
$ gcloud container clusters list
NAME                LOCATION        MASTER_VERSION      MASTER_IP      MACHINE_TYPE   NODE_VERSION        NUM_NODES  STATUS
kubernetes-cluster  europe-west1-b  1.27.8-gke.1067004  34.34.161.209  e2-standard-2  1.27.8-gke.1067004  1          RUNNING
```

## Reproducing the bug

### Install and configure gcloud CLI

Install gcloud usind the official documentation:
[_Install the Google Cloud CLI_ - cloud.google.com](https://cloud.google.com/sdk/docs/install-sdk).

Initialize and configure gcloud:

```sh title="Execute the following command(s) in a terminal"
# Initialize and login to Google Cloud
$ gcloud init
```

### Create a new project on GCP

Create a new project on GCP:

```sh
# Export the project ID
$ export GCP_PROJECT_ID=cml-k8s-github-runner-mre

# Create a new project
$ gcloud projects create $GCP_PROJECT_ID

# Select your Google Cloud project
$ gcloud config set project $GCP_PROJECT_ID
```

### Link the billing account to the project

Link the billing account to the project:

```sh
# List the billing accounts
$ gcloud billing accounts list

# Export the billing account ID
$ export GCP_BILLING_ACCOUNT_ID=YOUR_BILLING_ACCOUNT_ID

# Link the billing account to the project
$ gcloud billing projects link $GCP_PROJECT_ID \
    --billing-account $GCP_BILLING_ACCOUNT_ID
```

### Create a Kubernetes cluster

Enable the Google Kubernetes Engine API to create Kubernetes clusters on Google:

```sh
# Enable the Google Kubernetes Engine API
$ gcloud services enable container.googleapis.com
```

Create a new Kubernetes cluster:

```sh
# Export the cluster name and zone
$ export GCP_K8S_CLUSTER_NAME=kubernetes-cluster
$ export GCP_K8S_CLUSTER_ZONE=europe-west1-b

# Create the Kubernetes cluster
$ gcloud container clusters create \
    --machine-type=e2-standard-2 \
    --num-nodes=1 \
    --zone=$GCP_K8S_CLUSTER_ZONE \
    $GCP_K8S_CLUSTER_NAME
```

### Create a service account

Create a new service account to authenticate to Google Cloud:

```sh
# Create the Google Service Account
$ gcloud iam service-accounts create gcp-service-account \
    --display-name="Google Cloud Platform Service Account"

# Set the Kubernetes Cluster permissions for the Google Service Account
$ gcloud projects add-iam-policy-binding $GCP_PROJECT_ID \
    --member="serviceAccount:gcp-service-account@$GCP_PROJECT_ID.iam.gserviceaccount.com" \
    --role="roles/container.developer"

# Export the Google Service Account Key
$ gcloud iam service-accounts keys create ~/.config/gcloud/gcp-service-account.json \
    --iam-account=gcp-service-account@$GCP_PROJECT_ID.iam.gserviceaccount.com

# Display the Google Service Account Key
$ cat ~/.config/gcloud/gcp-service-account.json
```

### Add GitHub Actions Secrets

Add the following secrets to your GitHub repository:

- `GCP_SERVICE_ACCOUNT`: The service account key to authenticate to Google Cloud
- `GCP_K8S_CLUSTER_NAME`: The name of the Kubernetes cluster
- `GCP_K8S_CLUSTER_ZONE`: The zone of the Kubernetes cluster
- `PERSONAL_ACCESS_TOKEN`: A personal access token to authenticate to GitHub

### Create the GitHub Actions Workflow

Add the following content to the `.github/workflows/workflow.yaml` file:

```yaml
name: Workflow

on:
  push:
  workflow_dispatch:

jobs:
  setup-runner:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Login to Google Cloud
        uses: google-github-actions/auth@v2
        with:
          credentials_json: ${{ secrets.GCP_SERVICE_ACCOUNT }}

      - name: Get Google Cloud's Kubernetes credentials
        uses: google-github-actions/get-gke-credentials@v2
        with:
          cluster_name: ${{ secrets.GCP_K8S_CLUSTER_NAME }}
          location: ${{ secrets.GCP_K8S_CLUSTER_ZONE }}

      - name: Setup CML
        uses: iterative/setup-cml@v2
        with:
          version: '0.20.0'

      - name: Initialize runner on Kubernetes
        env:
          REPO_TOKEN: ${{ secrets.PERSONAL_ACCESS_TOKEN }}
          TF_LOG_PROVIDER: DEBUG
        run: |
          # Export the Kubernetes configuration
          export KUBERNETES_CONFIGURATION=$(cat $KUBECONFIG)

          # Initialize the runner on Kubernetes
          cml runner \
            --labels="cml-runner" \
            --cloud="kubernetes" \
            --cloud-type="s"

  use-runner:
    needs: setup-runner
    runs-on: [self-hosted, cml-runner]
    steps:
      - name: Execute command in Kubernetes runner
        run: echo "Hello, World!"
```

### Run the workflow

Commit and push the GitHub Workflow file.

The workflow will be triggered on each push or you can run the workflow
manually.

### Current behavior

The workflow fails with the following error:

```sh

```

### Expected behavior

### Elements that were tested

### Possible workarounds

## Conclusion

## References

- [CML Issue #1415](https://github.com/iterative/cml/issues/1415)
