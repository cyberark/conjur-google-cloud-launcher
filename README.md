# Overview

CyberArk Conjur automatically secures secrets used by privileged users and machine identities.

[Learn more.](https://www.conjur.org)

# Installation

## Quick install with Google Cloud Marketplace

Get up and running with a few clicks!
Install this Conjur app to a Google Kubernetes Engine cluster using Google Cloud Marketplace.
Follow the [on-screen instructions](https://console.cloud.google.com/marketplace/details/cyberark/conjur-open-source).

## Command line instructions

### Prerequisites

#### Set up command-line tools

You'll need the following tools in your development environment:
- [gcloud](https://cloud.google.com/sdk/gcloud/)
- [kubectl](https://kubernetes.io/docs/reference/kubectl/overview/)
- [helm](https://github.com/helm/helm)
- [docker](https://docs.docker.com/install/)
- [git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)

Configure `gcloud` as a Docker credential helper:

```shell
gcloud auth configure-docker
```

#### Create a Google Kubernetes Engine cluster

Create a new cluster from the command line:

```shell
export CLUSTER=conjur-cluster
export ZONE=us-central1-a

gcloud container clusters create "$CLUSTER" --zone "$ZONE"
```

Configure `kubectl` to connect to the new cluster:

```shell
gcloud container clusters get-credentials "$CLUSTER" --zone "$ZONE"
```

#### Clone this repo

Clone this repo and the associated tools repo:

```shell
git submodule sync --recursive
git submodule update --recursive --init --force
```

#### Install the Application resource definition

An Application resource is a collection of individual Kubernetes components,
such as Services, Deployments, and so on, that you can manage as a group.

To set up your cluster to understand Application resources, run the following command:

```shell
kubectl apply -f marketplace-k8s-app-tools/crd/*
```

You need to run this command once.

The Application resource is defined by the
[Kubernetes SIG-apps](https://github.com/kubernetes/community/tree/master/sig-apps)
community. The source code can be found on
[github.com/kubernetes-sigs/application](https://github.com/kubernetes-sigs/application).

### Install the Application

#### Configure the app with environment variables

Choose the instance name and namespace for the app.

```shell
export APP_INSTANCE_NAME=conjur-1
export NAMESPACE=conjur
```

Configure the container images:

```shell
export TAG_VERSION=1.4.0
export IMAGE_CONJUR="gcr.io/cloud-marketplace/cyberark/conjur-open-source:$TAG_VERSION"
export IMAGE_POSTGRES="gcr.io/cloud-marketplace/cyberark/conjur-open-source/postgres:$TAG_VERSION"
export IMAGE_NGINX="gcr.io/cloud-marketplace/cyberark/conjur-open-source/nginx:$TAG_VERSION"
```

The images above are referenced by
[tag](https://docs.docker.com/engine/reference/commandline/tag). We recommend
that you pin each image to an immutable
[content digest](https://docs.docker.com/registry/spec/api/#content-digests).
This ensures that the installed application always uses the same images,
until you are ready to upgrade. To get the digest for the image, use the
following script:

```shell
for i in "IMAGE_CONJUR" "IMAGE_POSTGRES" "IMAGE_NGINX"; do
  repo=$(echo ${!i} | cut -d: -f1);
  digest=$(docker pull ${!i} | sed -n -e 's/Digest: //p');
  export $i="$repo@$digest";
  env | grep $i;
done
```

#### Create namespace in your Kubernetes cluster

We recommend running Conjur in its own namespace.
If you use a different namespace than the `default`, run the command below to create a new namespace:

```shell
kubectl create namespace "$NAMESPACE"
kubectl config set-context --current --namespace="$NAMESPACE"
```

#### Install the application with Helm (v2) to your Kubernetes cluster

These instructions assume that your local `helm` client is version 2.

This project uses the upstream [cyberark/conjur-oss Helm chart](https://github.com/cyberark/conjur-oss-helm-chart). (You do not need to clone or helm install this repo directly; this will be done indirectly via the helm install of conjur below.)

Use `helm` to deploy the application to your Kubernetes cluster:

If you'd like to use an external database, 
use the `helm` argument `--set conjuross.databaseUrl='postgresql://[user[:password]@][netloc][:port][/dbname][?param1=value1&...]'` below. If `conjuross.databaseUrl` is not specified, a postgres deployment and service are created.
See [conjur/values.yaml](conjur/values.yaml) for all available parameters and their defaults.
See [conjur-oss/values.yaml](https://github.com/cyberark/conjur-oss-helm-chart/blob/master/conjur-oss/values.yaml)
for all available upstream Helm chart parameters and their defaults.

```shell
helm dependency update ./conjur
helm install conjur --set conjur-oss.dataKey="$(docker run --rm cyberark/conjur data-key generate)" ./conjur
```

#### View the app in the Google Cloud Console

To get the Console URL for your app, run the following command:

```shell
echo "https://console.cloud.google.com/kubernetes/application/${ZONE}/${CLUSTER}/${NAMESPACE}/${APP_INSTANCE_NAME}"
```

To view the app, open the URL in your browser.

### Set up Conjur

To initialize Conjur, an account must be created. This is done by executing a command on a Conjur pod. This only needs to be done when launching a new Conjur application, or creating a new Conjur account.

```
# Find conjur pod
$ kubectl get po -l app=$name-conjur
conjur-d76f44b64-rkxff   1/1       Running   0          16m

# Create a Conjur account
$ kubectl exec conjur-d76f44b64-rkxff -c conjur-oss conjurctl account create quick-start
Created new account account 'quick-start'
Token-Signing Public Key: -----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAv9+iXUFZISHlZfGrkio5
RG4hPijORLLMwNlHx7Bv0NMG2gFpwS25xDmBK2UgARlow+b9OS9spmLAdH15lXP5
40HBkKX/aYnLCH55TC4T/GslyUiaLZocVn1nkExdnBYO+tbNjUUJnHp2tljfQKCD
Lp+uqDlwbPDzUEqxPRj70ro0F8aeCVvjb0AQ7dHMnZ/FYYpJShxOwtn/FeaC+mhU
85QlcrtN3kn+pAA9wxuoKpIN8DoCK506AftL5Dra9xdconneSZ4XhaweAzut0BLp
Oi5nZFPVNXAUBJAg/RmmgO2C3J8zBS66wo3L7XAhJs/TXzhKYxHQKNa6GoWgc7uW
RwIDAQAB
-----END PUBLIC KEY-----
API key for admin: r41crc30ys1hv3njzrvz30xq6k210ec4y61y5qmmj1xyb300btrxmer
```

> Note that the `conjurctl account create` command gives you the public key and admin API key for the account you created. Back them up in a safe location.

Run `kubectl get svc "$APP_INSTANCE_NAME-conjur"` until the `EXTERNAL-IP` column resolves.

### Connect remote with the Conjur CLI

Now pull the latest [cyberark/conjur-cli:5 image](https://hub.docker.com/r/cyberark/conjur-cli/) and run it to connect to Conjur.

> Note that you must use the DNS name you used in your deployment to connect to Conjur instead of the IP or you will get errors trying to log in.

```
$ docker pull cyberark/conjur-cli:5
$ docker run --rm -it --entrypoint bash cyberark/conjur-cli:5

# conjur init -u [EXTERNAL_IP] -a default # or whatever account you created
# conjur authn login -u admin
Please enter admin's password (it will not be echoed):
Logged in

# conjur authn whoami
{"account":"default","username":"admin"}
```

### Next steps

- Go through the [Conjur Tutorials](https://www.conjur.org/tutorials/)
- View Conjur’s [API Documentation](https://www.conjur.org/api.html)

# Scaling

This is a single-instance version of Conjur.
It is not intended to be scaled up with the current configuration.

# Upgrade the Application

## Prepare the environment

If you are using a remote database, no changes are needed.

## Upgrade Conjur

Set the new image version in an environment variable:

```shell
export IMAGE_CONJUR=gcr.io/cloud-marketplace/cyberark/conjur-open-source:1.0
```

Update the Deployment definition with the reference to the new image:

```shell
kubectl patch deployment $APP_INSTANCE_NAME-conjur \
  --namespace $NAMESPACE \
  --type='json' \
  --patch="[{ \
      \"op\": \"replace\", \
      \"path\": \"/spec/template/spec/containers/0/image\", \
      \"value\":\"${IMAGE_CONJUR}\" \
    }]"
```

Monitor the process with:

```shell
kubectl get pods -l "app.kubernetes.io/name=$APP_INSTANCE_NAME" \
  --output go-template='Status={{.status.phase}} Image={{(index .spec.containers 0).image}}' \
  --watch
```

The Pod is terminated, and recreated with a new image for the `conjur`
container. After the update is complete, the final state of
the Pod is `Running`, and marked as 1/1 in the `READY` column.

# Uninstall the Application

## Using the Google Cloud Platform Console

1. In the GCP Console, open [Kubernetes Applications](https://console.cloud.google.com/kubernetes/application).

1. From the list of applications, click **Conjur by CyberArk**.

1. On the Application Details page, click **Delete**.

## Using the command line

Delete the application release using Helm:

```sh-session
# Find the release
$ helm list | grep conjur

conjur	conjur   	1       	2020-03-09 15:36:14.293351857 -0400 EDT	deployed	conjur-1.3.7

# Delete the release
$ helm delete conjur
release "conjur" uninstalled
```

## License

This repository is licensed under Apache License 2.0 - see [`LICENSE`](LICENSE) for more details.
