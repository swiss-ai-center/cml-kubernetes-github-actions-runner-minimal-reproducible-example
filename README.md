# CML and Kubernetes using a GitHub Actions Runner minimal reproducible example

This repository contains a minimal reproducible example to illustrate a bug
occuring with CML and Kubernetes using a GitHub Actions Runner as described in
[CML Issue #1415](https://github.com/iterative/cml/issues/1415).

## Prerequisites

In order to run this example, you will need the following:

- A Google Cloud Platform account (GCP)
- A valid billing account on GCP

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
$ gcloud container clusters describe kubernetes-cluster --zone=europe-west1-b | grep currentNodeVersion
currentNodeVersion: 1.27.8-gke.1067004
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

### Create a GitHub Personal Access Token

In your GitHub account, create a new personal access token (classic) with the
following permissions:

- `repo`

### Add GitHub Actions Secrets

Add the following secrets to your GitHub repository:

- `GCP_SERVICE_ACCOUNT`: The service account key to authenticate to Google Cloud
- `GCP_K8S_CLUSTER_NAME`: The name of the Kubernetes cluster
- `GCP_K8S_CLUSTER_ZONE`: The zone of the Kubernetes cluster
- `PERSONAL_ACCESS_TOKEN`: The personal access token to authenticate to GitHub

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
          version: "0.20.0"

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

### Delete the Kubernetes cluster

Delete the Kubernetes cluster to avoid incurring costs with the following
command:

```bash
# Delete the Kubernetes cluster
gcloud container clusters delete \
    --zone $GCP_K8S_CLUSTER_ZONE $GCP_K8S_CLUSTER_NAME
```

## Bug analysis

We were able to use CML succesfully with the same setup a year ago. Then,
suddenly, the workflows stopped working. We are unable to find the cause of the
issue.

### Current behavior

The workflow fails with the followind behavior:

1. A Kubernetes pod is created
2. The pod registers to the GitHub Actions Runner
3. The workflow is stuck at the `setup-runner` job
4. After a few minutes, the Kubernetes pod is deleted
5. After 20 minutes, the workflow is cancelled

<details>
<summary>Expand to see the full log of the GitHub Workflow execution</summary>

```text
2024-02-28T12:20:10.7445745Z Requested labels: ubuntu-latest
2024-02-28T12:20:10.7446125Z Job defined at: swiss-ai-center/cml-kubernetes-github-actions-runner-minimal-reproducible-example/.github/workflows/workflow.yaml@refs/heads/main
2024-02-28T12:20:10.7446277Z Waiting for a runner to pick up this job...
2024-02-28T12:20:11.3912511Z Job is waiting for a hosted runner to come online.
2024-02-28T12:20:14.9071502Z Job is about to start running on the hosted runner: GitHub Actions 12 (hosted)
2024-02-28T12:20:17.2371551Z Current runner version: '2.313.0'
2024-02-28T12:20:17.2395346Z ##[group]Operating System
2024-02-28T12:20:17.2396287Z Ubuntu
2024-02-28T12:20:17.2396896Z 22.04.4
2024-02-28T12:20:17.2397383Z LTS
2024-02-28T12:20:17.2397921Z ##[endgroup]
2024-02-28T12:20:17.2398526Z ##[group]Runner Image
2024-02-28T12:20:17.2399185Z Image: ubuntu-22.04
2024-02-28T12:20:17.2399748Z Version: 20240225.1.0
2024-02-28T12:20:17.2401057Z Included Software: https://github.com/actions/runner-images/blob/ubuntu22/20240225.1/images/ubuntu/Ubuntu2204-Readme.md
2024-02-28T12:20:17.2402724Z Image Release: https://github.com/actions/runner-images/releases/tag/ubuntu22%2F20240225.1
2024-02-28T12:20:17.2404464Z ##[endgroup]
2024-02-28T12:20:17.2405180Z ##[group]Runner Image Provisioner
2024-02-28T12:20:17.2406047Z 2.0.341.1
2024-02-28T12:20:17.2406552Z ##[endgroup]
2024-02-28T12:20:17.2409253Z ##[group]GITHUB_TOKEN Permissions
2024-02-28T12:20:17.2411191Z Actions: write
2024-02-28T12:20:17.2411786Z Checks: write
2024-02-28T12:20:17.2412607Z Contents: write
2024-02-28T12:20:17.2413363Z Deployments: write
2024-02-28T12:20:17.2414024Z Discussions: write
2024-02-28T12:20:17.2414553Z Issues: write
2024-02-28T12:20:17.2415201Z Metadata: read
2024-02-28T12:20:17.2415771Z Packages: write
2024-02-28T12:20:17.2416351Z Pages: write
2024-02-28T12:20:17.2416878Z PullRequests: write
2024-02-28T12:20:17.2417557Z RepositoryProjects: write
2024-02-28T12:20:17.2418205Z SecurityEvents: write
2024-02-28T12:20:17.2418819Z Statuses: write
2024-02-28T12:20:17.2419338Z ##[endgroup]
2024-02-28T12:20:17.2421682Z Secret source: Actions
2024-02-28T12:20:17.2422422Z Prepare workflow directory
2024-02-28T12:20:17.3048563Z Prepare all required actions
2024-02-28T12:20:17.3209534Z Getting action download info
2024-02-28T12:20:17.5633061Z Download action repository 'actions/checkout@v4' (SHA:b4ffde65f46336ab88eb53be808477a3936bae11)
2024-02-28T12:20:17.6729013Z Download action repository 'google-github-actions/auth@v2' (SHA:55bd3a7c6e2ae7cf1877fd1ccb9d54c0503c457c)
2024-02-28T12:20:18.1661229Z Download action repository 'google-github-actions/get-gke-credentials@v2' (SHA:c02be8662df01db62234e9b9cff0765d1c1827ae)
2024-02-28T12:20:18.6767073Z Download action repository 'iterative/setup-cml@v2' (SHA:f12ea7685438c1fc438d50f77519b3860cf5fa98)
2024-02-28T12:20:19.2792589Z Complete job name: setup-runner
2024-02-28T12:20:19.3722672Z ##[group]Run actions/checkout@v4
2024-02-28T12:20:19.3723337Z with:
2024-02-28T12:20:19.3724264Z   repository: swiss-ai-center/cml-kubernetes-github-actions-runner-minimal-reproducible-example
2024-02-28T12:20:19.3725420Z   token: ***
2024-02-28T12:20:19.3725991Z   ssh-strict: true
2024-02-28T12:20:19.3726503Z   persist-credentials: true
2024-02-28T12:20:19.3727145Z   clean: true
2024-02-28T12:20:19.3727648Z   sparse-checkout-cone-mode: true
2024-02-28T12:20:19.3728269Z   fetch-depth: 1
2024-02-28T12:20:19.3728800Z   fetch-tags: false
2024-02-28T12:20:19.3729362Z   show-progress: true
2024-02-28T12:20:19.3729852Z   lfs: false
2024-02-28T12:20:19.3730404Z   submodules: false
2024-02-28T12:20:19.3730976Z   set-safe-directory: true
2024-02-28T12:20:19.3731501Z ##[endgroup]
2024-02-28T12:20:19.5605186Z Syncing repository: swiss-ai-center/cml-kubernetes-github-actions-runner-minimal-reproducible-example
2024-02-28T12:20:19.5608660Z ##[group]Getting Git version info
2024-02-28T12:20:19.5611490Z Working directory is '/home/runner/work/cml-kubernetes-github-actions-runner-minimal-reproducible-example/cml-kubernetes-github-actions-runner-minimal-reproducible-example'
2024-02-28T12:20:19.5614325Z [command]/usr/bin/git version
2024-02-28T12:20:19.5615305Z git version 2.43.2
2024-02-28T12:20:19.5618188Z ##[endgroup]
2024-02-28T12:20:19.5633078Z Temporarily overriding HOME='/home/runner/work/_temp/0c978e97-7aaa-4b5a-b7d1-371bf3772505' before making global git config changes
2024-02-28T12:20:19.5635201Z Adding repository directory to the temporary git global config as a safe directory
2024-02-28T12:20:19.5638473Z [command]/usr/bin/git config --global --add safe.directory /home/runner/work/cml-kubernetes-github-actions-runner-minimal-reproducible-example/cml-kubernetes-github-actions-runner-minimal-reproducible-example
2024-02-28T12:20:19.5644195Z Deleting the contents of '/home/runner/work/cml-kubernetes-github-actions-runner-minimal-reproducible-example/cml-kubernetes-github-actions-runner-minimal-reproducible-example'
2024-02-28T12:20:19.5647181Z ##[group]Initializing the repository
2024-02-28T12:20:19.5649708Z [command]/usr/bin/git init /home/runner/work/cml-kubernetes-github-actions-runner-minimal-reproducible-example/cml-kubernetes-github-actions-runner-minimal-reproducible-example
2024-02-28T12:20:19.5700237Z hint: Using 'master' as the name for the initial branch. This default branch name
2024-02-28T12:20:19.5702096Z hint: is subject to change. To configure the initial branch name to use in all
2024-02-28T12:20:19.5703619Z hint: of your new repositories, which will suppress this warning, call:
2024-02-28T12:20:19.5704875Z hint:
2024-02-28T12:20:19.5705829Z hint: 	git config --global init.defaultBranch <name>
2024-02-28T12:20:19.5707842Z hint:
2024-02-28T12:20:19.5708931Z hint: Names commonly chosen instead of 'master' are 'main', 'trunk' and
2024-02-28T12:20:19.5710626Z hint: 'development'. The just-created branch can be renamed via this command:
2024-02-28T12:20:19.5711978Z hint:
2024-02-28T12:20:19.5712897Z hint: 	git branch -m <name>
2024-02-28T12:20:19.5714637Z Initialized empty Git repository in /home/runner/work/cml-kubernetes-github-actions-runner-minimal-reproducible-example/cml-kubernetes-github-actions-runner-minimal-reproducible-example/.git/
2024-02-28T12:20:19.5718188Z [command]/usr/bin/git remote add origin https://github.com/swiss-ai-center/cml-kubernetes-github-actions-runner-minimal-reproducible-example
2024-02-28T12:20:19.5750567Z ##[endgroup]
2024-02-28T12:20:19.5752173Z ##[group]Disabling automatic garbage collection
2024-02-28T12:20:19.5753564Z [command]/usr/bin/git config --local gc.auto 0
2024-02-28T12:20:19.5781393Z ##[endgroup]
2024-02-28T12:20:19.5782288Z ##[group]Setting up auth
2024-02-28T12:20:19.5786157Z [command]/usr/bin/git config --local --name-only --get-regexp core\.sshCommand
2024-02-28T12:20:19.5814807Z [command]/usr/bin/git submodule foreach --recursive sh -c "git config --local --name-only --get-regexp 'core\.sshCommand' && git config --local --unset-all 'core.sshCommand' || :"
2024-02-28T12:20:19.6104670Z [command]/usr/bin/git config --local --name-only --get-regexp http\.https\:\/\/github\.com\/\.extraheader
2024-02-28T12:20:19.6134504Z [command]/usr/bin/git submodule foreach --recursive sh -c "git config --local --name-only --get-regexp 'http\.https\:\/\/github\.com\/\.extraheader' && git config --local --unset-all 'http.https://github.com/.extraheader' || :"
2024-02-28T12:20:19.6373521Z [command]/usr/bin/git config --local http.https://github.com/.extraheader AUTHORIZATION: basic ***
2024-02-28T12:20:19.6408820Z ##[endgroup]
2024-02-28T12:20:19.6410709Z ##[group]Fetching the repository
2024-02-28T12:20:19.6418816Z [command]/usr/bin/git -c protocol.version=2 fetch --no-tags --prune --no-recurse-submodules --depth=1 origin +6336d82f3836de0a3bada090a11c06dc66cc029e:refs/remotes/origin/main
2024-02-28T12:20:20.0099199Z From https://github.com/swiss-ai-center/cml-kubernetes-github-actions-runner-minimal-reproducible-example
2024-02-28T12:20:20.0101441Z  * [new ref]         6336d82f3836de0a3bada090a11c06dc66cc029e -> origin/main
2024-02-28T12:20:20.0123958Z ##[endgroup]
2024-02-28T12:20:20.0124867Z ##[group]Determining the checkout info
2024-02-28T12:20:20.0126182Z ##[endgroup]
2024-02-28T12:20:20.0127213Z ##[group]Checking out the ref
2024-02-28T12:20:20.0130734Z [command]/usr/bin/git checkout --progress --force -B main refs/remotes/origin/main
2024-02-28T12:20:20.0170881Z Switched to a new branch 'main'
2024-02-28T12:20:20.0172172Z branch 'main' set up to track 'origin/main'.
2024-02-28T12:20:20.0177931Z ##[endgroup]
2024-02-28T12:20:20.0210916Z [command]/usr/bin/git log -1 --format='%H'
2024-02-28T12:20:20.0234001Z '6336d82f3836de0a3bada090a11c06dc66cc029e'
2024-02-28T12:20:20.0606744Z ##[group]Run google-github-actions/auth@v2
2024-02-28T12:20:20.0607452Z with:
2024-02-28T12:20:20.0619576Z   credentials_json: ***
2024-02-28T12:20:20.0620175Z   create_credentials_file: true
2024-02-28T12:20:20.0620890Z   export_environment_variables: true
2024-02-28T12:20:20.0621472Z   universe: googleapis.com
2024-02-28T12:20:20.0622088Z   cleanup_credentials: true
2024-02-28T12:20:20.0622612Z   access_token_lifetime: 3600s
2024-02-28T12:20:20.0623434Z   access_token_scopes: https://www.googleapis.com/auth/cloud-platform
2024-02-28T12:20:20.0624243Z   id_token_include_email: false
2024-02-28T12:20:20.0624764Z ##[endgroup]
2024-02-28T12:20:20.1515049Z Created credentials file at "/home/runner/work/cml-kubernetes-github-actions-runner-minimal-reproducible-example/cml-kubernetes-github-actions-runner-minimal-reproducible-example/gha-creds-1112903afc7915c2.json"
2024-02-28T12:20:20.1693852Z ##[group]Run google-github-actions/get-gke-credentials@v2
2024-02-28T12:20:20.1694740Z with:
2024-02-28T12:20:20.1695264Z   cluster_name: ***
2024-02-28T12:20:20.1695916Z   location: ***
2024-02-28T12:20:20.1696397Z   use_auth_provider: false
2024-02-28T12:20:20.1697071Z   use_internal_ip: false
2024-02-28T12:20:20.1697756Z   use_connect_gateway: false
2024-02-28T12:20:20.1698279Z env:
2024-02-28T12:20:20.1699873Z   CLOUDSDK_AUTH_CREDENTIAL_FILE_OVERRIDE: /home/runner/work/cml-kubernetes-github-actions-runner-minimal-reproducible-example/cml-kubernetes-github-actions-runner-minimal-reproducible-example/gha-creds-1112903afc7915c2.json
2024-02-28T12:20:20.1702390Z   GOOGLE_APPLICATION_CREDENTIALS: /home/runner/work/cml-kubernetes-github-actions-runner-minimal-reproducible-example/cml-kubernetes-github-actions-runner-minimal-reproducible-example/gha-creds-1112903afc7915c2.json
2024-02-28T12:20:20.1705060Z   GOOGLE_GHA_CREDS_PATH: /home/runner/work/cml-kubernetes-github-actions-runner-minimal-reproducible-example/cml-kubernetes-github-actions-runner-minimal-reproducible-example/gha-creds-1112903afc7915c2.json
2024-02-28T12:20:20.1706699Z   CLOUDSDK_CORE_PROJECT: cml-k8s-github-runner-mre
2024-02-28T12:20:20.1707378Z   CLOUDSDK_PROJECT: cml-k8s-github-runner-mre
2024-02-28T12:20:20.1708132Z   GCLOUD_PROJECT: cml-k8s-github-runner-mre
2024-02-28T12:20:20.1708792Z   GCP_PROJECT: cml-k8s-github-runner-mre
2024-02-28T12:20:20.1709549Z   GOOGLE_CLOUD_PROJECT: cml-k8s-github-runner-mre
2024-02-28T12:20:20.1710174Z ##[endgroup]
2024-02-28T12:20:21.5251127Z Successfully created and exported "KUBECONFIG" at: /home/runner/work/cml-kubernetes-github-actions-runner-minimal-reproducible-example/cml-kubernetes-github-actions-runner-minimal-reproducible-example/gha-kubeconfig-0ab51d4e7909dbae
2024-02-28T12:20:21.5379553Z ##[group]Run iterative/setup-cml@v2
2024-02-28T12:20:21.5380312Z with:
2024-02-28T12:20:21.5380754Z   version: 0.20.0
2024-02-28T12:20:21.5381299Z   vega: true
2024-02-28T12:20:21.5381787Z env:
2024-02-28T12:20:21.5383314Z   CLOUDSDK_AUTH_CREDENTIAL_FILE_OVERRIDE: /home/runner/work/cml-kubernetes-github-actions-runner-minimal-reproducible-example/cml-kubernetes-github-actions-runner-minimal-reproducible-example/gha-creds-1112903afc7915c2.json
2024-02-28T12:20:21.5385944Z   GOOGLE_APPLICATION_CREDENTIALS: /home/runner/work/cml-kubernetes-github-actions-runner-minimal-reproducible-example/cml-kubernetes-github-actions-runner-minimal-reproducible-example/gha-creds-1112903afc7915c2.json
2024-02-28T12:20:21.5388438Z   GOOGLE_GHA_CREDS_PATH: /home/runner/work/cml-kubernetes-github-actions-runner-minimal-reproducible-example/cml-kubernetes-github-actions-runner-minimal-reproducible-example/gha-creds-1112903afc7915c2.json
2024-02-28T12:20:21.5390149Z   CLOUDSDK_CORE_PROJECT: cml-k8s-github-runner-mre
2024-02-28T12:20:21.5390887Z   CLOUDSDK_PROJECT: cml-k8s-github-runner-mre
2024-02-28T12:20:21.5391841Z   GCLOUD_PROJECT: cml-k8s-github-runner-mre
2024-02-28T12:20:21.5392573Z   GCP_PROJECT: cml-k8s-github-runner-mre
2024-02-28T12:20:21.5393351Z   GOOGLE_CLOUD_PROJECT: cml-k8s-github-runner-mre
2024-02-28T12:20:21.5395095Z   KUBECONFIG: /home/runner/work/cml-kubernetes-github-actions-runner-minimal-reproducible-example/cml-kubernetes-github-actions-runner-minimal-reproducible-example/gha-kubeconfig-0ab51d4e7909dbae
2024-02-28T12:20:21.5397581Z   KUBE_CONFIG_PATH: /home/runner/work/cml-kubernetes-github-actions-runner-minimal-reproducible-example/cml-kubernetes-github-actions-runner-minimal-reproducible-example/gha-kubeconfig-0ab51d4e7909dbae
2024-02-28T12:20:21.5399142Z ##[endgroup]
2024-02-28T12:20:23.1600323Z [command]/usr/local/bin/npm install --global canvas@2 vega@5 vega-cli@5 vega-lite@5
2024-02-28T12:20:31.6133896Z
2024-02-28T12:20:31.6135048Z added 167 packages in 7s
2024-02-28T12:20:31.6136539Z
2024-02-28T12:20:31.6137066Z 8 packages are looking for funding
2024-02-28T12:20:31.6138469Z   run `npm fund` for details
2024-02-28T12:20:31.6476347Z ##[group]Run # Export the Kubernetes configuration
2024-02-28T12:20:31.6477334Z [36;1m# Export the Kubernetes configuration[0m
2024-02-28T12:20:31.6478138Z [36;1mexport KUBERNETES_CONFIGURATION=$(cat $KUBECONFIG)[0m
2024-02-28T12:20:31.6478936Z [36;1m[0m
2024-02-28T12:20:31.6479447Z [36;1m# Initialize the runner on Kubernetes[0m
2024-02-28T12:20:31.6480135Z [36;1mcml runner \[0m
2024-02-28T12:20:31.6480698Z [36;1m  --labels="cml-runner" \[0m
2024-02-28T12:20:31.6481412Z [36;1m  --cloud="kubernetes" \[0m
2024-02-28T12:20:31.6481970Z [36;1m  --cloud-type="s"[0m
2024-02-28T12:20:31.6526487Z shell: /usr/bin/bash -e ***0***
2024-02-28T12:20:31.6527174Z env:
2024-02-28T12:20:31.6528668Z   CLOUDSDK_AUTH_CREDENTIAL_FILE_OVERRIDE: /home/runner/work/cml-kubernetes-github-actions-runner-minimal-reproducible-example/cml-kubernetes-github-actions-runner-minimal-reproducible-example/gha-creds-1112903afc7915c2.json
2024-02-28T12:20:31.6531423Z   GOOGLE_APPLICATION_CREDENTIALS: /home/runner/work/cml-kubernetes-github-actions-runner-minimal-reproducible-example/cml-kubernetes-github-actions-runner-minimal-reproducible-example/gha-creds-1112903afc7915c2.json
2024-02-28T12:20:31.6533893Z   GOOGLE_GHA_CREDS_PATH: /home/runner/work/cml-kubernetes-github-actions-runner-minimal-reproducible-example/cml-kubernetes-github-actions-runner-minimal-reproducible-example/gha-creds-1112903afc7915c2.json
2024-02-28T12:20:31.6535542Z   CLOUDSDK_CORE_PROJECT: cml-k8s-github-runner-mre
2024-02-28T12:20:31.6536285Z   CLOUDSDK_PROJECT: cml-k8s-github-runner-mre
2024-02-28T12:20:31.6536990Z   GCLOUD_PROJECT: cml-k8s-github-runner-mre
2024-02-28T12:20:31.6537676Z   GCP_PROJECT: cml-k8s-github-runner-mre
2024-02-28T12:20:31.6538340Z   GOOGLE_CLOUD_PROJECT: cml-k8s-github-runner-mre
2024-02-28T12:20:31.6540023Z   KUBECONFIG: /home/runner/work/cml-kubernetes-github-actions-runner-minimal-reproducible-example/cml-kubernetes-github-actions-runner-minimal-reproducible-example/gha-kubeconfig-0ab51d4e7909dbae
2024-02-28T12:20:31.6542379Z   KUBE_CONFIG_PATH: /home/runner/work/cml-kubernetes-github-actions-runner-minimal-reproducible-example/cml-kubernetes-github-actions-runner-minimal-reproducible-example/gha-kubeconfig-0ab51d4e7909dbae
2024-02-28T12:20:31.6544068Z   REPO_TOKEN: ***
2024-02-28T12:20:31.6544541Z   TF_LOG_PROVIDER: DEBUG
2024-02-28T12:20:31.6545158Z ##[endgroup]
2024-02-28T12:20:33.1113260Z ***"level":"info","message":"POST /repos/swiss-ai-center/cml-kubernetes-github-actions-runner-minimal-reproducible-example/actions/runners/registration-token - 201 in 254ms"***
2024-02-28T12:20:33.3768420Z ***"level":"info","message":"GET /repos/swiss-ai-center/cml-kubernetes-github-actions-runner-minimal-reproducible-example/actions/runners?per_page=100 - 200 in 253ms"***
2024-02-28T12:20:33.3796176Z ***"level":"warn","message":"Github Actions timeout has been updated from 72h to 35 days. Update your workflow accordingly to be able to restart it automatically."***
2024-02-28T12:20:33.3799140Z ***"level":"warn","message":"ignoring RUNNER_NAME environment variable, use CML_RUNNER_NAME or --name instead"***
2024-02-28T12:20:33.3800750Z ***"level":"info","message":"Preparing workdir /home/runner/.cml/ec7vbos7hr..."***
2024-02-28T12:20:33.3805569Z ***"level":"info","message":"Deploying cloud runner plan..."***
2024-02-28T12:20:33.3807941Z ***"level":"info","message":"Terraform apply..."***
2024-02-28T12:20:41.5847495Z ***"level":"info","message":"Terraform 1.7.4"***
2024-02-28T12:20:42.1805251Z ***"level":"info","message":"iterative_cml_runner.runner: Plan to create"***
2024-02-28T12:20:42.1808731Z ***"level":"info","message":"Plan: 1 to add, 0 to change, 0 to destroy."***
2024-02-28T12:20:42.2458264Z ***"level":"info","message":"iterative_cml_runner.runner: Creating..."***
2024-02-28T12:20:52.2520277Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [10s elapsed]"***
2024-02-28T12:21:02.2529666Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [20s elapsed]"***
2024-02-28T12:21:12.2541235Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [30s elapsed]"***
2024-02-28T12:21:22.2546690Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [40s elapsed]"***
2024-02-28T12:21:32.2549497Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [50s elapsed]"***
2024-02-28T12:21:42.2557157Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [1m0s elapsed]"***
2024-02-28T12:21:52.2567655Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [1m10s elapsed]"***
2024-02-28T12:22:02.2569568Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [1m20s elapsed]"***
2024-02-28T12:22:12.2578601Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [1m30s elapsed]"***
2024-02-28T12:22:22.2584795Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [1m40s elapsed]"***
2024-02-28T12:22:32.2588693Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [1m50s elapsed]"***
2024-02-28T12:22:42.2592423Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [2m0s elapsed]"***
2024-02-28T12:22:52.2596895Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [2m10s elapsed]"***
2024-02-28T12:23:02.2604089Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [2m20s elapsed]"***
2024-02-28T12:23:12.2608181Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [2m30s elapsed]"***
2024-02-28T12:23:22.2611729Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [2m40s elapsed]"***
2024-02-28T12:23:32.2612777Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [2m50s elapsed]"***
2024-02-28T12:23:42.2619315Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [3m0s elapsed]"***
2024-02-28T12:23:52.2619069Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [3m10s elapsed]"***
2024-02-28T12:24:02.2625923Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [3m20s elapsed]"***
2024-02-28T12:24:12.2632619Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [3m30s elapsed]"***
2024-02-28T12:24:22.2640598Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [3m40s elapsed]"***
2024-02-28T12:24:32.2646501Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [3m50s elapsed]"***
2024-02-28T12:24:42.2652929Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [4m0s elapsed]"***
2024-02-28T12:24:52.2654196Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [4m10s elapsed]"***
2024-02-28T12:25:02.2661001Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [4m20s elapsed]"***
2024-02-28T12:25:12.2665723Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [4m30s elapsed]"***
2024-02-28T12:25:22.2669805Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [4m40s elapsed]"***
2024-02-28T12:25:32.2673072Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [4m50s elapsed]"***
2024-02-28T12:25:42.2676382Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [5m0s elapsed]"***
2024-02-28T12:25:52.2682410Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [5m10s elapsed]"***
2024-02-28T12:26:02.2693202Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [5m20s elapsed]"***
2024-02-28T12:26:12.2699690Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [5m30s elapsed]"***
2024-02-28T12:26:22.2705308Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [5m40s elapsed]"***
2024-02-28T12:26:32.2707952Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [5m50s elapsed]"***
2024-02-28T12:26:42.2708602Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [6m0s elapsed]"***
2024-02-28T12:26:52.2715219Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [6m10s elapsed]"***
2024-02-28T12:27:02.2719059Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [6m20s elapsed]"***
2024-02-28T12:27:12.2729859Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [6m30s elapsed]"***
2024-02-28T12:27:22.2741414Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [6m40s elapsed]"***
2024-02-28T12:27:32.2746845Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [6m50s elapsed]"***
2024-02-28T12:27:42.2749318Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [7m0s elapsed]"***
2024-02-28T12:27:52.2751245Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [7m10s elapsed]"***
2024-02-28T12:28:02.2755217Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [7m20s elapsed]"***
2024-02-28T12:28:12.2758898Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [7m30s elapsed]"***
2024-02-28T12:28:22.2764764Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [7m40s elapsed]"***
2024-02-28T12:28:32.2767765Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [7m50s elapsed]"***
2024-02-28T12:28:42.2771643Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [8m0s elapsed]"***
2024-02-28T12:28:52.2775729Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [8m10s elapsed]"***
2024-02-28T12:29:02.2780609Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [8m20s elapsed]"***
2024-02-28T12:29:12.2786061Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [8m30s elapsed]"***
2024-02-28T12:29:22.2793590Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [8m40s elapsed]"***
2024-02-28T12:29:32.2795731Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [8m50s elapsed]"***
2024-02-28T12:29:42.2806196Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [9m0s elapsed]"***
2024-02-28T12:29:52.2814685Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [9m10s elapsed]"***
2024-02-28T12:30:02.2825903Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [9m20s elapsed]"***
2024-02-28T12:30:12.2836867Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [9m30s elapsed]"***
2024-02-28T12:30:22.2847466Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [9m40s elapsed]"***
2024-02-28T12:30:32.2849279Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [9m50s elapsed]"***
2024-02-28T12:30:42.2854768Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [10m0s elapsed]"***
2024-02-28T12:30:52.2864295Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [10m10s elapsed]"***
2024-02-28T12:31:02.2868833Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [10m20s elapsed]"***
2024-02-28T12:31:12.2877704Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [10m30s elapsed]"***
2024-02-28T12:31:22.2882595Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [10m40s elapsed]"***
2024-02-28T12:31:32.2884225Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [10m50s elapsed]"***
2024-02-28T12:31:42.2888436Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [11m0s elapsed]"***
2024-02-28T12:31:52.2893570Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [11m10s elapsed]"***
2024-02-28T12:32:02.2896485Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [11m20s elapsed]"***
2024-02-28T12:32:12.2901226Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [11m30s elapsed]"***
2024-02-28T12:32:22.2913273Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [11m40s elapsed]"***
2024-02-28T12:32:32.2921300Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [11m50s elapsed]"***
2024-02-28T12:32:42.2927888Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [12m0s elapsed]"***
2024-02-28T12:32:52.2933094Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [12m10s elapsed]"***
2024-02-28T12:33:02.2943070Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [12m20s elapsed]"***
2024-02-28T12:33:12.2950398Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [12m30s elapsed]"***
2024-02-28T12:33:22.2955502Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [12m40s elapsed]"***
2024-02-28T12:33:32.2959025Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [12m50s elapsed]"***
2024-02-28T12:33:42.2965626Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [13m0s elapsed]"***
2024-02-28T12:33:52.2970798Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [13m10s elapsed]"***
2024-02-28T12:34:02.2974922Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [13m20s elapsed]"***
2024-02-28T12:34:12.2975166Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [13m30s elapsed]"***
2024-02-28T12:34:22.2980903Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [13m40s elapsed]"***
2024-02-28T12:34:32.2984491Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [13m50s elapsed]"***
2024-02-28T12:34:42.2989643Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [14m0s elapsed]"***
2024-02-28T12:34:52.2999665Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [14m10s elapsed]"***
2024-02-28T12:35:02.3011567Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [14m20s elapsed]"***
2024-02-28T12:35:12.3020405Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [14m30s elapsed]"***
2024-02-28T12:35:22.3022675Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [14m40s elapsed]"***
2024-02-28T12:35:32.3032589Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [14m50s elapsed]"***
2024-02-28T12:35:42.3034209Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [15m0s elapsed]"***
2024-02-28T12:35:52.3038656Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [15m10s elapsed]"***
2024-02-28T12:36:02.3040053Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [15m20s elapsed]"***
2024-02-28T12:36:12.3045610Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [15m30s elapsed]"***
2024-02-28T12:36:22.3056145Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [15m40s elapsed]"***
2024-02-28T12:36:32.3062951Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [15m50s elapsed]"***
2024-02-28T12:36:42.3068441Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [16m0s elapsed]"***
2024-02-28T12:36:52.3076573Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [16m10s elapsed]"***
2024-02-28T12:37:02.3081549Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [16m20s elapsed]"***
2024-02-28T12:37:12.3090205Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [16m30s elapsed]"***
2024-02-28T12:37:22.3094896Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [16m40s elapsed]"***
2024-02-28T12:37:32.3101113Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [16m50s elapsed]"***
2024-02-28T12:37:42.3103045Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [17m0s elapsed]"***
2024-02-28T12:37:52.3110126Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [17m10s elapsed]"***
2024-02-28T12:38:02.3112122Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [17m20s elapsed]"***
2024-02-28T12:38:12.3115384Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [17m30s elapsed]"***
2024-02-28T12:38:22.3122971Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [17m40s elapsed]"***
2024-02-28T12:38:32.3132837Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [17m50s elapsed]"***
2024-02-28T12:38:42.3137702Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [18m0s elapsed]"***
2024-02-28T12:38:52.3146449Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [18m10s elapsed]"***
2024-02-28T12:39:02.3149708Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [18m20s elapsed]"***
2024-02-28T12:39:12.3156088Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [18m30s elapsed]"***
2024-02-28T12:39:22.3165536Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [18m40s elapsed]"***
2024-02-28T12:39:32.3173250Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [18m50s elapsed]"***
2024-02-28T12:39:42.3179495Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [19m0s elapsed]"***
2024-02-28T12:39:52.3183428Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [19m10s elapsed]"***
2024-02-28T12:40:02.3188705Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [19m20s elapsed]"***
2024-02-28T12:40:12.3191101Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [19m30s elapsed]"***
2024-02-28T12:40:22.3198307Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [19m40s elapsed]"***
2024-02-28T12:40:32.3201434Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [19m50s elapsed]"***
2024-02-28T12:40:42.3206136Z ***"level":"info","message":"iterative_cml_runner.runner: Still creating... [20m0s elapsed]"***
2024-02-28T12:40:45.8956014Z ***"level":"info","message":"iterative_cml_runner.runner: Creation errored after 20m4s"***
2024-02-28T12:40:45.8983656Z ***"level":"error","message":"terraform error: Error: Failed disposing the machine: context deadline exceeded"***
2024-02-28T12:40:45.8986858Z ***"level":"error","message":"terraform error: Error: Failed creating the machine: Still creating... no pods in default matching controller-uid="***
2024-02-28T12:40:45.9043223Z 2024-02-28T12:20:41.909Z [INFO]  provider: configuring client automatic mTLS
2024-02-28T12:40:45.9047159Z 2024-02-28T12:20:41.937Z [DEBUG] provider: starting plugin: path=.terraform/providers/registry.terraform.io/iterative/iterative/0.11.20/linux_amd64/terraform-provider-iterative args=[".terraform/providers/registry.terraform.io/iterative/iterative/0.11.20/linux_amd64/terraform-provider-iterative"]
2024-02-28T12:40:45.9052369Z 2024-02-28T12:20:41.937Z [DEBUG] provider: plugin started: path=.terraform/providers/registry.terraform.io/iterative/iterative/0.11.20/linux_amd64/terraform-provider-iterative pid=1887
2024-02-28T12:40:45.9056158Z 2024-02-28T12:20:41.937Z [DEBUG] provider: waiting for RPC address: path=.terraform/providers/registry.terraform.io/iterative/iterative/0.11.20/linux_amd64/terraform-provider-iterative
2024-02-28T12:40:45.9059438Z 2024-02-28T12:20:41.960Z [INFO]  provider.terraform-provider-iterative: configuring server automatic mTLS: timestamp=2024-02-28T12:20:41.959Z
2024-02-28T12:40:45.9061705Z 2024-02-28T12:20:41.979Z [DEBUG] provider: using plugin: version=5
2024-02-28T12:40:45.9064772Z 2024-02-28T12:20:41.979Z [DEBUG] provider.terraform-provider-iterative: plugin address: address=/tmp/plugin1506352180 network=unix timestamp=2024-02-28T12:20:41.979Z
2024-02-28T12:40:45.9068115Z 2024-02-28T12:20:42.004Z [DEBUG] provider.stdio: received EOF, stopping recv loop: err="rpc error: code = Unavailable desc = error reading from server: EOF"
2024-02-28T12:40:45.9071873Z 2024-02-28T12:20:42.006Z [DEBUG] provider: plugin process exited: path=.terraform/providers/registry.terraform.io/iterative/iterative/0.11.20/linux_amd64/terraform-provider-iterative pid=1887
2024-02-28T12:40:45.9074490Z 2024-02-28T12:20:42.006Z [DEBUG] provider: plugin exited
2024-02-28T12:40:45.9076105Z 2024-02-28T12:20:42.007Z [INFO]  provider: configuring client automatic mTLS
2024-02-28T12:40:45.9080207Z 2024-02-28T12:20:42.013Z [DEBUG] provider: starting plugin: path=.terraform/providers/registry.terraform.io/iterative/iterative/0.11.20/linux_amd64/terraform-provider-iterative args=[".terraform/providers/registry.terraform.io/iterative/iterative/0.11.20/linux_amd64/terraform-provider-iterative"]
2024-02-28T12:40:45.9085074Z 2024-02-28T12:20:42.013Z [DEBUG] provider: plugin started: path=.terraform/providers/registry.terraform.io/iterative/iterative/0.11.20/linux_amd64/terraform-provider-iterative pid=1896
2024-02-28T12:40:45.9088818Z 2024-02-28T12:20:42.013Z [DEBUG] provider: waiting for RPC address: path=.terraform/providers/registry.terraform.io/iterative/iterative/0.11.20/linux_amd64/terraform-provider-iterative
2024-02-28T12:40:45.9090805Z 2024-02-28T12:20:42.034Z [INFO]  provider.terraform-provider-iterative: configuring server automatic mTLS: timestamp=2024-02-28T12:20:42.034Z
2024-02-28T12:40:45.9092680Z 2024-02-28T12:20:42.053Z [DEBUG] provider.terraform-provider-iterative: plugin address: network=unix address=/tmp/plugin1301342332 timestamp=2024-02-28T12:20:42.053Z
2024-02-28T12:40:45.9094135Z 2024-02-28T12:20:42.053Z [DEBUG] provider: using plugin: version=5
2024-02-28T12:40:45.9095477Z 2024-02-28T12:20:42.067Z [DEBUG] provider.stdio: received EOF, stopping recv loop: err="rpc error: code = Unavailable desc = error reading from server: EOF"
2024-02-28T12:40:45.9097499Z 2024-02-28T12:20:42.069Z [DEBUG] provider: plugin process exited: path=.terraform/providers/registry.terraform.io/iterative/iterative/0.11.20/linux_amd64/terraform-provider-iterative pid=1896
2024-02-28T12:40:45.9098902Z 2024-02-28T12:20:42.069Z [DEBUG] provider: plugin exited
2024-02-28T12:40:45.9099909Z 2024-02-28T12:20:42.110Z [INFO]  provider: configuring client automatic mTLS
2024-02-28T12:40:45.9102055Z 2024-02-28T12:20:42.116Z [DEBUG] provider: starting plugin: path=.terraform/providers/registry.terraform.io/iterative/iterative/0.11.20/linux_amd64/terraform-provider-iterative args=[".terraform/providers/registry.terraform.io/iterative/iterative/0.11.20/linux_amd64/terraform-provider-iterative"]
2024-02-28T12:40:45.9104679Z 2024-02-28T12:20:42.116Z [DEBUG] provider: plugin started: path=.terraform/providers/registry.terraform.io/iterative/iterative/0.11.20/linux_amd64/terraform-provider-iterative pid=1905
2024-02-28T12:40:45.9106690Z 2024-02-28T12:20:42.116Z [DEBUG] provider: waiting for RPC address: path=.terraform/providers/registry.terraform.io/iterative/iterative/0.11.20/linux_amd64/terraform-provider-iterative
2024-02-28T12:40:45.9108846Z 2024-02-28T12:20:42.139Z [INFO]  provider.terraform-provider-iterative: configuring server automatic mTLS: timestamp=2024-02-28T12:20:42.138Z
2024-02-28T12:40:45.9110134Z 2024-02-28T12:20:42.158Z [DEBUG] provider: using plugin: version=5
2024-02-28T12:40:45.9111527Z 2024-02-28T12:20:42.158Z [DEBUG] provider.terraform-provider-iterative: plugin address: address=/tmp/plugin2960935075 network=unix timestamp=2024-02-28T12:20:42.158Z
2024-02-28T12:40:45.9113827Z 2024-02-28T12:20:42.175Z [DEBUG] provider.stdio: received EOF, stopping recv loop: err="rpc error: code = Unavailable desc = error reading from server: EOF"
2024-02-28T12:40:45.9115985Z 2024-02-28T12:20:42.177Z [DEBUG] provider: plugin process exited: path=.terraform/providers/registry.terraform.io/iterative/iterative/0.11.20/linux_amd64/terraform-provider-iterative pid=1905
2024-02-28T12:40:45.9117538Z 2024-02-28T12:20:42.177Z [DEBUG] provider: plugin exited
2024-02-28T12:40:45.9118423Z 2024-02-28T12:20:42.180Z [INFO]  provider: configuring client automatic mTLS
2024-02-28T12:40:45.9120688Z 2024-02-28T12:20:42.186Z [DEBUG] provider: starting plugin: path=.terraform/providers/registry.terraform.io/iterative/iterative/0.11.20/linux_amd64/terraform-provider-iterative args=[".terraform/providers/registry.terraform.io/iterative/iterative/0.11.20/linux_amd64/terraform-provider-iterative"]
2024-02-28T12:40:45.9123211Z 2024-02-28T12:20:42.186Z [DEBUG] provider: plugin started: path=.terraform/providers/registry.terraform.io/iterative/iterative/0.11.20/linux_amd64/terraform-provider-iterative pid=1914
2024-02-28T12:40:45.9125319Z 2024-02-28T12:20:42.186Z [DEBUG] provider: waiting for RPC address: path=.terraform/providers/registry.terraform.io/iterative/iterative/0.11.20/linux_amd64/terraform-provider-iterative
2024-02-28T12:40:45.9127166Z 2024-02-28T12:20:42.208Z [INFO]  provider.terraform-provider-iterative: configuring server automatic mTLS: timestamp=2024-02-28T12:20:42.208Z
2024-02-28T12:40:45.9128442Z 2024-02-28T12:20:42.227Z [DEBUG] provider: using plugin: version=5
2024-02-28T12:40:45.9129901Z 2024-02-28T12:20:42.227Z [DEBUG] provider.terraform-provider-iterative: plugin address: address=/tmp/plugin2974799293 network=unix timestamp=2024-02-28T12:20:42.227Z
2024-02-28T12:40:45.9234690Z 2024-02-28T12:20:45.188Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:20:45 [INFO] Creating new Job: v1.Job***TypeMeta:v1.TypeMeta***Kind:"", APIVersion:""***, ObjectMeta:v1.ObjectMeta***Name:"cml-ec7vbos7hr-h338b14h-65p5e7ja", GenerateName:"", Namespace:"default", SelfLink:"", UID:"", ResourceVersion:"", Generation:0, CreationTimestamp:time.Date(1, time.January, 1, 0, 0, 0, 0, time.UTC), DeletionTimestamp:<nil>, DeletionGracePeriodSeconds:(*int64)(nil), Labels:map[string]string***, Annotations:map[string]string***, OwnerReferences:[]v1.OwnerReference(nil), Finalizers:[]string(nil), ClusterName:"", ManagedFields:[]v1.ManagedFieldsEntry(nil)***, Spec:v1.JobSpec***Parallelism:(*int32)(0xc000825db8), Completions:(*int32)(0xc000825db4), ActiveDeadlineSeconds:(*int64)(nil), BackoffLimit:(*int32)(0xc000825db0), Selector:(*v1.LabelSelector)(nil), ManualSelector:(*bool)(nil), Template:v1.PodTemplateSpec***ObjectMeta:v1.ObjectMeta***Name:"", GenerateName:"", Namespace:"", SelfLink:"", UID:"", ResourceVersion:"", Generation:0, CreationTimestamp:time.Date(1, time.January, 1, 0, 0, 0, 0, time.UTC), DeletionTimestamp:<nil>, DeletionGracePeriodSeconds:(*int64)(nil), Labels:map[string]string(nil), Annotations:map[string]string(nil), OwnerReferences:[]v1.OwnerReference(nil), Finalizers:[]string(nil), ClusterName:"", ManagedFields:[]v1.ManagedFieldsEntry(nil)***, Spec:v1.PodSpec***Volumes:[]v1.Volume(nil), InitContainers:[]v1.Container(nil), Containers:[]v1.Container***v1.Container***Name:"cml-ec7vbos7hr-h338b14h-65p5e7ja", Image:"dvcorg/cml:0-dvc2-base1-gpu", Command:[]string***"bash", "-c", "#!/bin/sh\nsudo systemctl is-enabled cml.service && return 0\n\nsudo curl --location https://github.com/iterative/terraform-provider-iterative/releases/latest/download/leo_linux_amd64 --output /usr/bin/leo\nsudo chmod a+x /usr/bin/leo\nexport KUBERNETES_CONFIGURATION='apiVersion: v1\nclusters:\n  - cluster:\n      certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUVMRENDQXBTZ0F3SUJBZ0lRUlVVbCtYWmVpVVRUT0drQ3VsbHhoakFOQmdrcWhraUc5dzBCQVFzRkFEQXYKTVMwd0t3WURWUVFERXlSbU9XRTJZbUUwWmkwME1qUmtMVFJtTUdNdE9XRXdOUzA0TXpaaVpERmhaR0UwWlRFdwpJQmNOTWpRd01qSTRNRGswT0RFM1doZ1BNakExTkRBeU1qQXhNRFE0TVRkYU1DOHhMVEFyQmdOVkJBTVRKR1k1CllUWmlZVFJtTFRReU5HUXROR1l3WXkwNVlUQTFMVGd6Tm1Ka01XRmtZVFJsTVRDQ0FhSXdEUVlKS29aSWh2Y04KQVFFQkJRQURnZ0dQQURDQ0FZb0NnZ0dCQU53Vy93ZW1SQ1hUTGZDWEQzeS93bVZLMUJSZGY3YWNiUTlieEY0SwpNRitqdDU5UFAzZlA4QkRpQy9pR1JsTDdFcmhNYkRuTjhTWVN5cG9YYUk2c0pTZ1RlZ3pEcjMzUDlxQm9hWndSClVwUy9lZC9WSGtwS0ZqVWNMQTR6TFM5MW9iMVdZMTEvcHY0SnRhNHhjQ2F4ejg4SUxPV3V1M0N1dTljTmh2c3gKMy9tVVRKRm1iN0VPMGZzdHVCWTVadXU1S3cyUGh1K2RXdm1KVHdJZFNtNlJrc3l0OEpleXFOd016c3I5b3RucwpTdXNvaXFzRlJvMTE5NnlwLzR6R0R4WWhGNWR4NDhJZFE0SjBVYWgyV0FxdkhDWEZMY2tJMy9yODl0YWdFYVlPCkgvOGI4a3oydHhJRlhJL1hodzRjOHdOSkdLaVVJSjUvTGYxRndCejdJZkVXa21vcURNM2NMVERrQ1N4VVFJL2oKc2FEb2FqNUtxdW5hWjBIWWV4WEMyVURFYVV3MDNMVSsyZUxLMUVUOVdJZzJpcEVmamk3cHZTejZHczdPc3o3Tgp6cTk0T2NJOEJLS3MyUjNrd1g1QWxBM1J5T0FmdTk2cGFqWTl2MGlORW94ZHhsd1FmZ2pzd3Z3MmRDQ04vVjc3CkllUDBZOHFqcDBpMmtKdHBVOHA1ZTBkUDB3SURBUUFCbzBJd1FEQU9CZ05WSFE4QkFmOEVCQU1DQWdRd0R3WUQKVlIwVEFRSC9CQVV3QXdFQi96QWRCZ05WSFE0RUZnUVVCSU5HRU8zUzlFeUFoaDhXamxxYS9XaTc5STR3RFFZSgpLb1pJaHZjTkFRRUxCUUFEZ2dHQkFDRW5ES0wxT3FIY3o2WG5zTEttUU9VSWNGQmNRZTZFMGkwaVFzUllhY0tmCjJPOVl2Ykp6enYxejBIdG9IdjF6ZHpFd3NoWmxub3pwM1JjR3h3Zk1aTGNTYkF6czJmekoxNGxkUmhDa1NQYVYKNjZKejBxdzRtZzdFVVVtRFQ2YmtCRDZ3amtWdUt3YmZKVGd5MndMK0JpTDk2V3VuUXdVT2MzOCsrVjM4RjIvNgo2YVZQbnZNd2JtbFU1MnB5RzRoYjMxVS8wN3pYeWkzTWNRN1o2anRQcWRqd2EvTzFXWFkrZTBJZklQU1l4dlZxClE2dzFiVGUwNlRobjZMK2EwbWVLbzNJVWd6VWtLeDV4Z0F6TmswcHlveDlGUmhPZkp4ZE9yVXNhNkZydWJyMGgKNWVWL0tpKzhkWDk5c2V3Q2JEek9PU1h0V0JVN2lnRzRQejNZUFRxZlNKZElFMVZtSHJPS3Y4a2RpS1dGU2cvaApza2FJU0ZtOXFvcGVZL29hbkxONmpjWi9Xdk9CVUh0VE84cW1tUVZDcGFpZHAzQ01Qam1Ndk1VWFpMYXlBYXZiCi9wTHlNV3gvTWhWK3kxYWUrSHpUNUIvNFBlSjhFdjFQOHlCYWRVb1o0MDRSMk9EK3VaQmM5N0dpQ3BkdUhTRjUKaHpDVityc250aW9VTDZrMEkzdTZmQT09Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K\n      server: https://34.34.161.209\n    name: ***\ncontexts:\n  - context:\n      cluster: ***\n      user: ***\n    name: gke_cml-k8s-github-runner-mre_***_***\nkind: Config\ncurrent-context: gke_cml-k8s-github-runner-mre_***_***\nusers:\n  - name: ***\n    user:\n      token: ya29.c.c0AY_VpZiim-sSd5FeCxKmwopUGEp4A-s_DgUwuLJ_-w8M7tU92ltSMlja1GmF-6UzbMZlubMl1lfIvp1VSskeZJChN6ARg4Mdai4VGgPdsVCghlr7o2M3PLG_m3cyxuv9BOmkTiXPbbRXIH5enF0sTLuttViYAJPltZO67_iCjhOgXInMbxBd39ttnKxvkiLzipy_LMV1mIYa_MFOvzYeA88tFgVkz8bpJCOLIDf8-3xMKxau_-xglR7bJAVkqBzoJerxwY8BHZTaEyTBW6RGhGE9mihXEVr_KVGPhgCoq7AzPgEpbfzrbiPh1rpp9rpgg3Vk0qjwxkP7TKclAYmvPacqfxxhN_eY7ci4aIW6qPwMp6WKE26fVlqQL385ARha1hlhUJpbQ5f8UMd2o2B83wV4FgirYzpnoydBq34anmRy7kfsXxgWos90iVqzsJaSs7vtl5sqRsY0BzxOOzMg8yz7QhM-hJ57fSv5Sndx5zVXZ1mrIFI-WUqIx4nR4Vv1_pO-5s-w162myOksQuQbr7JFcUzX6J0bvgfOSYVMFOVJ0c_-m8UQXJs08qSfjXkWf61v1x2h2lzWbpQQIWZ1u7md0-B52xF1WlVgs3F_JjtWjQjtQ5Ydbb4-lhXabU2i28Vk_V3l_qzheRyMm_mfhg-Mwixv3IoQFpmoiQrYB7ovRr8hfg9YreI0F1JMYUdu1587vc01-z5QoFXyp9SfFYg32ZkOYIdx2M3_ge1rhRleuocqS-lUSjyw56RggXlj0bqy52Q1UYtYBw0f9I3r1QlxzRhswFcbs2WoYQzqqJYajVzulYyzUQhOuIOR_uVIRX4R1tBy7MRhpc-khkW8bZ41516fzbQRJ13qzjF1mn_B38ll2dfnttnBnjOUUimvkc26-zfasrWwdtmJhOr2eaxs1Vjjz_t_97Jj0sXI3xSV-rUMf0VkByQt7UOtZjltWgrQ0yajhbobxy5MhSQnolrxOsFgm4tuYot8tJonjYJJiwtzs1iwMz9'\n\nwhile lsof /var/lib/dpkg/lock; do sleep 1; done\n\nHOME=\"$(mktemp -d)\" exec $(which cml-runner || echo $(which cml-internal || echo cml) runner) \\\n   --name cml-ec7vbos7hr-h338b14h-65p5e7ja \\\n   --labels cml-runner \\\n   --idle-timeout 300 \\\n   --driver github \\\n   --repo https://github.com/swiss-ai-center/cml-kubernetes-github-actions-runner-minimal-reproducible-example \\\n   --token *** \\\n   \\\n   \\\n   --tf-resource eyJtb2RlIjoibWFuYWdlZCIsInR5cGUiOiJpdGVyYXRpdmVfY21sX3J1bm5lciIsIm5hbWUiOiJydW5uZXIiLCJwcm92aWRlciI6InByb3ZpZGVyW1wicmVnaXN0cnkudGVycmFmb3JtLmlvL2l0ZXJhdGl2ZS9pdGVyYXRpdmVcIl0iLCJpbnN0YW5jZXMiOlt7InByaXZhdGUiOiIiLCJzY2hlbWFfdmVyc2lvbiI6MCwiYXR0cmlidXRlcyI6eyJuYW1lIjoiY21sLWVjN3Zib3M3aHItaDMzOGIxNGgtNjVwNWU3amEiLCJsYWJlbHMiOiIiLCJpZGxlX3RpbWVvdXQiOjMwMCwicmVwbyI6IiIsInRva2VuIjoiIiwiZHJpdmVyIjoiIiwiY2xvdWQiOiJrdWJlcm5ldGVzIiwic3BvdCI6ZmFsc2UsImN1c3RvbV9kYXRhIjoiIiwiaWQiOiJjbWwtZWM3dmJvczdoci1oMzM4YjE0aC02NXA1ZTdqYSIsImltYWdlIjoiIiwiaW5zdGFuY2VfZ3B1IjoiIiwiaW5zdGFuY2VfaGRkX3NpemUiOjM1LCJpbnN0YW5jZV9pcCI6IiIsImluc3RhbmNlX2xhdW5jaF90aW1lIjoiIiwiaW5zdGFuY2VfdHlwZSI6IiIsInJlZ2lvbiI6InVzLXdlc3QiLCJzc2hfbmFtZSI6IiIsInNzaF9wcml2YXRlIjoiIiwic3NoX3B1YmxpYyI6IiIsImF3c19zZWN1cml0eV9ncm91cCI6IiJ9fV19\n"***, Args:[]string(nil), WorkingDir:"", Ports:[]v1.ContainerPort(nil), EnvFrom:[]v1.EnvFromSource(nil), Env:[]v1.EnvVar(nil), Resources:v1.ResourceRequirements***Limits:v1.ResourceList***"cpu":resource.Quantity***i:resource.int64Amount***value:1, scale:0***, d:resource.infDecAmount***Dec:(*inf.Dec)(nil)***, s:"1", Format:"DecimalSI"***, "ephemeral-storage":resource.Quantity***i:resource.int64Amount***value:35, scale:9***, d:resource.infDecAmount***Dec:(*inf.Dec)(nil)***, s:"35G", Format:"DecimalSI"***, "memory":resource.Quantity***i:resource.int64Amount***value:1073741824, scale:0***, d:resource.infDecAmount***Dec:(*inf.Dec)(nil)***, s:"1Gi", Format:"BinarySI"***, Requests:v1.ResourceList***"cpu":resource.Quantity***i:resource.int64Amount***value:0, scale:0***, d:resource.infDecAmount***Dec:(*inf.Dec)(nil)***, s:"0", Format:"DecimalSI"***, "memory":resource.Quantity***i:resource.int64Amount***value:0, scale:0***, d:resource.infDecAmount***Dec:(*inf.Dec)(nil)***, s:"0", Format:"DecimalSI"***, VolumeMounts:[]v1.VolumeMount(nil), VolumeDevices:[]v1.VolumeDevice(nil), LivenessProbe:(*v1.Probe)(nil), ReadinessProbe:(*v1.Probe)(nil), StartupProbe:(*v1.Probe)(nil), Lifecycle:(*v1.Lifecycle)(nil), TerminationMessagePath:"", TerminationMessagePolicy:"", ImagePullPolicy:"", SecurityContext:(*v1.SecurityContext)(nil), Stdin:false, StdinOnce:false, TTY:false***, EphemeralContainers:[]v1.EphemeralContainer(nil), RestartPolicy:"Never", TerminationGracePeriodSeconds:(*int64)(0xc000825da8), ActiveDeadlineSeconds:(*int64)(nil), DNSPolicy:"", NodeSelector:map[string]string***, ServiceAccountName:"", DeprecatedServiceAccount:"", AutomountServiceAccountToken:(*bool)(nil), NodeName:"", HostNetwork:false, HostPID:false, HostIPC:false, ShareProcessNamespace:(*bool)(nil), SecurityContext:(*v1.PodSecurityContext)(nil), ImagePullSecrets:[]v1.LocalObjectReference(nil), Hostname:"", Subdomain:"", Affinity:(*v1.Affinity)(nil), SchedulerName:"", Tolerations:[]v1.Toleration(nil), HostAliases:[]v1.HostAlias(nil), PriorityClassName:"", Priority:(*int32)(nil), DNSConfig:(*v1.PodDNSConfig)(nil), ReadinessGates:[]v1.PodReadinessGate(nil), RuntimeClassName:(*string)(nil), EnableServiceLinks:(*bool)(nil), PreemptionPolicy:(*v1.PreemptionPolicy)(nil), Overhead:v1.ResourceList(nil), TopologySpreadConstraints:[]v1.TopologySpreadConstraint(nil), SetHostnameAsFQDN:(*bool)(nil)***, TTLSecondsAfterFinished:(*int32)(0xc000825da0), CompletionMode:(*v1.CompletionMode)(nil), Suspend:(*bool)(nil)***, Status:v1.JobStatus***Conditions:[]v1.JobCondition(nil), StartTime:<nil>, CompletionTime:<nil>, Active:0, Succeeded:0, Failed:0, CompletedIndexes:"", UncountedTerminatedPods:(*v1.UncountedTerminatedPods)(nil)***: timestamp=2024-02-28T12:20:45.188Z
2024-02-28T12:40:45.9440496Z 2024-02-28T12:20:45.890Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:20:45 [INFO] Submitted new job: &v1.Job***TypeMeta:v1.TypeMeta***Kind:"", APIVersion:""***, ObjectMeta:v1.ObjectMeta***Name:"cml-ec7vbos7hr-h338b14h-65p5e7ja", GenerateName:"", Namespace:"default", SelfLink:"", UID:"b70e94fb-afdd-45dc-9817-ac92943bb8f7", ResourceVersion:"40557", Generation:1, CreationTimestamp:time.Date(2024, time.February, 28, 12, 20, 45, 0, time.Local), DeletionTimestamp:<nil>, DeletionGracePeriodSeconds:(*int64)(nil), Labels:map[string]string***"batch.kubernetes.io/controller-uid":"b70e94fb-afdd-45dc-9817-ac92943bb8f7", "batch.kubernetes.io/job-name":"cml-ec7vbos7hr-h338b14h-65p5e7ja", "controller-uid":"b70e94fb-afdd-45dc-9817-ac92943bb8f7", "job-name":"cml-ec7vbos7hr-h338b14h-65p5e7ja"***, Annotations:map[string]string***"batch.kubernetes.io/job-tracking":""***, OwnerReferences:[]v1.OwnerReference(nil), Finalizers:[]string(nil), ClusterName:"", ManagedFields:[]v1.ManagedFieldsEntry***v1.ManagedFieldsEntry***Manager:"terraform-provider-iterative", Operation:"Update", APIVersion:"batch/v1", Time:time.Date(2024, time.February, 28, 12, 20, 45, 0, time.Local), FieldsType:"FieldsV1", FieldsV1:(*v1.FieldsV1)(0xc000a1b050), Subresource:""***, Spec:v1.JobSpec***Parallelism:(*int32)(0xc000aa0ea8), Completions:(*int32)(0xc000aa0eac), ActiveDeadlineSeconds:(*int64)(nil), BackoffLimit:(*int32)(0xc000aa0ee0), Selector:(*v1.LabelSelector)(0xc000c8e3e0), ManualSelector:(*bool)(nil), Template:v1.PodTemplateSpec***ObjectMeta:v1.ObjectMeta***Name:"", GenerateName:"", Namespace:"", SelfLink:"", UID:"", ResourceVersion:"", Generation:0, CreationTimestamp:time.Date(1, time.January, 1, 0, 0, 0, 0, time.UTC), DeletionTimestamp:<nil>, DeletionGracePeriodSeconds:(*int64)(nil), Labels:map[string]string***"batch.kubernetes.io/controller-uid":"b70e94fb-afdd-45dc-9817-ac92943bb8f7", "batch.kubernetes.io/job-name":"cml-ec7vbos7hr-h338b14h-65p5e7ja", "controller-uid":"b70e94fb-afdd-45dc-9817-ac92943bb8f7", "job-name":"cml-ec7vbos7hr-h338b14h-65p5e7ja"***, Annotations:map[string]string(nil), OwnerReferences:[]v1.OwnerReference(nil), Finalizers:[]string(nil), ClusterName:"", ManagedFields:[]v1.ManagedFieldsEntry(nil)***, Spec:v1.PodSpec***Volumes:[]v1.Volume(nil), InitContainers:[]v1.Container(nil), Containers:[]v1.Container***v1.Container***Name:"cml-ec7vbos7hr-h338b14h-65p5e7ja", Image:"dvcorg/cml:0-dvc2-base1-gpu", Command:[]string***"bash", "-c", "#!/bin/sh\nsudo systemctl is-enabled cml.service && return 0\n\nsudo curl --location https://github.com/iterative/terraform-provider-iterative/releases/latest/download/leo_linux_amd64 --output /usr/bin/leo\nsudo chmod a+x /usr/bin/leo\nexport KUBERNETES_CONFIGURATION='apiVersion: v1\nclusters:\n  - cluster:\n      certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUVMRENDQXBTZ0F3SUJBZ0lRUlVVbCtYWmVpVVRUT0drQ3VsbHhoakFOQmdrcWhraUc5dzBCQVFzRkFEQXYKTVMwd0t3WURWUVFERXlSbU9XRTJZbUUwWmkwME1qUmtMVFJtTUdNdE9XRXdOUzA0TXpaaVpERmhaR0UwWlRFdwpJQmNOTWpRd01qSTRNRGswT0RFM1doZ1BNakExTkRBeU1qQXhNRFE0TVRkYU1DOHhMVEFyQmdOVkJBTVRKR1k1CllUWmlZVFJtTFRReU5HUXROR1l3WXkwNVlUQTFMVGd6Tm1Ka01XRmtZVFJsTVRDQ0FhSXdEUVlKS29aSWh2Y04KQVFFQkJRQURnZ0dQQURDQ0FZb0NnZ0dCQU53Vy93ZW1SQ1hUTGZDWEQzeS93bVZLMUJSZGY3YWNiUTlieEY0SwpNRitqdDU5UFAzZlA4QkRpQy9pR1JsTDdFcmhNYkRuTjhTWVN5cG9YYUk2c0pTZ1RlZ3pEcjMzUDlxQm9hWndSClVwUy9lZC9WSGtwS0ZqVWNMQTR6TFM5MW9iMVdZMTEvcHY0SnRhNHhjQ2F4ejg4SUxPV3V1M0N1dTljTmh2c3gKMy9tVVRKRm1iN0VPMGZzdHVCWTVadXU1S3cyUGh1K2RXdm1KVHdJZFNtNlJrc3l0OEpleXFOd016c3I5b3RucwpTdXNvaXFzRlJvMTE5NnlwLzR6R0R4WWhGNWR4NDhJZFE0SjBVYWgyV0FxdkhDWEZMY2tJMy9yODl0YWdFYVlPCkgvOGI4a3oydHhJRlhJL1hodzRjOHdOSkdLaVVJSjUvTGYxRndCejdJZkVXa21vcURNM2NMVERrQ1N4VVFJL2oKc2FEb2FqNUtxdW5hWjBIWWV4WEMyVURFYVV3MDNMVSsyZUxLMUVUOVdJZzJpcEVmamk3cHZTejZHczdPc3o3Tgp6cTk0T2NJOEJLS3MyUjNrd1g1QWxBM1J5T0FmdTk2cGFqWTl2MGlORW94ZHhsd1FmZ2pzd3Z3MmRDQ04vVjc3CkllUDBZOHFqcDBpMmtKdHBVOHA1ZTBkUDB3SURBUUFCbzBJd1FEQU9CZ05WSFE4QkFmOEVCQU1DQWdRd0R3WUQKVlIwVEFRSC9CQVV3QXdFQi96QWRCZ05WSFE0RUZnUVVCSU5HRU8zUzlFeUFoaDhXamxxYS9XaTc5STR3RFFZSgpLb1pJaHZjTkFRRUxCUUFEZ2dHQkFDRW5ES0wxT3FIY3o2WG5zTEttUU9VSWNGQmNRZTZFMGkwaVFzUllhY0tmCjJPOVl2Ykp6enYxejBIdG9IdjF6ZHpFd3NoWmxub3pwM1JjR3h3Zk1aTGNTYkF6czJmekoxNGxkUmhDa1NQYVYKNjZKejBxdzRtZzdFVVVtRFQ2YmtCRDZ3amtWdUt3YmZKVGd5MndMK0JpTDk2V3VuUXdVT2MzOCsrVjM4RjIvNgo2YVZQbnZNd2JtbFU1MnB5RzRoYjMxVS8wN3pYeWkzTWNRN1o2anRQcWRqd2EvTzFXWFkrZTBJZklQU1l4dlZxClE2dzFiVGUwNlRobjZMK2EwbWVLbzNJVWd6VWtLeDV4Z0F6TmswcHlveDlGUmhPZkp4ZE9yVXNhNkZydWJyMGgKNWVWL0tpKzhkWDk5c2V3Q2JEek9PU1h0V0JVN2lnRzRQejNZUFRxZlNKZElFMVZtSHJPS3Y4a2RpS1dGU2cvaApza2FJU0ZtOXFvcGVZL29hbkxONmpjWi9Xdk9CVUh0VE84cW1tUVZDcGFpZHAzQ01Qam1Ndk1VWFpMYXlBYXZiCi9wTHlNV3gvTWhWK3kxYWUrSHpUNUIvNFBlSjhFdjFQOHlCYWRVb1o0MDRSMk9EK3VaQmM5N0dpQ3BkdUhTRjUKaHpDVityc250aW9VTDZrMEkzdTZmQT09Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K\n      server: https://34.34.161.209\n    name: ***\ncontexts:\n  - context:\n      cluster: ***\n      user: ***\n    name: gke_cml-k8s-github-runner-mre_***_***\nkind: Config\ncurrent-context: gke_cml-k8s-github-runner-mre_***_***\nusers:\n  - name: ***\n    user:\n      token: ya29.c.c0AY_VpZiim-sSd5FeCxKmwopUGEp4A-s_DgUwuLJ_-w8M7tU92ltSMlja1GmF-6UzbMZlubMl1lfIvp1VSskeZJChN6ARg4Mdai4VGgPdsVCghlr7o2M3PLG_m3cyxuv9BOmkTiXPbbRXIH5enF0sTLuttViYAJPltZO67_iCjhOgXInMbxBd39ttnKxvkiLzipy_LMV1mIYa_MFOvzYeA88tFgVkz8bpJCOLIDf8-3xMKxau_-xglR7bJAVkqBzoJerxwY8BHZTaEyTBW6RGhGE9mihXEVr_KVGPhgCoq7AzPgEpbfzrbiPh1rpp9rpgg3Vk0qjwxkP7TKclAYmvPacqfxxhN_eY7ci4aIW6qPwMp6WKE26fVlqQL385ARha1hlhUJpbQ5f8UMd2o2B83wV4FgirYzpnoydBq34anmRy7kfsXxgWos90iVqzsJaSs7vtl5sqRsY0BzxOOzMg8yz7QhM-hJ57fSv5Sndx5zVXZ1mrIFI-WUqIx4nR4Vv1_pO-5s-w162myOksQuQbr7JFcUzX6J0bvgfOSYVMFOVJ0c_-m8UQXJs08qSfjXkWf61v1x2h2lzWbpQQIWZ1u7md0-B52xF1WlVgs3F_JjtWjQjtQ5Ydbb4-lhXabU2i28Vk_V3l_qzheRyMm_mfhg-Mwixv3IoQFpmoiQrYB7ovRr8hfg9YreI0F1JMYUdu1587vc01-z5QoFXyp9SfFYg32ZkOYIdx2M3_ge1rhRleuocqS-lUSjyw56RggXlj0bqy52Q1UYtYBw0f9I3r1QlxzRhswFcbs2WoYQzqqJYajVzulYyzUQhOuIOR_uVIRX4R1tBy7MRhpc-khkW8bZ41516fzbQRJ13qzjF1mn_B38ll2dfnttnBnjOUUimvkc26-zfasrWwdtmJhOr2eaxs1Vjjz_t_97Jj0sXI3xSV-rUMf0VkByQt7UOtZjltWgrQ0yajhbobxy5MhSQnolrxOsFgm4tuYot8tJonjYJJiwtzs1iwMz9'\n\nwhile lsof /var/lib/dpkg/lock; do sleep 1; done\n\nHOME=\"$(mktemp -d)\" exec $(which cml-runner || echo $(which cml-internal || echo cml) runner) \\\n   --name cml-ec7vbos7hr-h338b14h-65p5e7ja \\\n   --labels cml-runner \\\n   --idle-timeout 300 \\\n   --driver github \\\n   --repo https://github.com/swiss-ai-center/cml-kubernetes-github-actions-runner-minimal-reproducible-example \\\n   --token *** \\\n   \\\n   \\\n   --tf-resource eyJtb2RlIjoibWFuYWdlZCIsInR5cGUiOiJpdGVyYXRpdmVfY21sX3J1bm5lciIsIm5hbWUiOiJydW5uZXIiLCJwcm92aWRlciI6InByb3ZpZGVyW1wicmVnaXN0cnkudGVycmFmb3JtLmlvL2l0ZXJhdGl2ZS9pdGVyYXRpdmVcIl0iLCJpbnN0YW5jZXMiOlt7InByaXZhdGUiOiIiLCJzY2hlbWFfdmVyc2lvbiI6MCwiYXR0cmlidXRlcyI6eyJuYW1lIjoiY21sLWVjN3Zib3M3aHItaDMzOGIxNGgtNjVwNWU3amEiLCJsYWJlbHMiOiIiLCJpZGxlX3RpbWVvdXQiOjMwMCwicmVwbyI6IiIsInRva2VuIjoiIiwiZHJpdmVyIjoiIiwiY2xvdWQiOiJrdWJlcm5ldGVzIiwic3BvdCI6ZmFsc2UsImN1c3RvbV9kYXRhIjoiIiwiaWQiOiJjbWwtZWM3dmJvczdoci1oMzM4YjE0aC02NXA1ZTdqYSIsImltYWdlIjoiIiwiaW5zdGFuY2VfZ3B1IjoiIiwiaW5zdGFuY2VfaGRkX3NpemUiOjM1LCJpbnN0YW5jZV9pcCI6IiIsImluc3RhbmNlX2xhdW5jaF90aW1lIjoiIiwiaW5zdGFuY2VfdHlwZSI6IiIsInJlZ2lvbiI6InVzLXdlc3QiLCJzc2hfbmFtZSI6IiIsInNzaF9wcml2YXRlIjoiIiwic3NoX3B1YmxpYyI6IiIsImF3c19zZWN1cml0eV9ncm91cCI6IiJ9fV19\n"***, Args:[]string(nil), WorkingDir:"", Ports:[]v1.ContainerPort(nil), EnvFrom:[]v1.EnvFromSource(nil), Env:[]v1.EnvVar(nil), Resources:v1.ResourceRequirements***Limits:v1.ResourceList***"cpu":resource.Quantity***i:resource.int64Amount***value:1, scale:0***, d:resource.infDecAmount***Dec:(*inf.Dec)(nil)***, s:"1", Format:"DecimalSI"***, "ephemeral-storage":resource.Quantity***i:resource.int64Amount***value:35, scale:9***, d:resource.infDecAmount***Dec:(*inf.Dec)(nil)***, s:"35G", Format:"DecimalSI"***, "memory":resource.Quantity***i:resource.int64Amount***value:1073741824, scale:0***, d:resource.infDecAmount***Dec:(*inf.Dec)(nil)***, s:"1Gi", Format:"BinarySI"***, Requests:v1.ResourceList***"cpu":resource.Quantity***i:resource.int64Amount***value:0, scale:0***, d:resource.infDecAmount***Dec:(*inf.Dec)(nil)***, s:"0", Format:"DecimalSI"***, "memory":resource.Quantity***i:resource.int64Amount***value:0, scale:0***, d:resource.infDecAmount***Dec:(*inf.Dec)(nil)***, s:"0", Format:"DecimalSI"***, VolumeMounts:[]v1.VolumeMount(nil), VolumeDevices:[]v1.VolumeDevice(nil), LivenessProbe:(*v1.Probe)(nil), ReadinessProbe:(*v1.Probe)(nil), StartupProbe:(*v1.Probe)(nil), Lifecycle:(*v1.Lifecycle)(nil), TerminationMessagePath:"/dev/termination-log", TerminationMessagePolicy:"File", ImagePullPolicy:"IfNotPresent", SecurityContext:(*v1.SecurityContext)(nil), Stdin:false, StdinOnce:false, TTY:false***, EphemeralContainers:[]v1.EphemeralContainer(nil), RestartPolicy:"Never", TerminationGracePeriodSeconds:(*int64)(0xc000aa0fb8), ActiveDeadlineSeconds:(*int64)(nil), DNSPolicy:"ClusterFirst", NodeSelector:map[string]string(nil), ServiceAccountName:"", DeprecatedServiceAccount:"", AutomountServiceAccountToken:(*bool)(nil), NodeName:"", HostNetwork:false, HostPID:false, HostIPC:false, ShareProcessNamespace:(*bool)(nil), SecurityContext:(*v1.PodSecurityContext)(0xc00046a230), ImagePullSecrets:[]v1.LocalObjectReference(nil), Hostname:"", Subdomain:"", Affinity:(*v1.Affinity)(nil), SchedulerName:"default-scheduler", Tolerations:[]v1.Toleration(nil), HostAliases:[]v1.HostAlias(nil), PriorityClassName:"", Priority:(*int32)(nil), DNSConfig:(*v1.PodDNSConfig)(nil), ReadinessGates:[]v1.PodReadinessGate(nil), RuntimeClassName:(*string)(nil), EnableServiceLinks:(*bool)(nil), PreemptionPolicy:(*v1.PreemptionPolicy)(nil), Overhead:v1.ResourceList(nil), TopologySpreadConstraints:[]v1.TopologySpreadConstraint(nil), SetHostnameAsFQDN:(*bool)(nil)***, TTLSecondsAfterFinished:(*int32)(0xc000aa0fcc), CompletionMode:(*v1.CompletionMode)(0xc0000fe130), Suspend:(*bool)(0xc000aa100a)***, Status:v1.JobStatus***Conditions:[]v1.JobCondition(nil), StartTime:<nil>, CompletionTime:<nil>, Active:0, Succeeded:0, Failed:0, CompletedIndexes:"", UncountedTerminatedPods:(*v1.UncountedTerminatedPods)(nil)***: timestamp=2024-02-28T12:20:45.890Z
2024-02-28T12:40:45.9542829Z 2024-02-28T12:20:45.890Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:20:45 [DEBUG] Waiting for state to become: [success]: timestamp=2024-02-28T12:20:45.890Z
2024-02-28T12:40:45.9544692Z 2024-02-28T12:20:46.066Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:20:46 [TRACE] Waiting 500ms before next try: timestamp=2024-02-28T12:20:46.066Z
2024-02-28T12:40:45.9546669Z 2024-02-28T12:20:46.721Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:20:46 [TRACE] Waiting 1s before next try: timestamp=2024-02-28T12:20:46.721Z
2024-02-28T12:40:45.9548499Z 2024-02-28T12:20:47.873Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:20:47 [TRACE] Waiting 2s before next try: timestamp=2024-02-28T12:20:47.873Z
2024-02-28T12:40:45.9550279Z 2024-02-28T12:20:50.025Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:20:50 [TRACE] Waiting 4s before next try: timestamp=2024-02-28T12:20:50.025Z
2024-02-28T12:40:45.9552412Z 2024-02-28T12:20:54.179Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:20:54 [TRACE] Waiting 8s before next try: timestamp=2024-02-28T12:20:54.179Z
2024-02-28T12:40:45.9554223Z 2024-02-28T12:21:02.334Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:21:02 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:21:02.334Z
2024-02-28T12:40:45.9556065Z 2024-02-28T12:21:12.487Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:21:12 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:21:12.487Z
2024-02-28T12:40:45.9557816Z 2024-02-28T12:21:22.642Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:21:22 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:21:22.642Z
2024-02-28T12:40:45.9559636Z 2024-02-28T12:21:32.795Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:21:32 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:21:32.795Z
2024-02-28T12:40:45.9561353Z 2024-02-28T12:21:42.948Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:21:42 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:21:42.948Z
2024-02-28T12:40:45.9563225Z 2024-02-28T12:21:53.102Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:21:53 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:21:53.102Z
2024-02-28T12:40:45.9565080Z 2024-02-28T12:22:03.254Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:22:03 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:22:03.254Z
2024-02-28T12:40:45.9566847Z 2024-02-28T12:22:13.407Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:22:13 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:22:13.407Z
2024-02-28T12:40:45.9568879Z 2024-02-28T12:22:23.559Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:22:23 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:22:23.559Z
2024-02-28T12:40:45.9570655Z 2024-02-28T12:22:33.712Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:22:33 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:22:33.712Z
2024-02-28T12:40:45.9572433Z 2024-02-28T12:22:43.864Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:22:43 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:22:43.864Z
2024-02-28T12:40:45.9574195Z 2024-02-28T12:22:54.057Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:22:54 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:22:54.057Z
2024-02-28T12:40:45.9576122Z 2024-02-28T12:23:04.209Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:23:04 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:23:04.209Z
2024-02-28T12:40:45.9578007Z 2024-02-28T12:23:14.362Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:23:14 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:23:14.361Z
2024-02-28T12:40:45.9579750Z 2024-02-28T12:23:24.516Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:23:24 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:23:24.516Z
2024-02-28T12:40:45.9581547Z 2024-02-28T12:23:34.669Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:23:34 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:23:34.669Z
2024-02-28T12:40:45.9583280Z 2024-02-28T12:23:44.823Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:23:44 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:23:44.823Z
2024-02-28T12:40:45.9585164Z 2024-02-28T12:23:54.979Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:23:54 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:23:54.978Z
2024-02-28T12:40:45.9586892Z 2024-02-28T12:24:05.132Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:24:05 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:24:05.132Z
2024-02-28T12:40:45.9588715Z 2024-02-28T12:24:15.285Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:24:15 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:24:15.285Z
2024-02-28T12:40:45.9590461Z 2024-02-28T12:24:25.435Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:24:25 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:24:25.435Z
2024-02-28T12:40:45.9592683Z 2024-02-28T12:24:35.590Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:24:35 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:24:35.590Z
2024-02-28T12:40:45.9594541Z 2024-02-28T12:24:45.745Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:24:45 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:24:45.745Z
2024-02-28T12:40:45.9596313Z 2024-02-28T12:24:55.944Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:24:55 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:24:55.944Z
2024-02-28T12:40:45.9598116Z 2024-02-28T12:25:06.123Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:25:06 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:25:06.123Z
2024-02-28T12:40:45.9599864Z 2024-02-28T12:25:16.279Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:25:16 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:25:16.279Z
2024-02-28T12:40:45.9601716Z 2024-02-28T12:25:26.434Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:25:26 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:25:26.434Z
2024-02-28T12:40:45.9603559Z 2024-02-28T12:25:36.591Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:25:36 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:25:36.591Z
2024-02-28T12:40:45.9605364Z 2024-02-28T12:25:46.746Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:25:46 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:25:46.746Z
2024-02-28T12:40:45.9607079Z 2024-02-28T12:25:56.901Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:25:56 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:25:56.901Z
2024-02-28T12:40:45.9609104Z 2024-02-28T12:26:07.056Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:26:07 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:26:07.056Z
2024-02-28T12:40:45.9610902Z 2024-02-28T12:26:17.217Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:26:17 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:26:17.217Z
2024-02-28T12:40:45.9612683Z 2024-02-28T12:26:27.369Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:26:27 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:26:27.369Z
2024-02-28T12:40:45.9614666Z 2024-02-28T12:26:37.521Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:26:37 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:26:37.521Z
2024-02-28T12:40:45.9616450Z 2024-02-28T12:26:47.672Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:26:47 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:26:47.672Z
2024-02-28T12:40:45.9618245Z 2024-02-28T12:26:57.856Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:26:57 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:26:57.856Z
2024-02-28T12:40:45.9620048Z 2024-02-28T12:27:08.007Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:27:08 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:27:08.007Z
2024-02-28T12:40:45.9621854Z 2024-02-28T12:27:18.159Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:27:18 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:27:18.159Z
2024-02-28T12:40:45.9623668Z 2024-02-28T12:27:28.310Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:27:28 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:27:28.309Z
2024-02-28T12:40:45.9625385Z 2024-02-28T12:27:38.460Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:27:38 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:27:38.460Z
2024-02-28T12:40:45.9627256Z 2024-02-28T12:27:48.611Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:27:48 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:27:48.611Z
2024-02-28T12:40:45.9628978Z 2024-02-28T12:27:58.764Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:27:58 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:27:58.764Z
2024-02-28T12:40:45.9630798Z 2024-02-28T12:28:08.916Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:28:08 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:28:08.916Z
2024-02-28T12:40:45.9632741Z 2024-02-28T12:28:19.069Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:28:19 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:28:19.069Z
2024-02-28T12:40:45.9634600Z 2024-02-28T12:28:29.221Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:28:29 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:28:29.221Z
2024-02-28T12:40:45.9636327Z 2024-02-28T12:28:39.372Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:28:39 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:28:39.372Z
2024-02-28T12:40:45.9638125Z 2024-02-28T12:28:49.529Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:28:49 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:28:49.528Z
2024-02-28T12:40:45.9640009Z 2024-02-28T12:28:59.709Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:28:59 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:28:59.709Z
2024-02-28T12:40:45.9641719Z 2024-02-28T12:29:09.864Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:29:09 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:29:09.864Z
2024-02-28T12:40:45.9643562Z 2024-02-28T12:29:20.017Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:29:20 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:29:20.017Z
2024-02-28T12:40:45.9645270Z 2024-02-28T12:29:30.170Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:29:30 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:29:30.170Z
2024-02-28T12:40:45.9647370Z 2024-02-28T12:29:40.323Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:29:40 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:29:40.323Z
2024-02-28T12:40:45.9649104Z 2024-02-28T12:29:50.476Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:29:50 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:29:50.476Z
2024-02-28T12:40:45.9650913Z 2024-02-28T12:30:00.631Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:30:00 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:30:00.630Z
2024-02-28T12:40:45.9652833Z 2024-02-28T12:30:10.784Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:30:10 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:30:10.784Z
2024-02-28T12:40:45.9654690Z 2024-02-28T12:30:20.938Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:30:20 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:30:20.938Z
2024-02-28T12:40:45.9656469Z 2024-02-28T12:30:31.092Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:30:31 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:30:31.092Z
2024-02-28T12:40:45.9658223Z 2024-02-28T12:30:41.246Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:30:41 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:30:41.246Z
2024-02-28T12:40:45.9659990Z 2024-02-28T12:30:51.402Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:30:51 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:30:51.402Z
2024-02-28T12:40:45.9661747Z 2024-02-28T12:31:01.586Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:31:01 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:31:01.586Z
2024-02-28T12:40:45.9663543Z 2024-02-28T12:31:11.741Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:31:11 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:31:11.741Z
2024-02-28T12:40:45.9665338Z 2024-02-28T12:31:21.899Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:31:21 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:31:21.899Z
2024-02-28T12:40:45.9667186Z 2024-02-28T12:31:32.050Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:31:32 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:31:32.050Z
2024-02-28T12:40:45.9668888Z 2024-02-28T12:31:42.200Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:31:42 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:31:42.200Z
2024-02-28T12:40:45.9670706Z 2024-02-28T12:31:52.354Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:31:52 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:31:52.354Z
2024-02-28T12:40:45.9842583Z 2024-02-28T12:32:02.505Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:32:02 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:32:02.505Z
2024-02-28T12:40:45.9845507Z 2024-02-28T12:32:12.656Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:32:12 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:32:12.656Z
2024-02-28T12:40:45.9848259Z 2024-02-28T12:32:22.808Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:32:22 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:32:22.808Z
2024-02-28T12:40:45.9851085Z 2024-02-28T12:32:32.959Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:32:32 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:32:32.959Z
2024-02-28T12:40:45.9853831Z 2024-02-28T12:32:43.110Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:32:43 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:32:43.110Z
2024-02-28T12:40:45.9856768Z 2024-02-28T12:32:53.260Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:32:53 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:32:53.260Z
2024-02-28T12:40:45.9859713Z 2024-02-28T12:33:03.451Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:33:03 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:33:03.451Z
2024-02-28T12:40:45.9862989Z 2024-02-28T12:33:13.610Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:33:13 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:33:13.610Z
2024-02-28T12:40:45.9865849Z 2024-02-28T12:33:23.765Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:33:23 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:33:23.765Z
2024-02-28T12:40:45.9868755Z 2024-02-28T12:33:33.917Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:33:33 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:33:33.917Z
2024-02-28T12:40:45.9872307Z 2024-02-28T12:33:44.068Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:33:44 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:33:44.068Z
2024-02-28T12:40:45.9875257Z 2024-02-28T12:33:54.220Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:33:54 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:33:54.220Z
2024-02-28T12:40:45.9878184Z 2024-02-28T12:34:04.371Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:34:04 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:34:04.371Z
2024-02-28T12:40:45.9881080Z 2024-02-28T12:34:14.528Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:34:14 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:34:14.528Z
2024-02-28T12:40:45.9884071Z 2024-02-28T12:34:24.683Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:34:24 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:34:24.683Z
2024-02-28T12:40:45.9887031Z 2024-02-28T12:34:34.837Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:34:34 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:34:34.837Z
2024-02-28T12:40:45.9889998Z 2024-02-28T12:34:44.991Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:34:44 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:34:44.991Z
2024-02-28T12:40:45.9892200Z 2024-02-28T12:34:55.144Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:34:55 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:34:55.143Z
2024-02-28T12:40:45.9893847Z 2024-02-28T12:35:05.323Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:35:05 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:35:05.323Z
2024-02-28T12:40:45.9895421Z 2024-02-28T12:35:15.476Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:35:15 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:35:15.476Z
2024-02-28T12:40:45.9897045Z 2024-02-28T12:35:25.631Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:35:25 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:35:25.631Z
2024-02-28T12:40:45.9898643Z 2024-02-28T12:35:35.786Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:35:35 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:35:35.785Z
2024-02-28T12:40:45.9900227Z 2024-02-28T12:35:45.938Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:35:45 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:35:45.938Z
2024-02-28T12:40:45.9901850Z 2024-02-28T12:35:56.093Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:35:56 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:35:56.093Z
2024-02-28T12:40:45.9903420Z 2024-02-28T12:36:06.302Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:36:06 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:36:06.302Z
2024-02-28T12:40:45.9905007Z 2024-02-28T12:36:16.456Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:36:16 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:36:16.456Z
2024-02-28T12:40:45.9906577Z 2024-02-28T12:36:26.611Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:36:26 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:36:26.611Z
2024-02-28T12:40:45.9908182Z 2024-02-28T12:36:36.766Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:36:36 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:36:36.766Z
2024-02-28T12:40:45.9909999Z 2024-02-28T12:36:46.921Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:36:46 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:36:46.921Z
2024-02-28T12:40:45.9911839Z 2024-02-28T12:36:57.076Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:36:57 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:36:57.076Z
2024-02-28T12:40:45.9913435Z 2024-02-28T12:37:07.265Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:37:07 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:37:07.265Z
2024-02-28T12:40:45.9914999Z 2024-02-28T12:37:17.416Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:37:17 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:37:17.416Z
2024-02-28T12:40:45.9916778Z 2024-02-28T12:37:27.568Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:37:27 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:37:27.568Z
2024-02-28T12:40:45.9918386Z 2024-02-28T12:37:37.719Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:37:37 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:37:37.719Z
2024-02-28T12:40:45.9919977Z 2024-02-28T12:37:47.872Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:37:47 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:37:47.872Z
2024-02-28T12:40:45.9921566Z 2024-02-28T12:37:58.024Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:37:58 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:37:58.023Z
2024-02-28T12:40:45.9923126Z 2024-02-28T12:38:08.176Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:38:08 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:38:08.176Z
2024-02-28T12:40:45.9924689Z 2024-02-28T12:38:18.327Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:38:18 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:38:18.327Z
2024-02-28T12:40:45.9926264Z 2024-02-28T12:38:28.482Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:38:28 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:38:28.482Z
2024-02-28T12:40:45.9927834Z 2024-02-28T12:38:38.635Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:38:38 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:38:38.635Z
2024-02-28T12:40:45.9929404Z 2024-02-28T12:38:48.788Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:38:48 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:38:48.788Z
2024-02-28T12:40:45.9931006Z 2024-02-28T12:38:58.949Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:38:58 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:38:58.949Z
2024-02-28T12:40:45.9932911Z 2024-02-28T12:39:09.131Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:39:09 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:39:09.131Z
2024-02-28T12:40:45.9934505Z 2024-02-28T12:39:19.284Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:39:19 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:39:19.284Z
2024-02-28T12:40:45.9936091Z 2024-02-28T12:39:29.438Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:39:29 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:39:29.438Z
2024-02-28T12:40:45.9937653Z 2024-02-28T12:39:39.591Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:39:39 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:39:39.591Z
2024-02-28T12:40:45.9939212Z 2024-02-28T12:39:49.745Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:39:49 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:39:49.745Z
2024-02-28T12:40:45.9940778Z 2024-02-28T12:39:59.899Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:39:59 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:39:59.899Z
2024-02-28T12:40:45.9942340Z 2024-02-28T12:40:10.054Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:40:10 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:40:10.054Z
2024-02-28T12:40:45.9944071Z 2024-02-28T12:40:20.208Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:40:20 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:40:20.207Z
2024-02-28T12:40:45.9945649Z 2024-02-28T12:40:30.361Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:40:30 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:40:30.361Z
2024-02-28T12:40:45.9947280Z 2024-02-28T12:40:40.515Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:40:40 [TRACE] Waiting 10s before next try: timestamp=2024-02-28T12:40:40.515Z
2024-02-28T12:40:45.9948905Z 2024-02-28T12:40:45.893Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:40:45 [WARN] WaitForState timeout after 20m0s: timestamp=2024-02-28T12:40:45.893Z
2024-02-28T12:40:45.9950724Z 2024-02-28T12:40:45.893Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:40:45 [WARN] WaitForState starting 30s refresh grace period: timestamp=2024-02-28T12:40:45.893Z
2024-02-28T12:40:45.9952756Z 2024-02-28T12:40:45.894Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:40:45 [INFO] Deleting job: "cml-ec7vbos7hr-h338b14h-65p5e7ja": timestamp=2024-02-28T12:40:45.894Z
2024-02-28T12:40:45.9954672Z 2024-02-28T12:40:45.894Z [INFO]  provider.terraform-provider-iterative: 2024/02/28 12:40:45 [DEBUG] Received error: context.deadlineExceededError***: timestamp=2024-02-28T12:40:45.894Z
2024-02-28T12:40:45.9956390Z 2024-02-28T12:40:45.898Z [DEBUG] provider.stdio: received EOF, stopping recv loop: err="rpc error: code = Unavailable desc = error reading from server: EOF"
2024-02-28T12:40:45.9958187Z 2024-02-28T12:40:45.901Z [DEBUG] provider: plugin process exited: path=.terraform/providers/registry.terraform.io/iterative/iterative/0.11.20/linux_amd64/terraform-provider-iterative pid=1914
2024-02-28T12:40:45.9959450Z 2024-02-28T12:40:45.901Z [DEBUG] provider: plugin exited
2024-02-28T12:40:46.2458625Z ***"level":"error","message":"terraform apply error","stack":"Error: terraform apply error\n    at Object.apply (/snapshot/cml/src/terraform.js:55:11)\n    at processTicksAndRejections (node:internal/process/task_queues:96:5)\n    at async runTerraform (/snapshot/cml/bin/cml/runner/launch.js:184:5)\n    at async runCloud (/snapshot/cml/bin/cml/runner/launch.js:193:19)\n    at async run (/snapshot/cml/bin/cml/runner/launch.js:435:14)\n    at async Object.exports.handler (/snapshot/cml/bin/cml/runner/launch.js:448:5)"***
2024-02-28T12:40:46.2528044Z ##[error]Process completed with exit code 1.
2024-02-28T12:40:46.2617921Z Post job cleanup.
2024-02-28T12:40:46.3457626Z Removed exported credentials at "/home/runner/work/cml-kubernetes-github-actions-runner-minimal-reproducible-example/cml-kubernetes-github-actions-runner-minimal-reproducible-example/gha-creds-1112903afc7915c2.json".
2024-02-28T12:40:46.3601378Z Post job cleanup.
2024-02-28T12:40:46.4338937Z [command]/usr/bin/git version
2024-02-28T12:40:46.4379167Z git version 2.43.2
2024-02-28T12:40:46.4417033Z Copying '/home/runner/.gitconfig' to '/home/runner/work/_temp/e0d4975b-bc05-4d88-a795-7645116d3a2f/.gitconfig'
2024-02-28T12:40:46.4428698Z Temporarily overriding HOME='/home/runner/work/_temp/e0d4975b-bc05-4d88-a795-7645116d3a2f' before making global git config changes
2024-02-28T12:40:46.4430799Z Adding repository directory to the temporary git global config as a safe directory
2024-02-28T12:40:46.4436657Z [command]/usr/bin/git config --global --add safe.directory /home/runner/work/cml-kubernetes-github-actions-runner-minimal-reproducible-example/cml-kubernetes-github-actions-runner-minimal-reproducible-example
2024-02-28T12:40:46.4471728Z [command]/usr/bin/git config --local --name-only --get-regexp core\.sshCommand
2024-02-28T12:40:46.4502771Z [command]/usr/bin/git submodule foreach --recursive sh -c "git config --local --name-only --get-regexp 'core\.sshCommand' && git config --local --unset-all 'core.sshCommand' || :"
2024-02-28T12:40:46.4761122Z [command]/usr/bin/git config --local --name-only --get-regexp http\.https\:\/\/github\.com\/\.extraheader
2024-02-28T12:40:46.4782151Z http.https://github.com/.extraheader
2024-02-28T12:40:46.4794418Z [command]/usr/bin/git config --local --unset-all http.https://github.com/.extraheader
2024-02-28T12:40:46.4822473Z [command]/usr/bin/git submodule foreach --recursive sh -c "git config --local --name-only --get-regexp 'http\.https\:\/\/github\.com\/\.extraheader' && git config --local --unset-all 'http.https://github.com/.extraheader' || :"
2024-02-28T12:40:46.5280068Z Cleaning up orphan processes
```

</details>

<details>
<summary>Expand to see the full log of the Kubernetes pod</summary>

```json
[
  {
    "textPayload": "Failed to get unit file state for cml.service: No such file or directory",
    "insertId": "szpl6te9br98kz2m",
    "resource": {
      "type": "k8s_container",
      "labels": {
        "container_name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq",
        "namespace_name": "default",
        "pod_name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq-89rbs",
        "location": "europe-west1-b",
        "cluster_name": "kubernetes-cluster",
        "project_id": "cml-k8s-github-runner-mre"
      }
    },
    "timestamp": "2024-02-28T13:19:19.463409540Z",
    "severity": "ERROR",
    "labels": {
      "compute.googleapis.com/resource_name": "gke-kubernetes-cluster-default-pool-96aa9230-hqq3",
      "k8s-pod/batch_kubernetes_io/job-name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq",
      "k8s-pod/job-name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq",
      "k8s-pod/batch_kubernetes_io/controller-uid": "906ace45-2ab9-40cc-aa48-7d29b74a73b9",
      "k8s-pod/controller-uid": "906ace45-2ab9-40cc-aa48-7d29b74a73b9"
    },
    "logName": "projects/cml-k8s-github-runner-mre/logs/stderr",
    "receiveTimestamp": "2024-02-28T13:19:21.719792035Z"
  },
  {
    "textPayload": "  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current",
    "insertId": "4siecsbn154mhggv",
    "resource": {
      "type": "k8s_container",
      "labels": {
        "pod_name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq-89rbs",
        "location": "europe-west1-b",
        "cluster_name": "kubernetes-cluster",
        "project_id": "cml-k8s-github-runner-mre",
        "namespace_name": "default",
        "container_name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq"
      }
    },
    "timestamp": "2024-02-28T13:19:19.578628712Z",
    "severity": "ERROR",
    "labels": {
      "k8s-pod/batch_kubernetes_io/controller-uid": "906ace45-2ab9-40cc-aa48-7d29b74a73b9",
      "k8s-pod/batch_kubernetes_io/job-name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq",
      "k8s-pod/controller-uid": "906ace45-2ab9-40cc-aa48-7d29b74a73b9",
      "compute.googleapis.com/resource_name": "gke-kubernetes-cluster-default-pool-96aa9230-hqq3",
      "k8s-pod/job-name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq"
    },
    "logName": "projects/cml-k8s-github-runner-mre/logs/stderr",
    "receiveTimestamp": "2024-02-28T13:19:21.719792035Z"
  },
  {
    "textPayload": "                                 Dload  Upload   Total   Spent    Left  Speed",
    "insertId": "ulqy1e6opa28bja2",
    "resource": {
      "type": "k8s_container",
      "labels": {
        "namespace_name": "default",
        "location": "europe-west1-b",
        "cluster_name": "kubernetes-cluster",
        "container_name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq",
        "project_id": "cml-k8s-github-runner-mre",
        "pod_name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq-89rbs"
      }
    },
    "timestamp": "2024-02-28T13:19:19.578667703Z",
    "severity": "ERROR",
    "labels": {
      "k8s-pod/job-name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq",
      "k8s-pod/batch_kubernetes_io/controller-uid": "906ace45-2ab9-40cc-aa48-7d29b74a73b9",
      "compute.googleapis.com/resource_name": "gke-kubernetes-cluster-default-pool-96aa9230-hqq3",
      "k8s-pod/controller-uid": "906ace45-2ab9-40cc-aa48-7d29b74a73b9",
      "k8s-pod/batch_kubernetes_io/job-name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq"
    },
    "logName": "projects/cml-k8s-github-runner-mre/logs/stderr",
    "receiveTimestamp": "2024-02-28T13:19:21.719792035Z"
  },
  {
    "textPayload": "\r  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0\r  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0\r  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0",
    "insertId": "kgvwjrhlczlqabe8",
    "resource": {
      "type": "k8s_container",
      "labels": {
        "namespace_name": "default",
        "cluster_name": "kubernetes-cluster",
        "pod_name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq-89rbs",
        "project_id": "cml-k8s-github-runner-mre",
        "location": "europe-west1-b",
        "container_name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq"
      }
    },
    "timestamp": "2024-02-28T13:19:19.782620933Z",
    "severity": "ERROR",
    "labels": {
      "k8s-pod/job-name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq",
      "k8s-pod/batch_kubernetes_io/job-name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq",
      "k8s-pod/batch_kubernetes_io/controller-uid": "906ace45-2ab9-40cc-aa48-7d29b74a73b9",
      "k8s-pod/controller-uid": "906ace45-2ab9-40cc-aa48-7d29b74a73b9",
      "compute.googleapis.com/resource_name": "gke-kubernetes-cluster-default-pool-96aa9230-hqq3"
    },
    "logName": "projects/cml-k8s-github-runner-mre/logs/stderr",
    "receiveTimestamp": "2024-02-28T13:19:21.719792035Z"
  },
  {
    "textPayload": "\r  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0",
    "insertId": "y99saqblnn9f3jwo",
    "resource": {
      "type": "k8s_container",
      "labels": {
        "project_id": "cml-k8s-github-runner-mre",
        "pod_name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq-89rbs",
        "location": "europe-west1-b",
        "namespace_name": "default",
        "cluster_name": "kubernetes-cluster",
        "container_name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq"
      }
    },
    "timestamp": "2024-02-28T13:19:19.929872836Z",
    "severity": "ERROR",
    "labels": {
      "k8s-pod/controller-uid": "906ace45-2ab9-40cc-aa48-7d29b74a73b9",
      "k8s-pod/batch_kubernetes_io/controller-uid": "906ace45-2ab9-40cc-aa48-7d29b74a73b9",
      "k8s-pod/batch_kubernetes_io/job-name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq",
      "k8s-pod/job-name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq",
      "compute.googleapis.com/resource_name": "gke-kubernetes-cluster-default-pool-96aa9230-hqq3"
    },
    "logName": "projects/cml-k8s-github-runner-mre/logs/stderr",
    "receiveTimestamp": "2024-02-28T13:19:21.719792035Z"
  },
  {
    "textPayload": "\r 76 84.5M   76 64.4M    0     0  56.5M      0  0:00:01  0:00:01 --:--:-- 56.5M\r100 84.5M  100 84.5M    0     0  67.7M      0  0:00:01  0:00:01 --:--:--  186M",
    "insertId": "xgshreh6zrjgh6v8",
    "resource": {
      "type": "k8s_container",
      "labels": {
        "namespace_name": "default",
        "cluster_name": "kubernetes-cluster",
        "project_id": "cml-k8s-github-runner-mre",
        "location": "europe-west1-b",
        "container_name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq",
        "pod_name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq-89rbs"
      }
    },
    "timestamp": "2024-02-28T13:19:20.824606312Z",
    "severity": "ERROR",
    "labels": {
      "k8s-pod/batch_kubernetes_io/controller-uid": "906ace45-2ab9-40cc-aa48-7d29b74a73b9",
      "k8s-pod/job-name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq",
      "k8s-pod/controller-uid": "906ace45-2ab9-40cc-aa48-7d29b74a73b9",
      "compute.googleapis.com/resource_name": "gke-kubernetes-cluster-default-pool-96aa9230-hqq3",
      "k8s-pod/batch_kubernetes_io/job-name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq"
    },
    "logName": "projects/cml-k8s-github-runner-mre/logs/stderr",
    "receiveTimestamp": "2024-02-28T13:19:21.719792035Z"
  },
  {
    "textPayload": "bash: line 23: lsof: command not found",
    "insertId": "lnpwdpwqueh4ssrq",
    "resource": {
      "type": "k8s_container",
      "labels": {
        "location": "europe-west1-b",
        "cluster_name": "kubernetes-cluster",
        "namespace_name": "default",
        "container_name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq",
        "pod_name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq-89rbs",
        "project_id": "cml-k8s-github-runner-mre"
      }
    },
    "timestamp": "2024-02-28T13:19:21.068149849Z",
    "severity": "ERROR",
    "labels": {
      "compute.googleapis.com/resource_name": "gke-kubernetes-cluster-default-pool-96aa9230-hqq3",
      "k8s-pod/job-name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq",
      "k8s-pod/batch_kubernetes_io/controller-uid": "906ace45-2ab9-40cc-aa48-7d29b74a73b9",
      "k8s-pod/controller-uid": "906ace45-2ab9-40cc-aa48-7d29b74a73b9",
      "k8s-pod/batch_kubernetes_io/job-name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq"
    },
    "logName": "projects/cml-k8s-github-runner-mre/logs/stderr",
    "receiveTimestamp": "2024-02-28T13:19:21.719792035Z"
  },
  {
    "insertId": "8v8vn8ka1u4ioylu",
    "jsonPayload": {
      "level": "info",
      "message": "POST /repos/swiss-ai-center/cml-kubernetes-github-actions-runner-minimal-reproducible-example/actions/runners/registration-token - 201 in 299ms"
    },
    "resource": {
      "type": "k8s_container",
      "labels": {
        "project_id": "cml-k8s-github-runner-mre",
        "container_name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq",
        "namespace_name": "default",
        "cluster_name": "kubernetes-cluster",
        "pod_name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq-89rbs",
        "location": "europe-west1-b"
      }
    },
    "timestamp": "2024-02-28T13:19:22.711544613Z",
    "severity": "INFO",
    "labels": {
      "compute.googleapis.com/resource_name": "gke-kubernetes-cluster-default-pool-96aa9230-hqq3",
      "k8s-pod/controller-uid": "906ace45-2ab9-40cc-aa48-7d29b74a73b9",
      "k8s-pod/batch_kubernetes_io/controller-uid": "906ace45-2ab9-40cc-aa48-7d29b74a73b9",
      "k8s-pod/batch_kubernetes_io/job-name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq",
      "k8s-pod/job-name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq"
    },
    "logName": "projects/cml-k8s-github-runner-mre/logs/stderr",
    "receiveTimestamp": "2024-02-28T13:19:26.760960936Z"
  },
  {
    "insertId": "4p1pvcut3v64yl5r",
    "jsonPayload": {
      "level": "info",
      "message": "GET /repos/swiss-ai-center/cml-kubernetes-github-actions-runner-minimal-reproducible-example/actions/runners?per_page=100 - 200 in 212ms"
    },
    "resource": {
      "type": "k8s_container",
      "labels": {
        "container_name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq",
        "location": "europe-west1-b",
        "namespace_name": "default",
        "project_id": "cml-k8s-github-runner-mre",
        "pod_name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq-89rbs",
        "cluster_name": "kubernetes-cluster"
      }
    },
    "timestamp": "2024-02-28T13:19:22.947404081Z",
    "severity": "INFO",
    "labels": {
      "compute.googleapis.com/resource_name": "gke-kubernetes-cluster-default-pool-96aa9230-hqq3",
      "k8s-pod/batch_kubernetes_io/job-name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq",
      "k8s-pod/batch_kubernetes_io/controller-uid": "906ace45-2ab9-40cc-aa48-7d29b74a73b9",
      "k8s-pod/controller-uid": "906ace45-2ab9-40cc-aa48-7d29b74a73b9",
      "k8s-pod/job-name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq"
    },
    "logName": "projects/cml-k8s-github-runner-mre/logs/stderr",
    "receiveTimestamp": "2024-02-28T13:19:26.760960936Z"
  },
  {
    "insertId": "jbk75zzcc5emlk5m",
    "jsonPayload": {
      "message": "Github Actions timeout has been updated from 72h to 35 days. Update your workflow accordingly to be able to restart it automatically.",
      "level": "warn"
    },
    "resource": {
      "type": "k8s_container",
      "labels": {
        "pod_name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq-89rbs",
        "cluster_name": "kubernetes-cluster",
        "project_id": "cml-k8s-github-runner-mre",
        "container_name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq",
        "namespace_name": "default",
        "location": "europe-west1-b"
      }
    },
    "timestamp": "2024-02-28T13:19:22.951601273Z",
    "severity": "WARNING",
    "labels": {
      "compute.googleapis.com/resource_name": "gke-kubernetes-cluster-default-pool-96aa9230-hqq3",
      "k8s-pod/batch_kubernetes_io/job-name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq",
      "k8s-pod/batch_kubernetes_io/controller-uid": "906ace45-2ab9-40cc-aa48-7d29b74a73b9",
      "k8s-pod/job-name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq",
      "k8s-pod/controller-uid": "906ace45-2ab9-40cc-aa48-7d29b74a73b9"
    },
    "logName": "projects/cml-k8s-github-runner-mre/logs/stderr",
    "receiveTimestamp": "2024-02-28T13:19:26.760960936Z"
  },
  {
    "insertId": "d9kcauaby3ys0dr9",
    "jsonPayload": {
      "message": "Preparing workdir /home/runner...",
      "level": "info"
    },
    "resource": {
      "type": "k8s_container",
      "labels": {
        "namespace_name": "default",
        "project_id": "cml-k8s-github-runner-mre",
        "cluster_name": "kubernetes-cluster",
        "container_name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq",
        "location": "europe-west1-b",
        "pod_name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq-89rbs"
      }
    },
    "timestamp": "2024-02-28T13:19:22.951758657Z",
    "severity": "INFO",
    "labels": {
      "compute.googleapis.com/resource_name": "gke-kubernetes-cluster-default-pool-96aa9230-hqq3",
      "k8s-pod/controller-uid": "906ace45-2ab9-40cc-aa48-7d29b74a73b9",
      "k8s-pod/batch_kubernetes_io/job-name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq",
      "k8s-pod/batch_kubernetes_io/controller-uid": "906ace45-2ab9-40cc-aa48-7d29b74a73b9",
      "k8s-pod/job-name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq"
    },
    "logName": "projects/cml-k8s-github-runner-mre/logs/stderr",
    "receiveTimestamp": "2024-02-28T13:19:26.760960936Z"
  },
  {
    "insertId": "unowq9pguft1apq8",
    "jsonPayload": {
      "message": "Launching github runner",
      "level": "info"
    },
    "resource": {
      "type": "k8s_container",
      "labels": {
        "project_id": "cml-k8s-github-runner-mre",
        "cluster_name": "kubernetes-cluster",
        "container_name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq",
        "namespace_name": "default",
        "location": "europe-west1-b",
        "pod_name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq-89rbs"
      }
    },
    "timestamp": "2024-02-28T13:19:22.953016273Z",
    "severity": "INFO",
    "labels": {
      "k8s-pod/job-name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq",
      "k8s-pod/batch_kubernetes_io/job-name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq",
      "k8s-pod/controller-uid": "906ace45-2ab9-40cc-aa48-7d29b74a73b9",
      "compute.googleapis.com/resource_name": "gke-kubernetes-cluster-default-pool-96aa9230-hqq3",
      "k8s-pod/batch_kubernetes_io/controller-uid": "906ace45-2ab9-40cc-aa48-7d29b74a73b9"
    },
    "logName": "projects/cml-k8s-github-runner-mre/logs/stderr",
    "receiveTimestamp": "2024-02-28T13:19:26.760960936Z"
  },
  {
    "insertId": "09zpx82g4peai6xh",
    "jsonPayload": {
      "level": "info",
      "message": "Terraform 1.6.2"
    },
    "resource": {
      "type": "k8s_container",
      "labels": {
        "container_name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq",
        "location": "europe-west1-b",
        "cluster_name": "kubernetes-cluster",
        "namespace_name": "default",
        "pod_name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq-89rbs",
        "project_id": "cml-k8s-github-runner-mre"
      }
    },
    "timestamp": "2024-02-28T13:19:29.123063153Z",
    "severity": "INFO",
    "labels": {
      "k8s-pod/batch_kubernetes_io/controller-uid": "906ace45-2ab9-40cc-aa48-7d29b74a73b9",
      "compute.googleapis.com/resource_name": "gke-kubernetes-cluster-default-pool-96aa9230-hqq3",
      "k8s-pod/job-name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq",
      "k8s-pod/controller-uid": "906ace45-2ab9-40cc-aa48-7d29b74a73b9",
      "k8s-pod/batch_kubernetes_io/job-name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq"
    },
    "logName": "projects/cml-k8s-github-runner-mre/logs/stderr",
    "receiveTimestamp": "2024-02-28T13:19:31.987843933Z"
  },
  {
    "insertId": "13xpw0for0ojxqyh",
    "jsonPayload": {
      "message": "Plan: 0 to add, 0 to change, 0 to destroy.",
      "level": "info"
    },
    "resource": {
      "type": "k8s_container",
      "labels": {
        "namespace_name": "default",
        "project_id": "cml-k8s-github-runner-mre",
        "cluster_name": "kubernetes-cluster",
        "pod_name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq-89rbs",
        "location": "europe-west1-b",
        "container_name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq"
      }
    },
    "timestamp": "2024-02-28T13:19:29.963315881Z",
    "severity": "INFO",
    "labels": {
      "k8s-pod/job-name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq",
      "k8s-pod/controller-uid": "906ace45-2ab9-40cc-aa48-7d29b74a73b9",
      "k8s-pod/batch_kubernetes_io/job-name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq",
      "compute.googleapis.com/resource_name": "gke-kubernetes-cluster-default-pool-96aa9230-hqq3",
      "k8s-pod/batch_kubernetes_io/controller-uid": "906ace45-2ab9-40cc-aa48-7d29b74a73b9"
    },
    "logName": "projects/cml-k8s-github-runner-mre/logs/stderr",
    "receiveTimestamp": "2024-02-28T13:19:31.987843933Z"
  },
  {
    "insertId": "6xcab9hbb3w8toyd",
    "jsonPayload": {
      "message": "Apply complete! Resources: 0 added, 0 changed, 0 destroyed.",
      "level": "info"
    },
    "resource": {
      "type": "k8s_container",
      "labels": {
        "cluster_name": "kubernetes-cluster",
        "location": "europe-west1-b",
        "pod_name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq-89rbs",
        "container_name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq",
        "namespace_name": "default",
        "project_id": "cml-k8s-github-runner-mre"
      }
    },
    "timestamp": "2024-02-28T13:19:29.986556552Z",
    "severity": "INFO",
    "labels": {
      "k8s-pod/controller-uid": "906ace45-2ab9-40cc-aa48-7d29b74a73b9",
      "k8s-pod/job-name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq",
      "compute.googleapis.com/resource_name": "gke-kubernetes-cluster-default-pool-96aa9230-hqq3",
      "k8s-pod/batch_kubernetes_io/controller-uid": "906ace45-2ab9-40cc-aa48-7d29b74a73b9",
      "k8s-pod/batch_kubernetes_io/job-name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq"
    },
    "logName": "projects/cml-k8s-github-runner-mre/logs/stderr",
    "receiveTimestamp": "2024-02-28T13:19:31.987843933Z"
  },
  {
    "insertId": "6ck697ogiyr4j393",
    "jsonPayload": {
      "level": "info",
      "message": "Outputs: 0"
    },
    "resource": {
      "type": "k8s_container",
      "labels": {
        "cluster_name": "kubernetes-cluster",
        "container_name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq",
        "location": "europe-west1-b",
        "pod_name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq-89rbs",
        "project_id": "cml-k8s-github-runner-mre",
        "namespace_name": "default"
      }
    },
    "timestamp": "2024-02-28T13:19:29.987518660Z",
    "severity": "INFO",
    "labels": {
      "k8s-pod/batch_kubernetes_io/job-name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq",
      "k8s-pod/controller-uid": "906ace45-2ab9-40cc-aa48-7d29b74a73b9",
      "compute.googleapis.com/resource_name": "gke-kubernetes-cluster-default-pool-96aa9230-hqq3",
      "k8s-pod/batch_kubernetes_io/controller-uid": "906ace45-2ab9-40cc-aa48-7d29b74a73b9",
      "k8s-pod/job-name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq"
    },
    "logName": "projects/cml-k8s-github-runner-mre/logs/stderr",
    "receiveTimestamp": "2024-02-28T13:19:31.987843933Z"
  },
  {
    "insertId": "wslg0hbfwv35bm2s",
    "jsonPayload": {
      "level": "warn",
      "message": "Error connecting to ACPI socket: connect ENOENT /var/run/acpid.socket. The acpid.service helps with instance termination detection."
    },
    "resource": {
      "type": "k8s_container",
      "labels": {
        "container_name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq",
        "project_id": "cml-k8s-github-runner-mre",
        "cluster_name": "kubernetes-cluster",
        "location": "europe-west1-b",
        "pod_name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq-89rbs",
        "namespace_name": "default"
      }
    },
    "timestamp": "2024-02-28T13:19:29.994530757Z",
    "severity": "WARNING",
    "labels": {
      "k8s-pod/batch_kubernetes_io/job-name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq",
      "k8s-pod/batch_kubernetes_io/controller-uid": "906ace45-2ab9-40cc-aa48-7d29b74a73b9",
      "k8s-pod/controller-uid": "906ace45-2ab9-40cc-aa48-7d29b74a73b9",
      "compute.googleapis.com/resource_name": "gke-kubernetes-cluster-default-pool-96aa9230-hqq3",
      "k8s-pod/job-name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq"
    },
    "logName": "projects/cml-k8s-github-runner-mre/logs/stderr",
    "receiveTimestamp": "2024-02-28T13:19:31.987843933Z"
  },
  {
    "insertId": "ndwqf5x0dhfbvf3r",
    "jsonPayload": {
      "level": "info",
      "message": "POST /repos/swiss-ai-center/cml-kubernetes-github-actions-runner-minimal-reproducible-example/actions/runners/registration-token - 201 in 284ms"
    },
    "resource": {
      "type": "k8s_container",
      "labels": {
        "cluster_name": "kubernetes-cluster",
        "container_name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq",
        "pod_name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq-89rbs",
        "location": "europe-west1-b",
        "namespace_name": "default",
        "project_id": "cml-k8s-github-runner-mre"
      }
    },
    "timestamp": "2024-02-28T13:19:52.618055666Z",
    "severity": "INFO",
    "labels": {
      "k8s-pod/batch_kubernetes_io/job-name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq",
      "k8s-pod/job-name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq",
      "compute.googleapis.com/resource_name": "gke-kubernetes-cluster-default-pool-96aa9230-hqq3",
      "k8s-pod/batch_kubernetes_io/controller-uid": "906ace45-2ab9-40cc-aa48-7d29b74a73b9",
      "k8s-pod/controller-uid": "906ace45-2ab9-40cc-aa48-7d29b74a73b9"
    },
    "logName": "projects/cml-k8s-github-runner-mre/logs/stderr",
    "receiveTimestamp": "2024-02-28T13:19:56.719671861Z"
  },
  {
    "insertId": "6nglkdoy3li3yuir",
    "jsonPayload": {
      "status": "ready",
      "date": "2024-02-28T13:19:58.116Z",
      "message": "runner status",
      "repo": "https://github.com/swiss-ai-center/cml-kubernetes-github-actions-runner-minimal-reproducible-example",
      "level": "info"
    },
    "resource": {
      "type": "k8s_container",
      "labels": {
        "pod_name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq-89rbs",
        "namespace_name": "default",
        "cluster_name": "kubernetes-cluster",
        "location": "europe-west1-b",
        "container_name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq",
        "project_id": "cml-k8s-github-runner-mre"
      }
    },
    "timestamp": "2024-02-28T13:19:58.116664776Z",
    "severity": "INFO",
    "labels": {
      "k8s-pod/batch_kubernetes_io/controller-uid": "906ace45-2ab9-40cc-aa48-7d29b74a73b9",
      "k8s-pod/job-name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq",
      "k8s-pod/controller-uid": "906ace45-2ab9-40cc-aa48-7d29b74a73b9",
      "compute.googleapis.com/resource_name": "gke-kubernetes-cluster-default-pool-96aa9230-hqq3",
      "k8s-pod/batch_kubernetes_io/job-name": "cml-4sy2libvak-3sz8fu4m-1r8a2vmq"
    },
    "logName": "projects/cml-k8s-github-runner-mre/logs/stderr",
    "receiveTimestamp": "2024-02-28T13:20:01.720083434Z"
  }
]
```

</details>

### Expected behavior

1. A Kubernetes pod is created
2. The pod runs a GitHub Actions runner
3. The runner executes the workflow
4. The workflow runs successfully

### Elements that were tested

Other then the elements mentioned in the related issue, the following elements
were tested:

- Open all ports in the GCP firewall - not recommended but it was to test the
  hypothesis that GitHub Actions was not able to communicate with the runner on
  the Kubernetes cluster as mentioned in the related issue
- Use a different runner image
- Use a different runner version
- Use a different runner configuration
- Give the personal access token all permissions
- Updated all GitHub Actions versions

### Possible workarounds/fixes

- Could the `lsof` missing command be the cause of the issue? - It does not seem
  to be the cause of the issue, as the runner is successfully sregistered with
  the repository.
- Could the Kubernetes cluster be misconfigured?
- Could the Kubernetes version be incompatible with GitHub Actions?

## Conclusion

We are currently unable to run GitHub Actions workflows on a self-hosted
Kubernetes runner. The runner is able to start, but the workflow fails to
execute. The issue is not reproducible on a self-hosted runner on a virtual
machine.

We are three people investigating the issue and are unable to find a solution.
We are currently unable to use GitHub Actions on Kubernetes with CML.

We have tried to find other solutions to replace CML but CML is the only
easy-to-use solution that we have found.

## Questions

As a result of the investigation, we have the following questions:

- How can we debug the issue further?
- Are we the only ones experiencing this issue?
- How can we help Iterative to resolve the issue?
- Are there any workarounds or fixes that we can apply?
- Should we try to use a different cloud provider?

We have available resources to help Iterative to resolve the issue and are
willing to contribute to the resolution of the issue.

## References

- https://github.com/iterative/cml/issues/1415
- https://cml.dev/doc/self-hosted-runners
- https://cml.dev/doc/ref/runner

## Additional resources

- https://github.com/actions/actions-runner-controller - does not allow to
  select a specific pod using Kubernetes node selectors to train the model on a
  specific node
