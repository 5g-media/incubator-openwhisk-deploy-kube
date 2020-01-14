# Nvidia-GPU Node

Start with preparing your GPU host(s) to be installed with needed drivers so that they can function as Kubernetes GPU node(s).

You need to install nvidia drivers and nvidia-docker in each GPU node you have.

Let's start with a **fresh** RHEL 7.7 system, 4 CPU cores, 16 GB RAM, more then 100 GB disk and Quadro (M1000M) GPU.

## Nvidia cuda drivers

Install latest Cuda. Nvidia drivers automatically installed as well

The following steps taken from [Cuda-toolkit Installation](https://docs.nvidia.com/cuda/cuda-installation-guide-linux/index.html). It is recommended to refer to document for any clarification.

### Verify You Have a CUDA-Capable GPU
```
lspci | grep -i nvidia
```

### Verify the System Has gcc Installed
```
gcc --version
```

### Verify the System has the Correct Kernel Headers and Development Packages Installed
Install headers for RHEL

```
sudo yum install kernel-devel-$(uname -r) kernel-headers-$(uname -r)
```

### Choose an Installation Method
We will install via RPM

### Download the NVIDIA CUDA Toolkit
Donload the local RPM for RHEL 7 as mentioned [here](https://developer.nvidia.com/cuda-downloads?target_os=Linux&target_arch=x86_64&target_distro=RHEL&target_version=7&target_type=rpmlocal)

```
wget http://developer.download.nvidia.com/compute/cuda/10.2/Prod/local_installers/cuda-repo-rhel7-10-2-local-10.2.89-440.33.01-1.0-1.x86_64.rpm
```

**Only donwload it. Do not install at this point**

### Third-party package dependency
Browse [EPEL](https://fedoraproject.org/wiki/EPEL) link to install:

```
sudo subscription-manager repos --enable rhel-7-server-optional-rpms --enable rhel-7-server-extras-rpmssudo
sudo yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
```

### Install repository meta-data
Install the local RPM
```
sudo rpm --install cuda-repo-rhel7-10-2-local-10.2.89-440.33.01-1.0-1.x86_64.rpm
```

### Clean Yum repository cache
```
sudo yum clean expire-cache
```

### Install CUDA
```
sudo yum install nvidia-driver-latest-dkms
sudo yum install cuda
```

**Note:** If you have secured boot enabled in BIOS continue to next section to sign your modules, otherwise skip to [here](./k8s-gpu-rhel-prerequisites.md#nvidia-cuda-drivers-continue)

## Sign Nvidia kernel modules

The below steps are taken from [signing-kernel-modules-for-secure-boot](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/Kernel_Administration_Guide/sect-signing-kernel-modules-for-secure-boot.html) and [signing-a-compressed-kernel-module-for-use-with-secure-boot](https://unix.stackexchange.com/questions/438954/signing-a-compressed-kernel-module-for-use-with-secure-boot) 

### Create configuration file
```
cat << EOF > configuration_file.config 
[ req ] 
default_bits = 4096 
distinguished_name = req_distinguished_name 
prompt = no 
string_mask = utf8only 
x509_extensions = myexts 
[ req_distinguished_name ] 
O = Organization 
CN = Organization signing key 
emailAddress = E-mail address 
[ myexts ] 
basicConstraints=critical,CA:FALSE 
keyUsage=digitalSignature 
subjectKeyIdentifier=hash 
authorityKeyIdentifier=keyid 
EOF
```

Edit the file and update `O`, `CN` and `emailAddress`

### Generate the keys
```
openssl req -x509 -new -nodes -utf8 -sha256 -days 36500 -batch -config configuration_file.config -outform DER -out public_key.der -keyout private_key.priv 
```

### Add public key to persistence store
```
mokutil --import my_signing_key_pub.der 
```

Enter some password (remember it)

### Reboot the server 
```
shutdown -r now
```

**Note:** A dialog box is presented upon reboot. Provide the same password entered in previous `mokutil` step

### Sign the NVIDIA modules with the keys
Modules: nvidia, nvidia-uvm, nvidia-drm, nvidia-modeset

The below commands are invoked for a single module file. Repeat for rest.
```
unxz nvidia.ko.xz
perl /usr/src/kernels/$(uname -r)/scripts/sign-file sha256 private_key.priv public_key.der
nvidia.ko
xz -f nvidia.ko
```

### Reboot the server 
The modules should now get loaded
```
sudo lsmod | grep nvidia
```

## Nvidia cuda drivers continue

### Post-installation Actions
Set PATH as documented in [environment-setup](https://docs.nvidia.com/cuda/cuda-installation-guide-linux/index.html#environment-setup)

### Install Persistence Daemon
```
/usr/bin/nvidia-persistenced --verbose
```

### Install Writable Samples
```
cuda-install-samples-10.2.sh ~/
```

### Compiling and running the Examples
The NVIDIA CUDA Toolkit includes sample programs in source form. You should compile them by changing to `~/NVIDIA_CUDA-10.2_Samples` and typing make. The resulting binaries will be placed under `~/NVIDIA_CUDA-10.2_Samples/bin`

**Note:** if it failes, compile with `make -k`


Refer to [running-binaries](https://docs.nvidia.com/cuda/cuda-installation-guide-linux/index.html#running-binaries)


## Docker-CE
On your GPU RHEL nodes, install docker per [installing-docker-ce-on-redhat](https://nickjanetakis.com/blog/docker-tip-39-installing-docker-ce-on-redhat-rhel-7x) link.

The below commands and tips are taken from above link

### Obtain latest version of container-selinux

Refer to [centos packages](http://mirror.centos.org/centos/7/extras/x86_64/Packages/) to obtain the latest version. At this time of writing the latest version is: `container-selinux-2.107-3.el7.noarch.rpm`

### Install Docker-CE
Run the below using the latest version found above

```
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum makecache fast
sudo yum install -y http://mirror.centos.org/centos/7/extras/x86_64/Packages/container-selinux-2.107-3.el7.noarch.rpm
sudo yum install -y docker-ce
```

### Install socat
Download socat RPM from (here)[http://rpm.pbone.net/index.php3/stat/4/idpl/26480526/dir/redhat_el_7/com/socat-1.7.2.4-1.el7.rf.x86_64.rpm.html]

**Note:** You may need to install it with `rpm -i –-nopgpcheck`


### Verify docker
```
sudo docker run hello-world
```

### nvidia-docker runtime

Your next step would be to install nvidia docker runtime. Refer to [nvidia-docker]([https://github.com/NVIDIA/nvidia-docker) for more information.

In short, run these commands

```
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.repo | sudo tee /etc/yum.repos.d/nvidia-docker.repo

sudo yum install -y nvidia-container-toolkit
sudo systemctl restart docker
```

### Test nvidia-smi with the official CUDA image
```
docker run --gpus all nvidia/cuda:9.0-base nvidia-smi
```

### Install the deprecated nvidia-docker2
It is needed for nvidia k8s plugin

Run the below per [link](https://github.com/NVIDIA/nvidia-docker/wiki/Installation-\(version-2.0\)#centos-distributions-1)
```
sudo yum install nvidia-docker2
sudo pkill -SIGHUP dockerd
```

### Verify nvidia-smi
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

Then, restart the daemon
```
sudo systemctl stop docker
sudo systemctl start docker
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

**Note:** If coredns does not start and its logs indicate it cannot connect to API server then flush iptables per this [link](https://github.com/kubernetes/kubeadm/issues/193)
```
sudo minikube stop
sudo minikube delete

systemctl stop kubelet
systemctl stop docker
iptables --flush
iptables -tnat --flush
systemctl start docker
```

Refer again to [start-minikube](./k8s-gpu-rhel-prerequisites.md#start-minikube)

### Install Nvidia k8s plugin

(command taken from [enabling-gpu-support-in-kubernetes](https://github.com/NVIDIA/k8s-device-plugin#enabling-gpu-support-in-kubernetes))

```
kubectl create -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/1.0.0-beta4/nvidia-device-plugin.yml
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

In short, follow the below instructions.

Log into your GPU node where Minikube is installed

### mycluster.yaml

Create the fllowing file under `~/mycluster.yaml`. Replace ingress.apiHostName with output of `minikube ip` command

More information on mycluster.yaml can be found [here](https://github.com/5g-media/incubator-openwhisk-deploy-kube/blob/gpu/docs/configurationChoices.md)

```
whisk:
  ingress:
    type: NodePort
    apiHostName: <MINIKUBE IP>
    apiHostPort: 31001
  limits:
    actions:
      memory:
        max: "2048m"
nginx:
  httpsNodePort: 31001
```

### Helm

Download [helm](https://get.helm.sh/helm-v2.16.1-linux-amd64.tar.gz) and extract it to ~

Init helm causing tiller pod to get created

```
~/linux-amd64/helm init
```

When tiller pod is in `Running` state, grant necessary privileges to helm:

```
kubectl create clusterrolebinding tiller-cluster-admin --clusterrole=cluster-admin --serviceaccount=kube-system:default
```

### Actual OpenWhisk install

Label the node

```
kubectl label nodes --all openwhisk-role=invoker
```

Clone the repository and run helm to install OpenWhisk

```
cd ~
git clone https://github.com/5g-media/incubator-openwhisk-deploy-kube.git
cd incubator-openwhisk-deploy-kube
git checkout gpu
~/linux-amd64/helm install ./helm/openwhisk --namespace=openwhisk --name=owdev -f ~/mycluster.yaml
```

Installation can take a few minutes..

Wait for invoker-health pod to get created:
```
kubectl get pods -n openwhisk | grep invokerhealthtestaction
```

### Pull large GPU runtime images

GPU runtime images are large. When GPU action is being invoked for the first time, OpenWhisk invoker pulls them out from docker hub.
Depending on your network conditions this can take a fairly amount of time and can cause timeouts for the action invocation. In oder to avoid this, pull the following images in advance, by invoking these commands from all of your GPU nodes.

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
