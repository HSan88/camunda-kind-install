# Camunda Installation Tutorial from Zero to One Hundred
Deploy Camunda BPM on Kubernetes with Kind: A guide to setting up Camunda BPMS on a lightweight Kubernetes cluster using Kind, Helm, and Docker. Includes detailed steps, configuration files, and best practices for deploying, containerizing, and testing in a local Kubernetes environment.

## Prerequisites:
1. Familiarity with Linux
2. Familiarity with Docker
3. Familiarity with Helm
4. Familiarity with Kubernetes

Ubuntu 24.04 is used as the operating system, Docker is used for the Container Manager, and Kind is used for Kubernetes.
Important note: If you are installing Camunda from inside Iran, you must keep in mind that due to sanctions, it is not possible to receive Docker images and a series of packages, and you must use a VPN and also disable IP version 6 from within the VPN settings, otherwise you will still be identified.
## Install docker:
source: https://docs.docker.com/engine/install/ubuntu/


### Add Docker's official GPG key:
```
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```
### Add the repository to Apt sources:
```
echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
$(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \

sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
```
### Install docker
```
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-build-plugin docker-compose-plugin
```

# install helm:
source: https://helm.sh/docs/intro/install/#from-script
```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

# install kind:
source: https://kind.sigs.k8s.io/docs/user/quick-start#installing-from-release-binaries
```
[ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.25.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```
# Install kubectl:
source: https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/
```
curl -LO https://dl.k8s.io/release/v1.30.0/bin/linux/amd64/kubectl
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

# Install Camunda
Note: During the Camunda installation, there are commands that require you to wait after executing them until the necessary tasks are completed or, if they require downloading, the download is complete. To prevent this from happening again (waiting), we mark the commands that require waiting with a ⏳ sign. Usually these commands create pods that need to be in READY state to continue. To check that the pods are running and ready to use, run the its next following command.

```
sudo kubectl get pods -A
```
Run each of the following commands one by one
Creating a cluster for Camunda
1. ⏳
```
sudo kind create cluster -n camunda --image='kindest/node:v1.30.0' --config kind.config.yaml
sudo kubectl get pods -A
```
After run `sudo kubectl get pods -A` you see 1/1 in the READY column for all pods, it means that the pods are fully running and ready to use, you can now continue with the installation.

2. ⏳ After running each commands (one-by-one), you need wait and check pods state with `sudo kubectl get pods -A` command until the READY state of pods get 1/1.
```
sudo kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
sudo kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml
sudo kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml
```
3. You don't have to wait after running following commands.
```
sudo kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
helm repo add camunda https://helm.camunda.io
helm repo update
```

4. At this stage, you need to change the values ​​in the secret.yaml file because the access keys to Camunda services are stored in this file. One thing you need to pay attention to is that the values ​​are stored in base64, which means that first you need to convert your desired keys to base64 and then replace the default value in this file.
If you do not do this, the key to use Camunda services will be the same as the default value.
Default key: default-api-key
After changing this file, run the following command
```
kubectl apply -f secret.yaml
```

4. To access Camunda services, we need to use ingress-nginx and we also need to use the https protocol. To do this, run the following commands,
   Keep in mind that you will access Camunda services with `camunda.local` domain, you can use `zeebe.camunda.local` domain to access zeebe.

```
openssl req -x509 -newkey rsa:4096 -sha256 -days 3650 \
-nodes -keyout zeebe.key -out zeebe.crt -subj "/CN=zeebe.camunda.local" \
-addext "subjectAltName=DNS:zeebe.camunda.local"

openssl req -x509 -newkey rsa:4096 -sha256 -days 3650 \
-nodes -keyout camunda.key -out camunda.crt -subj "/CN=camunda.local" \
-addext "subjectAltName=DNS:camunda.local"

kubectl create secret tls tls-secret --cert=camunda.crt --key=camunda.key

kubectl create secret tls tls-secret-zeebe --cert=zeebe.crt --key=zeebe.key
```

This is the main installation step, and depending on your internet speed it will take a while. Be sure to run the `sudo kubectl get pods` command to see if the work is complete and all Camunda pods should be in READEY. Note that if your VPN is not working properly, you may have trouble downloading the images.

```
helm install camunda camunda/camunda-platform -f values-8.6.2.yaml
```
After the READY status of all pods changes to 1/1, run the following commands:
Keep in mind you must change *CAMUNDA_IP* to your machine IP
```
kubectl apply -f zeebe-gateway.yaml
kubectl patch svc camunda-zeebe-gateway-lb -n default -p '{"spec": {"type": "LoadBalancer", "externalIPs":["CAMUNDA_IP"]}}'
```
based on this link https://kind.sigs.k8s.io/docs/user/known-issues/#pod-errors-due-to-too-many-open-files run following commands:

```
sudo sysctl fs.inotify.max_user_watches=2097152
sudo sysctl fs.inotify.max_user_instances=2048

sudo echo "fs.inotify.max_user_watches = 2097152" >> /etc/sysctl.conf
sudo echo "fs.inotify.max_user_instances = 2048" >> /etc/sysctl.conf
```
