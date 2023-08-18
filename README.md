# Overview

Public repository for the configuration files, user guide, and other resources to run the Synthesized TDK on GCP

[What is Synthesized TDK?](https://docs.synthesized.io/tdk/latest/user_guide/getting_started/what_is_tdk)

Available on the GCP Cloud Marketplace: https://console.cloud.google.com/marketplace/product/synthesized-marketplace-public/synthesized-tdk

# Installation

After installation is done, the TDK CronJob should be created.

## Prerequisites
### Databases
Please check the list of supported database dialects: https://docs.synthesized.io/tdk/latest/#_whats_included.

It is assumed that there are two databases: input and output.
You must know the [JDBC URL](https://en.wikipedia.org/wiki/Java_Database_Connectivity), username and password for each.

### Synthesized config
The Synthesized YAML configuration should be provided. 

This can be: 
* [MASKING](https://docs.synthesized.io/tdk/latest/user_guide/tutorial/masking)
* [GENERATION](https://docs.synthesized.io/tdk/latest/user_guide/tutorial/generation)
* [KEEP](https://docs.synthesized.io/tdk/latest/user_guide/tutorial/subsetting)

For example, the following config can be used:
```yaml
default_config:
  mode: MASKING
safety_mode: RELAXED
```

## Quick install with Google Cloud Marketplace

To install Synthesized TDK to a Google Kubernetes Engine cluster via Google Cloud Marketplace, follow the
[on-screen instructions](https://console.cloud.google.com/marketplace/product/synthesized-marketplace-public/synthesized-tdk).

## Command-line instructions

> **_NOTE:_**  The CLI installation is only available if you have successfully deployed TDK from the marketplace 
> and reporting service key was generated.

### Prerequisites

#### Setting up command-line tools

You need the following tools in your development environment:

- [gcloud](https://cloud.google.com/sdk/gcloud/)
- [kubectl](https://kubernetes.io/docs/reference/kubectl/overview/)
- [Docker](https://docs.docker.com/install/)
- [Git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)
- [Helm](https://helm.sh/)

Configure `gcloud` as a Docker credential helper:

```shell
gcloud auth configure-docker
```

#### Creating a Google Kubernetes Engine (GKE) cluster

Create a new cluster from the command line. Please note that the BigQuery scope is required.
You can change values of the properties CLUSTER and ZONE.

```shell
export CLUSTER=tdk-cluster
export ZONE=us-west1-a

gcloud container clusters create "${CLUSTER}" --zone "${ZONE}"
```

Configure `kubectl` to connect to the new cluster:

```shell
gcloud container clusters get-credentials "${CLUSTER}" --zone "${ZONE}"
```

#### Cloning this repo

Clone this repo, as well as its associated tools repo:

```shell
git clone --recursive https://github.com/synthesized-io/tdk-on-gcp.git
```

#### Installing the Application resource definition

An Application resource is a collection of individual Kubernetes
components, such as Services, Deployments, and so on, that you can
manage as a group.

To set up your cluster to understand Application resources, run the
following command:

```shell
kubectl apply -f "https://raw.githubusercontent.com/GoogleCloudPlatform/marketplace-k8s-app-tools/master/crd/app-crd.yaml"
```

You need to run this command once.

The Application resource is defined by the
[Kubernetes SIG-apps](https://github.com/kubernetes/community/tree/master/sig-apps)
community. You can find the source code at
[github.com/kubernetes-sigs/application](https://github.com/kubernetes-sigs/application).

### Installing the app

#### Configuring the app with environment variables

Choose an instance name and
[namespace](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)
for the app.

```shell
export APP_INSTANCE_NAME=synthesized-tdk
export NAMESPACE=synthesized-tdk
```

Set up the image tag.
Example:

```shell
export TAG="1.32.0"
```

Configure the container images:

```shell
export IMAGE_REGISTRY="gcr.io/synthesized-marketplace-public/synthesized-tdk"
```

(Optional) Set computation resources limit:

```shell
export RESOURCES_LIMITS_CPU=1
export RESOURCES_LIMITS_MEMORY=1Gi
```

If you use a different namespace than the `default`, create a new namespace by running the following command:

```shell
kubectl create namespace "${NAMESPACE}"
```

#### Configuring database URLs and credentials

Please replace the values and run:
```shell
export SYNTHESIZED_INPUT_URL={YOUR_INPUT_JDBC_URL}
export SYNTHESIZED_INPUT_USERNAME={YOUR_INPUT_USERNAME}
export SYNTHESIZED_INPUT_PASSWORD={YOUR_INPUT_PASSWORD}
export SYNTHESIZED_OUTPUT_URL={YOUR_OUTPUT_JDBC_URL}
export SYNTHESIZED_OUTPUT_USERNAME={YOUR_OUTPUT_USERNAME}
export SYNTHESIZED_OUTPUT_PASSWORD={YOUR_OUTPUT_PASSWORD}
```

#### Creating Synthesized transformation configuration

Create configuration file:
```shell
touch synthesized_config.yaml
```

Fill the `synthesized_config.yaml` with [MASKING](https://docs.synthesized.io/tdk/latest/user_guide/tutorial/masking), [GENERATION](https://docs.synthesized.io/tdk/latest/user_guide/tutorial/generation) or [KEEP](https://docs.synthesized.io/tdk/latest/user_guide/tutorial/subsetting) config, e.g:

```yaml
default_config:
  mode: MASKING
safety_mode: RELAXED
```

#### Configure schedule

(Optional) Set schedule when calling TDK. You can use the following value to disable scheduled startup.
```shell
export SCHEDULE="* * 31 2 *"
```

#### Expanding the manifest template

Use `helm template` to expand the template. We recommend that you save the
expanded manifest file for future updates to your app.

```shell
helm template chart/synthesized-tdk-cli \
  --name-template "${APP_INSTANCE_NAME}" \
  --namespace "${NAMESPACE}" \
  --set image.repository="${IMAGE_REGISTRY}" \
  --set image.tag="${TAG}" \
  --set schedule="${SCHEDULE}" \
  --set resources.limits.cpu="${RESOURCES_LIMITS_CPU}" \
  --set resources.limits.memory="${RESOURCES_LIMITS_MEMORY}" \
  --set-file env.SYNTHESIZED_USERCONFIG="synthesized_config.yaml" \
  --set envRenderSecret.SYNTHESIZED_INPUT_URL="${SYNTHESIZED_INPUT_URL}" \
  --set envRenderSecret.SYNTHESIZED_INPUT_USERNAME="${SYNTHESIZED_INPUT_USERNAME}" \
  --set envRenderSecret.SYNTHESIZED_INPUT_PASSWORD="${SYNTHESIZED_INPUT_PASSWORD}" \
  --set envRenderSecret.SYNTHESIZED_OUTPUT_URL="${SYNTHESIZED_OUTPUT_URL}" \
  --set envRenderSecret.SYNTHESIZED_OUTPUT_USERNAME="${SYNTHESIZED_OUTPUT_USERNAME}" \
  --set envRenderSecret.SYNTHESIZED_OUTPUT_PASSWORD="${SYNTHESIZED_OUTPUT_PASSWORD}" \
  --set reportingSecret="${APP_INSTANCE_NAME}-reporting-secret" \
  > "${APP_INSTANCE_NAME}_manifest.yaml"
```

#### Applying the manifest to your Kubernetes cluster

To apply the manifest to your Kubernetes cluster, use `kubectl`:

```shell
kubectl apply -f "${APP_INSTANCE_NAME}_manifest.yaml" --namespace "${NAMESPACE}"
```

#### Viewing your app in the Google Cloud Console

To get the Cloud Console URL for your app, run the following command:

```shell
echo "https://console.cloud.google.com/kubernetes/application/${ZONE}/${CLUSTER}/${NAMESPACE}/${APP_INSTANCE_NAME}"
```

To view the app, open the URL in your browser.

# Using the app

## How to use TDK

After Synthesized TDK is Installed you can either run the job manually or wait when cronjob is triggered by schedule. 

To trigger the cronjob manually run:
```shell
kubectl create job --from=cronjob/${APP_INSTANCE_NAME}-cron ${APP_INSTANCE_NAME}-cron -n ${NAMESPACE}
``` 

To see logs for the job run (use "JOB NAME" from the previous step):
```shell
kubectl logs -f jobs/{JOB NAME} -n ${NAMESPACE}
```

# App metrics

At the moment, the application does not support exporting Prometheus metrics and does not have any exporter.

# Uninstalling the app

## Using the Google Cloud Console

1.  In the Cloud Console, open
    [Kubernetes Applications](https://console.cloud.google.com/kubernetes/application).

2.  From the list of apps, choose your app installation.

3.  On the **Application Details** page, click **Delete**.

## Using the command-line

### Preparing your environment

Set your installation name and Kubernetes namespace:

```shell
export APP_INSTANCE_NAME=synthesized-tdk
export NAMESPACE=default
```

### Deleting your resources

> **NOTE:** We recommend using a `kubectl` version that is the same as the
> version of your cluster. Using the same version for `kubectl` and the cluster
> helps to avoid unforeseen issues.

#### Deleting the deployment with the generated manifest file

Run `kubectl` on the expanded manifest file:

```shell
kubectl delete -f ${APP_INSTANCE_NAME}_manifest.yaml --namespace ${NAMESPACE}
```

#### Deleting the deployment by deleting the Application resource

If you don't have the expanded manifest file, delete the resources by using
types and a label:

```shell
kubectl delete application,secret,cronjob,job \
  --namespace ${NAMESPACE} \
  --selector name=${APP_INSTANCE_NAME}
```
