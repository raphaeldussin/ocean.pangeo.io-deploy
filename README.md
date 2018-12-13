# About

This repository contains the reproducible configuration for deploying a Pangeo
instance on Google Kubernetes Engine.

It contains scripts to automatically redeploy when the image definition or
chart parameters are changed.

# Before we start

## What you need on your local machine

* google cloud sdk: download the latest archive from [GOOGLE-SDK](https://cloud.google.com/sdk/docs/#install_the_latest_cloud_tools_version_cloudsdk_current_version) and install the sdk. Here I give the example for OSX, with bash shell and install in /opt. Adapt to your config.
```
tar -xf google-cloud-sdk-225.0.0-darwin-x86_64.tar
mv google-cloud-sdk /opt/google-cloud-sdk
cd /opt
./google-cloud-sdk/install.sh
cd $HOME ; . .bashrc ; cd /opt
./google-cloud-sdk/bin/gcloud init
```

* kubectl: install with your favourite package manager (apt, yum, macports,...)
* helm: get the binaries [HELM-Install](https://github.com/helm/helm/releases/tag/v2.11.0) section Intalling and updating and
copy into your PATH.

## Roles on Google Cloud

From the Google Cloud Console [GCE](https://console.cloud.google.com), make sure you have the following roles (else adjust
with owner):

* Kubernetes Engine Admin
* Security Reviewer
* Service Account Admin
* Compute Admin
* Service Account User
* Storage Admin
* Viewer

# Setup

The first step to using this automation is to create a Kubernetes cluster and
install the server-side component Tiller on it (used for deploying applications
with the Helm tool). Scripts to do so can be found here:
https://github.com/pangeo-data/pangeo/tree/master/gce/setup-guide

* Modify the script `gce-pangeo-environment.sh` with the parameters for your deployment
* Run the following scripts:
```
. gce-pangeo-environment.sh
./1_create_cluster.sh
./2_configure_kubernetes.sh
```

## Install git-crypt

You will need to install
[`git-crypt`](https://www.agwa.name/projects/git-crypt/). `git-crypt` is used
to encrypt the secrets that are used for deploying your cluster. Please read this [HOW GIT-CRYPT WORKS](https://www.agwa.name/projects/git-crypt/) if new to it.

# Configure this repository

Once you have a cluster created, you can begin customizing the configuration.

* Create a fork of this repository in GitHub. (Note: the default branch is staging)
* Copy the deployments/example.pangeo.io directory to your desired name
  * `cp -r example.pangeo.io newname.pangeo.io`
* Regenerate the git-crypt key. This will be used to encrypt the secrets
that are used for your deployment.
  * `git crypt init`
* Create a CircleCI job for the repo.
  * Log in or Sign up [here](https://circleci.com). Go to [dashboard](https://circleci.com/dashboard).
  * Click add project under your github account name.
  * Click Set Up Project for the example.pangeo.io-deploy repo. Click Linux. Click Start building at bottom.
  * You will need to add the below environmental variables to your CircleCI configuration:
* Create a Google service account
  * (For the pangeo project, reuse pangeo-automated-deploy service account)
  * Required IAM roles:
    * Kubernetes Engine Admin
    * Storage Admin

| Name | Description |
| ---- | ----------- |
| GCR_READWRITE_KEY | The JSON output of `gcloud iam service-accounts keys create` (ask Pangeo administrator to configure this) |
| GIT_CRYPT_KEY | The base64 encoded output of `git crypt export-key key.txt` Then run `base64 key.txt` to get pipe the key to standard output. Copy past into the env var. Delete `key.txt` afterwards!|
| GKE_CLUSTER | The name that your cluster was created with |
| GOOGLE_PROJECT | The identifier of the project where your cluster was created |
| GOOGLE_REGION | The Google compute region where your cluster is located (e.g. us-central1) |
| GOOGLE_ZONE | The Google compute zone where your cluster is located (e.g. us-central1-b) |
| IMAGE_NAME | The container registry image to build and use for your notebook and worker containers (e.g. us.gcr.io/pangeo-181919/example-pangeo-io-notebook). See [documentation](https://cloud.google.com/container-registry/) for setting up with your own project. Note: you need write permisson for this project. You will also have to give permisson (Storage Admin, Kubernetes Engine Admin, and Viewer) to the circleCI service account on the container registry ([see doc](https://cloud.google.com/container-registry/docs/access-control)). Then enable the registry API [here](https://console.cloud.google.com/flows/enableapi?apiid=containerregistry.googleapis.com&redirect=https://cloud.google.com/container-registry/docs/quickstart&_ga=2.12214260.-1113544925.1533776076)|
| DEPLOYMENT | The name of the directory in `deployments` you wish to deploy (e.g., example.pangeo.io) |

* Make a commit to test if the job succeeded on circleCI. If it failed troubleshoot it.

# Common Issues

* `Error: could not find a ready tiller pod` Some times it takes a while to download the new tiller image, wait and try again.
* `Error: UPGRADE FAILED: "example.pangeo.io-staging" has no deployed releases` If your first deploy of an application fails. Run `helm delete polar.pangeo.io-staging --purge` anywhere you have run `gcloud container clusters get-credentials`
