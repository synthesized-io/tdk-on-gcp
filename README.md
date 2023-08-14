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
```




```
helm template helm/synthesized-tdk \
  --name-template "${APP_INSTANCE_NAME}" \
  --namespace "${NAMESPACE}" \
  --set envRenderSecret.SYNTHESIZED_KEY="${SYNTHESIZED_KEY}" \
  --set image.repository="${IMAGE_REGISTRY}" \
  --set image.tag="${TAG}" \
  --set resources.limits.cpu="${RESOURCES_LIMITS_CPU}" \
  --set resources.limits.memory="${RESOURCES_LIMITS_MEMORY}" \
  > "${APP_INSTANCE_NAME}_manifest.yaml"
```


```
sudo chmod +x /var/run/docker.sock
```


```
mpdev install --deployer=gcr.io/synthesized-marketplace-public/sdk-jupyter-server/deployer:2.7.6  --parameters='{"name": "sdk-jupyter-server", "namespace": "default"}'

mpdev verify --deployer=gcr.io/synthesized-marketplace-public/sdk-jupyter-server/deployer:2.7.6
```
