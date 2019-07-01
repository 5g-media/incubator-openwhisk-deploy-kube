### Kubernetes

Your first step is to create a Kubernetes cluster that is capable of supporting an OpenWhisk deployment.

In our validation tests, we deployed with either Minikube (**TBD**) or Kubernetes cluster (via [kubeadm](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/))

### GPU nodes

It is assumed that you already joined the nodes to the Kubernetes cluster as you normally do (e.g. via kubeadm join).

Your next step would be to register them as GPU nodes. For that, you need to install nvidia drivers
and nvidia-docker in each GPU node you have just joined, and deploy the k8s-nvidia-plugin:

* Follow the [Kubernetes Schedule GPUs](https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/) guide, in particular pay attention to [Official NVIDIA GPU device plugin](https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/#official-nvidia-gpu-device-plugin) section that details the requirements.


### OpenWhisk

You should deploy OpenWhisk control plane on your Kubernetes cluster using the forked [incubator-openwhisk-deploy-kube](https://github.com/5g-media/incubator-openwhisk-deploy-kube/tree/gpu) which already supports invoking actions on GPU Kubernetes nodes.

Your next step before the actual OpenWhisk deployment would be to:

* Ensure that the supported GPU runtimes are specified in `helm/openwhisk/runtimes.json`
  ```
  "deepspeech":[
     {
          "kind": "python:3ds@gpu",
          "default": true,
          "image": {
              "prefix": "docker5gmedia",
              "name": "python3dscudaaction",
              "tag": "latest"
          },
          "deprecated": false,
          "attached": {
              "attachmentName": "codefile",
              "attachmentType": "text/plain"
          }
     }
  ],
  "cuda":[
     {
          "kind": "cuda:8@gpu",
          "default": true,
          "image": {
              "prefix": "docker5gmedia",
              "name": "cuda8action",
              "tag": "latest"
          },
          "deprecated": false,
          "attached": {
              "attachmentName": "codefile",
              "attachmentType": "text/plain"
          }
     }
  ]
  ```

* Increase action memory limit by adding the below stanza to `mycluster.yaml`. Refer to [here](https://github.com/5g-media/incubator-openwhisk-deploy-kube/tree/gpu#customize-the-deployment) for more information on this file.
  ```
  whisk:
  actions:
    limits:
      memory:
        max: "2048m"
  ```
