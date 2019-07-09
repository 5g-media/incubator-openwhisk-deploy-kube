# Nvidia-GPU Node

Start with preparing your GPU host(s) to be installed with needed drivers so that they can function as Kubernetes GPU node(s).

You need to install nvidia drivers and nvidia-docker in each GPU node you have.

Let's start with a **fresh** Ubuntu 16.04 system, 4 CPU cores, 16 GB RAM, more then 100 GB disk and GeoForce (`TBD`) GPU.

Upgrade its latest version:

```
sudo apt-get update
sudo apt-get upgrade
```

### Nvidia cuda drivers

Invoke the below commands (taken from) [Cuda-toolkit 9.2 Installation](https://developer.nvidia.com/cuda-92-download-archive?target_os=Linux&target_arch=x86_64&target_distro=Ubuntu&target_version=1604&target_type=debnetwork):

```
sudo dpkg -i cuda-repo-ubuntu1604_9.2.148-1_amd64.deb
sudo apt-key adv --fetch-keys http://developer.download.nvidia.com/compute/cuda/repos/ubuntu1604/x86_64/7fa2af80.pub
sudo apt-get update
sudo apt-get install cuda
```

Then, reboot your host.

Verify cuda driver installation:
```
nvidia-smi
```

### Docker-CE 18.06.0

On your GPU nodes, install this specific version of [docker](https://docs.docker.com/install/linux/docker-ce/ubuntu/) verified to be working with Nvidia, Kubernetes and Minikube.

The below commands are taken from the docker installation guide.

```
sudo apt-get update
```

Install some utilities
```
sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common
```

Add the key and ensure its fingerprint
```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo apt-key fingerprint 0EBFCD88
```

Register docker repository
```
sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
sudo apt-get update
```

Install docker 18.06.0
```
sudo apt-get install docker-ce=18.06.0~ce~3-0~ubuntu containerd.io
```

Verify docker
```
sudo docker run hello-world
```

### nvidia-docker runtime

Your next step would be to install nvidia docker runtime. It should be matched with the docker version you installed. Refer to [nvidia-docker]([https://github.com/NVIDIA/nvidia-docker) for more information.

In short, follow the below instructions

Add the package repositories:
```
curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | \
  sudo apt-key add -
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | \
  sudo tee /etc/apt/sources.list.d/nvidia-docker.list
sudo apt-get update
```

Install the matched version. Command taken from [(here](https://github.com/NVIDIA/nvidia-docker/wiki/Frequently-Asked-Questions#how-do-i-install-20-if-im-not-using-the-latest-docker-version)
```
sudo apt-get install -y nvidia-docker2=2.0.3+docker18.06.0-1 nvidia-container-runtime=2.0.0+docker18.06.0-1
```

Restart the daemon
```
sudo pkill -SIGHUP dockerd
```

Test nvidia-smi with the latest official CUDA image
```
sudo docker run --runtime=nvidia --rm nvidia/cuda:9.0-base nvidia-smi
```

You will need to enable the nvidia runtime as your default runtime on your node. We will be editing the docker daemon config file which
is usually present at /etc/docker/daemon.json

```
{
    "default-runtime": "nvidia",
    "runtimes": {
        "nvidia": {
            "path": "/usr/bin/nvidia-container-runtime",
            "runtimeArgs": []
        }
    }
}
```

Restart docker
```
sudo service docker stop
sudo service docker start
```

# Minikube

Now that we have GPU host ready, its time to install Minikube.

Log into your GPU node.

### Install Kubectl

Start with installing kubectl

(commands taken from [install-kubectl-on-linux](https://kubernetes.io/docs/tasks/tools/install-kubectl/#install-kubectl-on-linux))

```
curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
```

Verify it
```
sudo kubectl version
```

### Install Minikube

(commands taken from [install-minikube-via-direct-download](https://kubernetes.io/docs/tasks/tools/install-minikube/#install-minikube-via-direct-download))

```
curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 \
  && chmod +x minikube
sudo install minikube /usr/local/bin
```

**Note: You may need to run kubectl/minikube/helm commands under sudo if you have not configured them to run under your own user**

### Install dependency

```
sudo apt-get install socat
```

### Start Minikube

```
sudo minikube start --vm-driver=none --memory=8192
```

The command may take a few minutes to complete..

After it starts, invoke:

```
sudo ip link set docker0 promisc on
```

### Verify Minikube

Check status and wait for all pods to become `Running` or `Completed`
```
minikube status
get pod --all-namespaces
```

### Install Nvidia k8s plugin

(command taken from [enabling-gpu-support-in-kubernetes](https://github.com/NVIDIA/k8s-device-plugin#enabling-gpu-support-in-kubernetes))

```
kubectl create -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/1.0.0-beta/nvidia-device-plugin.yml
```

### Verify pod can conume GPU

Create the following pod definition yaml
```
cat <<EOF > pod-gpu.yaml
apiVersion: v1
kind: Pod
metadata:
  name: cuda-vector-add
spec:
  restartPolicy: OnFailure
  containers:
    - name: cuda-vector-add
      image: "k8s.gcr.io/cuda-vector-add:v0.1"
      resources:
        limits:
          nvidia.com/gpu: 1 # requesting 1 GPU
EOF
```

Create the POD
```
kubectl create -f pod-gpu.yaml
```

Wait for it to enter `Completed` state and verify its logs

```
kubectl logs cuda-vector-add
```

### Special notes

* You should run the following command after each time you start minikube 
  ```
  sudo ip link set docker0 promisc on
  ```

* To delete minukbe and all of its resources
  ```
  minikube stop
  minikube delete
  ```

# OpenWhisk

You should deploy OpenWhisk control plane using the forked [incubator-openwhisk-deploy-kube](https://github.com/5g-media/incubator-openwhisk-deploy-kube/tree/gpu) which already supports invoking actions on GPU Kubernetes nodes.

In short, follow the below instructions

### mycluster.yaml

Create the fllowing file under `~/mycluster.yaml`. Replace ingress.apiHostName with output of `minikube ip` command

More information on mycluster.yaml can be found [here](https://github.com/5g-media/incubator-openwhisk-deploy-kube/blob/gpu/docs/configurationChoices.md)

```
ingress:
  type: NodePort
  apiHostName: <MINIKUBE IP>
  apiHostPort: 31001
whisk:
  limits:
    actions:
      memory:
        max: "2048m"
nginx:
  httpsNodePort: 31001
```

### Helm

Download [helm](https://github.com/helm/helm/releases/tag/v2.14.1) and extract it to ~

Init helm causing tiller pod to get created

```
~/linux-amd64/helm init
```

When tiller pod is in `Running` state, grant necessary privileges to helm:

```
kubectl create clusterrolebinding tiller-cluster-admin --clusterrole=cluster-admin --serviceaccount=kube-system:default
```

### Actual OW install

Label the node

```
kubectl label nodes --all openwhisk-role=invoker
```

Clone the repository and run helm to install OW

```
cd ~
git clone https://github.com/5g-media/incubator-openwhisk-deploy-kube.git
cd incubator-openwhisk-deploy-kube
git checkout gpu
~/linux-amd64/helm install ./helm/openwhisk --namespace=openwhisk --name=owdev -f ~/mycluster.yaml
```

Installation can take a few minutes..

Wait for invoker-health pod to get created and run
```
kubectl get pods -n openwhisk | grep invokerhealthtestaction
```

### Pull large GPU runtime images

GPU runtime images are large. When GPU action is being invoked for the first time, OpenWhisk invoker pulls them out from docker hub. Depending on your
network condition this can take fairly amount of time and causes timeouts for the action invocation. In oder to avoid this, pull the following images
in all GPU kubernetes nodes

```
sudo docker pull docker5gmedia/python3dscudaaction
sudo docker pull docker5gmedia/cuda8action
```

### Uninstallation

```
~/linux-amd64/helm delete owdev --purge
```

### OpenWhisk CLI

Follow this [configure-the-wsk-cli](https://github.com/5g-media/incubator-openwhisk-deploy-kube/tree/gpu#configure-the-wsk-cli) for installation and configuration of the CLI
