# tdk-on-gcp
Public repository for the configuration files, user guide, and other resources to run the Synthesized TDK on GCP



Install Docker rootless: https://docs.docker.com/engine/install/ubuntu

Remove preinstalled `gcloud` tool:
```
sudo snap remove google-cloud-cli
```

install gcloud from tar archive - https://cloud.google.com/sdk/docs/install#linux


Now you can switch gcloud to using the Service Account by creating and downloading a one-time key, and activate it:
```
gcloud iam service-accounts keys create ./marketplace-dev-robot-key.json \
  --iam-account marketplace-dev-robot@synthesized-marketplace-public.iam.gserviceaccount.com

gcloud auth activate-service-account \
  --key-file ./marketplace-dev-robot-key.json
```

Change the project:
```
gcloud config set project synthesized-marketplace-public
```

Install gke-gcloud-auth-plugin (see more in https://cloud.google.com/blog/products/containers-kubernetes/kubectl-auth-changes-in-gke):
```
gcloud components install gke-gcloud-auth-plugin
```


Create a new cluster from the command-line:
```
export CLUSTER=gramin-tdk-cluster
export ZONE=europe-west2-a

gcloud container clusters create "${CLUSTER}" --zone "${ZONE}"
```



```
docker pull gcr.io/cloud-marketplace-tools/k8s/dev

mkdir bin

BIN_FILE="$HOME/bin/mpdev"
docker run \
  gcr.io/cloud-marketplace-tools/k8s/dev \
  cat /scripts/dev > "$BIN_FILE"
chmod +x "$BIN_FILE"
```

Run mpdev and wait the "It works" message:
```
bin/mpdev
```


Install kubectl:
```
gcloud components install kubectl
```

Or:
```
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```


Install helm:
```
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```



```
export APP_INSTANCE_NAME=tdk-gramin
export NAMESPACE=default
export TAG="1.31.0"
export IMAGE_REGISTRY="gcr.io/synthesized-marketplace-public/synthesized-tdk-cli"
export SYNTHESIZED_KEY="test_key"
export RESOURCES_LIMITS_CPU=500m
export RESOURCES_LIMITS_MEMORY=1Gi
```

```
kubectl apply -f "https://raw.githubusercontent.com/GoogleCloudPlatform/marketplace-k8s-app-tools/master/crd/app-crd.yaml"
```


If you use a different namespace than the default, create a new namespace by running the following command:
```
kubectl create namespace "${NAMESPACE}"
```

### Creating the Service Account

To create the Service Account and ClusterRoleBinding:
```
export TDK_SERVICE_ACCOUNT="${APP_INSTANCE_NAME}-serviceaccount"
kubectl create serviceaccount "${TDK_SERVICE_ACCOUNT}" --namespace "${NAMESPACE}"
kubectl create clusterrole "${TDK_SERVICE_ACCOUNT}-role" --verb=get,list,watch --resource=services,nodes,pods,namespaces
kubectl create clusterrolebinding "${TDK_SERVICE_ACCOUNT}-rule" --clusterrole="${TDK_SERVICE_ACCOUNT}-role" --serviceaccount="${NAMESPACE}:${TDK_SERVICE_ACCOUNT}"
```



### Expanding the manifest template

```
helm template helm/synthesized-tdk-cli \
  --name-template "${APP_INSTANCE_NAME}" \
  --namespace "${NAMESPACE}" \
  --set envRenderSecret.SYNTHESIZED_KEY="${SYNTHESIZED_KEY}" \
  --set image.repository="${IMAGE_REGISTRY}" \
  --set image.tag="${TAG}" \
  --set resources.limits.cpu="${RESOURCES_LIMITS_CPU}" \
  --set resources.requests.cpu="${RESOURCES_LIMITS_CPU}" \
  --set resources.limits.memory="${RESOURCES_LIMITS_MEMORY}" \
  > "${APP_INSTANCE_NAME}_manifest.yaml"
```


### Applying the manifest to your Kubernetes cluster

```
kubectl apply -f "${APP_INSTANCE_NAME}_manifest.yaml" --namespace "${NAMESPACE}"
```


### Running the job

```
kubectl create job --from=cronjob.batch/synthesized-tdk-cli-cron my-tdk-gramin-synthesized-tdk-cron -n "${NAMESPACE}"
```

Get pods:
```
kubectl get pods -n ${NAMESPACE}
```

Get pod logs:
```
kubectl logs ${POD_NAME} -n ${NAMESPACE}
```


### Run GCP deployer

```
sudo chmod 666 /var/run/docker.sock
```

Install:
```shell
export SYNTHESIZED_KEY="AbGt3..."
export POSTGRES_PASSWORD="strong_postgres_password..."

mpdev install --deployer=gcr.io/synthesized-marketplace-public/synthesized-tdk-cli/deployer:1.31.0 --parameters='{"name": "synthesized-tdk-cli", "namespace": "default", "env.SYNTHESIZED_INPUT_URL": "jdbc:postgresql://10.91.48.3:5432/input_db", "env.SYNTHESIZED_OUTPUT_URL": "jdbc:postgresql://10.91.48.3:5432/output_db", "envRenderSecret.SYNTHESIZED_INPUT_USERNAME": "postgres", "envRenderSecret.SYNTHESIZED_INPUT_PASSWORD": "$POSTGRES_PASSWORD", "envRenderSecret.SYNTHESIZED_OUTPUT_USERNAME": "postgres", "envRenderSecret.SYNTHESIZED_OUTPUT_PASSWORD": "$POSTGRES_PASSWORD", "envRenderSecret.SYNTHESIZED_KEY": "$SYNTHESIZED_KEY"}'
```

Or verify:
```shell
mpdev verify --deployer=gcr.io/synthesized-marketplace-public/synthesized-tdk-cli/deployer:1.31.0
```

Delete kube resources:
```
kubectl delete pods --all -n default
kubectl delete jobs --all -n default
kubectl delete cronjobs --all -n default
```


### TODO

Add this logic to the `serviceAccount.yaml` file:
```shell
if Values.serviceAccount.name is not empty
  # use the predefined account
else if Values.serviceAccount.create == true
  # create service acccount
else
  fail "If you want us to create a service account, please set Values.serviceAccount.create = true"
```
