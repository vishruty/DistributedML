# Kubernetes - Federation Learning Toolkit ((K)FLTK)
[![License](https://img.shields.io/badge/license-BSD-blue.svg)](LICENSE)
[![Python 3.6](https://img.shields.io/badge/python-3.7-blue.svg)](https://www.python.org/downloads/release/python-370/)
[![Python 3.6](https://img.shields.io/badge/python-3.8-blue.svg)](https://www.python.org/downloads/release/python-380/)
[![Python 3.6](https://img.shields.io/badge/python-3.9-blue.svg)](https://www.python.org/downloads/release/python-390/)

This toolkit can be used to run Distributed and Federated experiments. This project makes use of
Pytorch Distributed (Data Parallel) ([docs](https://pytorch.org/tutorials/beginner/dist_overview.html))
as well as Kubernetes, KubeFlow (Pytorch-Operator) ([docs](https://www.kubeflow.org/docs/)) and Helm ([docs](https://helm.sh/docs)) for deployment.
The goal of this project is to launch Federated Learning nodes in a true distribution fashion, with simple deployments 
using proven technology.

This project builds on the work by Bart Cox, on the Federated Learning toolkit developed to run with Docker and 
Docker Compose ([repo](https://github.com/bacox/fltk))


This project is tested with Ubuntu 20.04 and Arch Linux and Python {3.7, 3.8, 3.9}.

## Global idea
Pytorch Distributed works based on a `world_size` and `rank`s. The ranks should be between `0` and `world_size-1`.
Generally, the process leading the learning process has rank `0` and the clients have ranks `[1,..., world_size-1]`.

Currently, it is assumed that Distributed Learning is performed (and *not* Federated Learning), however, future 
extension of the project is planned to implement a `FederatedClient` that allows for a more realistic simulation of 
*Federated* Learning experiments.

### (Distributed Learning)

**General protocol:**

1. Client creation and spawning by the Orchestrator (using KubeFlows Pytorch-Operator)
2. Clients prepare needed data and model and synchronize using PyTorch Distributed.
   1. `WORLD_SIZE = 1`: Client performs training locally. 
   2. `WORLD_SIZE > 1`: Clients run epochs with DistributedDataParallel together.
   3. (FUTURE: ) Your federated learning experiment.
3. Client logs/reports progress during and after training.

**Important notes:**

* Data between clients (`WORLD_SIZE > 1`) is not shared
* Hardware can heterogeneous
* The location of devices matters (network latency and bandwidth)
* Communication is performed through RPC, aggregation is performed with `AllReduce`.

### Federated Learning
**General protocol:**

1. Client selection by the Federator.
2. The selected clients download the model.
3. Local training on the clients for X number of epochs
4. Weights/gradients of the trained model are send to the Federator
5. Federator aggregates the weights/gradients to create a new and improved model
6. Updated model is shared to the clients
7. Repeat step 1 to 6 until convergence/stopping condition.

**Important notes:**

* Data between clients is not shared to each other
* The data is non-IID
* Hardware can heterogeneous
* The location of devices matters (network latency and bandwidth)
* Communication can be costly




### Overview of deployed project
When deploying the system, the following diagram shows how the system operates. `PyTorchJob`s are launched by the 
Orchestrator (see the [Orchestrator charts](./charts/orchestrator)). The Extractor keeps track of progress (see the 
[Extractor charts](./charts/extractor)).

The `PyTorchJob`s can consist on a variable number of machines, with different hardware for the Master/Leader node and the
Client nodes. KubeFlow (not depicted) orchestrates the deployment of the `PyTorchJob`s.

![Overview of deployment](https://lucid.app/publicSegments/view/027793d8-a059-4c45-a030-660a492a4c0a/image.png)
## Something is broken/missing

It might be that something is missing, please open a pull request/issue).

## Project structure
Structure with important folders and files explained:

```
project
├── charts                       # Templates for deploying projects with Helm 
│   ├── extractor                   - Template for 'extractor' for centralized logging (using NFS)
│   └── orchestrator                - Template for 'orchestrator' for launching distributed experiments 
├── configs                      # General configuration files
│   ├── quantities                  - Files for (Kubernetes) quantity conversion
│   └── tasks                       - Files for experiment description
├── data                         # Directory for default datasets (for a reduced load on hosting providers)
├── fltk                         # Source code
│   ├── datasets                    - Datasets (by default provide Pytorchs Distributed Data-Parallel)
│   ├── nets                        - Default models
│   ├── schedulers                  - Learningrate schedulers
│   ├── strategy                    - (Future) Basic strategies for Federated Learning experiments
│   └── util                        - Helper utilities
│       ├── cluster                    * Cluster interaction (Job Creation and monitoring) 
│       ├── config                     * Configuration file loading
│       └── task                       * Arrival/TrainTask generation
└── logging                      # Default logging location
```

## Execution modes
Federatd Learning experiments can be set up in various ways (Simulation, Emulation, or fully distributed). Not all have the same requirements and thus some setup are more suited then others depending on the experiment.

### Simulation
With the method as single machine is used to execute all the different nodes in the system.
The execution is done in a sequential manner, i.e. first node 1 is executed, then node 2, and so on. One of the upsides of this option is the ability to use GPU acceleration for the computations.

### Docker-Compose (Emulation)
With systems like docker we can emulate a federated learning system on a single machine. Each node is allocated to one or more CPU cores and executed in an isolated container. This allows for real-time experiments where timing is important and where the execution of clients have effect on eachother. Docker also allows for containers to be limited by CPU speed, RAM, and network properties.

### Real distributed (Google Cloud)
In this case, the code is deployed natively on a machine, for example a cluster. 
The is the closest real-world approximation when experimenting with Federated Learning systems. This allows for real-time experiments where timing is important and where the execution of clients have effect on eachother. A downside of this method is the shear number of machines needed to run an experiment. Additionally the compute speed and other hardware spcifications are more difficult to limit.

### Hybrid
The Docker (Compose) and real-distributed method can be mixed in a hybrid system. For example two servers can run a set of docker containers that are linked to each other. Similarly, a set of docker images on a server can participate in a system with real distributed machines. 

## Models

* Cifar10-CNN (CIFAR10CNN)
* Cifar10-ResNet
* Cifar100-ResNet
* Cifar100-VGG
* Fashion-MNIST-CNN
* Fashion-MNIST-ResNet
* Reddit-LSTM

## Datasets

* CIFAR10
* Cifar100
* Fashion-MNIST
* MNIST

## Pre-requisites

The following tools need to be set up in your development environment before working with the (Kubernetes) FLTK.

* Hard requirements
  * Docker ([docs](https://www.docker.com/get-started)) (with support for BuildKit [docs](https://docs.docker.com/develop/develop-images/build_enhancements/))
  * Kubectl ([docs](https://kubernetes.io/docs/setup/))
  * Helm ([docs](https://helm.sh/docs/chart_template_guide/getting_started/))
  * Kustomize (3.2.0) ([docs](https://kubectl.docs.kubernetes.io/installation/kustomize/))
* Local execution (single machine):
  * MiniKube ([docs](https://minikube.sigs.k8s.io/docs/start/))
    * It must be noted that certain functionality might require additional steps to work on MiniKube. This is currently untested.
* Google Cloud Environment (GKE) execution:
  * GCloud SDK ([docs](https://cloud.google.com/sdk/docs/quickstart))
* Your own cluster provider:
  * A Kubernetes cluster supporting Kubernetes 1.16+.

## Getting started

Before continuing a deployment, first, the used datasets need to be downloaded. This is done to prevent the need for
downloading each dataset for each container. Per default, these models are included in the Docker container that gets
deployed on a Kubernetes Cluster.

### Download datasets
To download the models, execute the following command from the [project root](.).

```bash
python3 -m fltk extractor ./configs/example_cloud_experiment.json  
```

## Deployment

This deployment guide will provide the general process of deploying an example deployment on
the created cluster. It is assumed that you have already set up a cluster (or emulation tool like MiniKube to execute the 
commands locally).

**N.B.** This setup expects the NodePool on which you **want** to run training experiments, to have **Taints**, 
this should be set for the selected nodes. For more information on GKE see [docs](https://cloud.google.com/kubernetes-engine/docs/how-to/node-taints).

In this project we assume the following taint to be set, this can also be done using `kubectl` for each node.

```
fltk.node=normal:NoSchedule
```

Programmatically, the following `V1Toleration` allows pods to be scheduled on such 'tainted' nodes, regardless of the value
for `fltk.node`.
```python
from kubernetes.client import V1Toleration

V1Toleration(key="fltk.node",
             operator="Exists",
             effect="NoSchedule")
```

For a more strict Toleration (specific to a value), the following `V1Toleration` should be generated.

```python
V1Toleration(key="fltk.node",
             operator="Equals",
             value="normal",
             effect='NoSchedule')
```

For more information on the programmatic creation of `PyTorchJobs` to spawn on a cluster, refer to the 
`DeploymentBuilder` found in [`./fltk/util/cluster/client.py`](./fltk/util/cluster/client.py) and the function
`construct_job`.

### GKE / MiniKube
Currently, this guide was tested to result in a working FLTK setup on GKE and MiniKube.

The guide is structured as follows:

1. (Optional) Setup a Kubernetes Dashboard instance for monitoring
2. Install KubeFlow's Pytorch-Operator (in a bare minimum configuration).
   * KubeFlow is used to create and manage Training jobs for Pytorch Training jobs. However, you can also
      extend the work by making use of KubeFlows TF-Operator, to make use of Tensorflow.
3. (Optional) Deploy KubeFlow PyTorch Job using an example project.
4. Install an NFS server.
   * To simplify FLTK's deployment, an NFS server is used to allow for the creation of `ReadWriteMany` volumes in Kubernetes.
      These volumes are, for example, used to create a centralized logging point, that allows for easy extraction of data
      from the `Extractor` pod.
5. Setup and install the `Extractor` pod.
   * The `Extractor` pod is used to create the required volume claims, as well as create a single access point to gain
     insight into the training process. Currently, it spawns a pod that runs the a `Tensorboard` instance, as a
     `SummaryWriter` is used to record progress in a `Tensorboard` format. These are written to a `ReadWriteMany` mounted
     on a pods `$WORKING_DIR/logging` by default during execution.
6. Deploy a default FLTK experiment.

### (Optional) setup Kubernetes Dashboard
Kubernetes Dashboard provides a comprehensive interface into some metrics, logs and status information of your cluster
and the deployments it's running. To setup this dashboard, Helm can be used as follows:


```bash
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
helm install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard
```

After setup completes, running the following commands (in case you change the release name to something different, you can 
fetch the command using `helm status your-release-name --namespace optional-namespace-name`) to connect to your Kubernetes
Dashboard.
```bash
export POD_NAME=$(kubectl get pods -n default -l "app.kubernetes.io/name=kubernetes-dashboard,app.kubernetes.io/instance=kubernetes-dashboard" -o jsonpath="{.items[0].metadata.name}")
kubectl -n default port-forward $POD_NAME 8443:8443
```

Then browsing to [https://localhost:8443](https://localhost:8443) on your machine will connect you to the Dashboard instance.
Note that the certificate is self-signed of the Kubernetes Dashboard, so your browser may give warnings that the site is 
unsafe.

### Installing KubeFlow + PyTorch-Operator
Kubeflow is an ML toolkit that allows to for a wide range of distributed machine and deep learning operations on Kubernetes clusters. 
FLTK makes use of the 1.3 release. We will deploy a minimal configuration, following the documentation of KubeFlows 
[manifests repository](https://github.com/kubeflow/manifests). If you have already setup KubeFlow (and PyTorch-Operator) 
you may want to skip this step.

```bash
git clone https://github.com/kubeflow/manifests.git --branch=v1.3-branch
cd manifests
```

You might want to read the `README.md` file for more information. Using Kustomize, we will install the default configuration
files for each KubeFlow component that is needed for a minimal setup. If you have already worked with KubeFlow on GKE
you might want to follow the GKE deployment on the official KubeFlow documentation. This will, however, result in a slightly 
higher strain on your cluster, as more components will be installed.


#### Setup cert-manager
```bash
kustomize build common/cert-manager/cert-manager/base | kubectl apply -f -
# Wait before executing the following command, as 
kustomize build common/cert-manager/kubeflow-issuer/base | kubectl apply -f -
```

#### Setup Isto
```bash
kustomize build common/istio-1-9/istio-crds/base | kubectl apply -f -
kustomize build common/istio-1-9/istio-namespace/base | kubectl apply -f -
kustomize build common/istio-1-9/istio-install/base | kubectl apply -f -
```

```bash
kustomize build common/dex/overlays/istio | kubectl apply -f -
```

```bash
kustomize build common/oidc-authservice/base | kubectl apply -f -
```

#### Setup knative
```bash

kustomize build common/knative/knative-serving/base | kubectl apply -f -
kustomize build common/istio-1-9/cluster-local-gateway/base | kubectl apply -f -
```

#### Setup KubeFlow
```bash
kustomize build common/kubeflow-namespace/base | kubectl apply -f -
```

```bash
kustomize build common/kubeflow-roles/base | kubectl apply -f -
kustomize build common/istio-1-9/kubeflow-istio-resources/base | kubectl apply -f -
```

#### Setup PyTorch-Operator
```bash
kustomize build apps/pytorch-job/upstream/overlays/kubeflow | kubectl apply -f -
```

### (Optional) Testing KubeFlow deployment

In case you want to test your KubeFlow deployment, an example training job can be run. For this, an example project of
the pytorch-operator [repository](https://github.com/kubeflow/pytorch-operator/) can be used.

```bash
git checkout https://github.com/kubeflow/pytorch-operator.git
cd pytorch-operator/examples/mnist
```

Follow the `README.md` instructions, and make sure to *rename* the image name in `pytorch-operator/examples/mnist/v1/pytorch_job_mnist_gloo.yaml`
(line 33 and 35), to your project on GCE. Also commend out the `resource` descriptions in lines 20-22 and 36-38. Otherwise
jobs require GPU support to run.

Build and push the Docker container, and execute the command to launch your first PyTorchJob on your cluster.

```bash
kubectl create -f ./v1/pytorch_job_mnist_gloo.yaml
```

### Create experiment Namespace
Create your namespace in your cluster, that will later be used to deploy experiments. This guide (and the default
setup of the project) assumes that the namespace `test` is used. To create a namespace, run the following command with your cluster credentials set up before running these commands.

```bash
kubectl namespace create test
```

### Installing NFS
During the execution, `ReadWriteMany` persistent volumes are needed. This is because each training processes master
pod uses a`SummaryWriter` to log the training progress. As such, multiple containers on potentially different nodes require 
read-write access to a single volume. One way to resolve this is to make use of Google Firestore (or 
equivalent on your service provider of choice). However, this will incur significant operating costs, as operation starts at 1 TiB (~200 USD per month). As such, we will deploy our own a NFS on our cluster. 

In case this does not need your scalability requirements, you may want to set up a (sharded) CouchDB instance, and use
that as a data store. This is not provided in this guide.


For FLTK, we make use of the `nfs-server-provisioner` Helm chart created by `kvaps`, which neatly wraps this functionality in an easy
to deploy chart. Make sure to install the NFS server in the same *namespace* as where you want to run your experiments.

Running the following commands will deploy a `nfs-server` instance (named `nfs-server`) with the default configuration. 
In addition, it creates a Persistent Volume of `20 Gi`, allowing for `20 Gi` `ReadWriteMany` persistent volume claims. 
You may want to change this amount, depending on your need. Other service providers, such as DigitalOcean, might require the
`storageClass` to be set to `do-block-storage` instead of `default`.

```bash
helm repo add kvaps https://kvaps.github.io/charts
helm update
helm install nfs-server kvaps/nfs-server-provisioner --namespace test --set persistence.enabled=true,persistence.storageClass=standard,persistence.size=20Gi
```

To create a Persistent Volume (for a Persistent Volume Claim), the following syntax should be used, similar to the Persistent
Volume description provided in [./charts/extractor/templates/fl-log-claim-persistentvolumeclaim.yaml](./charts/extractor/templates/fl-log-claim-persistentvolumeclaim.yaml).
Which creates a Persistent Volume that uses the values provided in [./charts/fltk-values.yaml](./charts/fltk-values.yaml).


**N.B.** If you wish to use a Volume as both **ReadWriteOnce** and **ReadOnlyMany**, GCE does **NOT** provide this functionality
You'll need to either create a **ReadWriteMany** Volume with read-only Claims, or ensure that the writer completes before 
the readers are spawned (and thus allowing for **ReadWriteOnce** to be allowed during deployment). For more information
consult the Kubernetes and GKE Kubernetes 

### Creating and pushing Docker containers
On your remote cluster, you need to have set up a docker registry. For example, Google provides the Google Container Registry
(GCR). In this example, we will make use of GCR, to push our container to a project `test-bed-distml` under the tag `fltk`.

This requires you to have enabled the GCR in your GCE project beforehand. Make sure that your docker installation supports
Docker Buildkit, or remove the `DOCKER_BUILDKIT=1` part from the command before running (this might require additional changes
in the Dockerfile).

**N.B.** When running for the first time, make sure to have downloaded the Datasets used in the repo.
This can be done by executing the following command with an activated environment.

```bash
python3 -m fltk extractor configs/example_cloud_experiment.json
```

To build the container and push to your GCR repository, execute the following commands.
```bash
DOCKER_BUILDKIT=1 docker build . --tag gcr.io/test-bed-distml/fltk
docker push gcr.io/test-bed-distml/fltk
```

**N.B.** when running in Minikube, you can also set up a local registry. An example of how this can be achieved 
can be found [in this Medium post by Shashank Srivastava](https://shashanksrivastava.medium.com/how-to-set-up-minikube-to-use-your-local-docker-registry-10a5b564883).


### Setting up the Extractor

This section only needs to be run once, as this will set up the TensorBoard service, as well as create the Volumes needed
for the deployment of the `Orchestrator`'s chart. It does, however, require you to have pushed the docker container to a 
registry that can be accessed from your Cluster.

**N.B.** that removing the `Extractor` chart will result in the deletion of the Persistent Volumes once all Claims are 
released. This **will remove** the data that is stored on these volumes. **Make sure to copy**  the contents of these directories to your local file system before uninstalling the `Extractor` Helm chart. The following commands deploy the `Extractor`
Helm chart, under the name `extractor` in the `test` namespace.
```bash
cd charts
helm install extractor -f values.yaml --namespace test
```

And wait for it to deploy. (Check with `helm ls --namespace test`)

**N.B.** To download data from the `Extrator` node (which mounts the logging director), the following `kubectl`
 command can be used. This will download the data in the logging directory to your file system. Note that downloading
many small files is slow (as they will be compressed individually). The command assumes that the default name is used 
`fl-extractor`.

```bash
kubectl cp --namespace test fl-extractor:/opt/federation-lab/logging ./logging
```

⚠️ Make sure to test your configurations before running your experiments, to ensure that your data is written in a
persistent fashion. Writing to a directory that is not mounted to an NFS disk may result in dataloss.

⚠️ For federated learning experiments, data is written by the Federator to a disk, as such only a single mount of an
NFS is needed  (see for example how [`V1PytorchTrainJob`](fltk/util/cluster/client.py)s are constructed by the Orchestrator).
It is advisable to load the data of Federated learning experiments using the `pandas` library, this can be done as follows.
For data exploration, using a Jupyter Notebook may be advisable.

```python
import pandas as pd
experiment_file = 'path/to/your/experiment.csv'
df = pd.read_csv(experiment_file)

df
```
This should display the contents of the csv file parsed into a `pd.DataFrame`, which should have the following schema.
```
round_id,train_duration,test_duration,round_duration,num_epochs,trained_items,accuracy,train_loss,test_loss,timestamp,node_name,confusion_matrix
```


### Launching an experiment
We have now completed the setup of the project and can continue by running actual experiments. If no errors occur, this
should. You may also skip this step and work on your code, but it might be good to test your deployment
before running into trouble later.

#### Federated Experiment
```bash
helm install flearner charts/orchestrator --namespace test -f charts/fltk-values.yaml\
  --set-file orchestrator.experiment=./configs/federated_tasks/example_arrival_config.json,\
  orchestrator.configuration=./configs/example_cloud_experiment.json
```

#### Distributed Experiment
```bash
helm install flearner charts/orchestrator --namespace test -f charts/fltk-values.yaml\
  --set-file orchestrator.experiment=./configs/distributed_tasks/example_arrival_config.json,\
  orchestrator.configuration=./configs/example_cloud_experiment.json
```

**N.B.** The `./configs/distributed_tasks` and `./configs/distributed_tasks` dictories contain experiment configurations,
i.e. which networks should be run, etc. The `./configs/example_cloud_experiment.json` file contains shared configuration 
information, e.g. on how to log experiments. Make sure to ensure that all data can be saved to a persistent volume. 
E.g. by creating a shared volume for `models` and saving to the corresponding directory.

To debug the deployment append with the `--debug` flag, note that you may need to uninstall a prior deployment.
Alternatively, you can use the `upgrade` argument, however, currently the orchestrator does not support updated
releases. Pull requests are welcome for adding this functionality.

**N.B.** Passing the `--set-file` flag is optional, but will take by default the 
`benchmarking/example_cloud_experiment.json` file. This follows the symlink in `charts/orchestrator/configs/`, 
as Helm does not allow for accessing files outside a charts directory.

This will spawn an `fl-server` Pod in the `test` Namespace, which will spawn Pods (using `V1PyTorchJobs`), that
run experiments. It will currently make use of the [`configs/example_cloud_experiment.json`](configs/benchmarking/example_cloud_experiment.json)
default configuration. As described in the [values](./charts/orchestrator/values.yaml) file of the `Orchestrator`s Helm chart


## Running tests
In addition to the FLTK framework implementation, some tests are available to prevent regression of bugs. Currently, only a limited subset of features 
is tested of FLTK. All current tests are deterministic, flaky tests indicate that something is likely broken.

To run the test included, run the following command to run the tests in a terminal:


```bash
python3 -m pytest tests
```

These tests should all pass (currently, mainly providing smoke tests). Warnings can be safely ignored.
### Prerequisites
Setup a `development` virtual environment, using the [`requirements-dev.txt`](requirements-dev.txt) requirements file.
This will install the same requirements as the [`requirements.txt`](requirements.txt), with some additional packages needed to run the tests.

```bash
python3 -m venv venv-dev
source venv-dev/bin/activate
pip install -r requirements.txt
```

### Executing tests
Make sure to run in a shell with the `venv-dev` virtual environment. With the environment enabled, we can run using:

```bash
python3 -m pytest -v
```

Which will collect and run all the tests in the repository, and show in `verbose` which tests passed. 


## Known issues / Limitations

* Currently, there is no GPU support in the Docker containers, for this the `Dockerfile` will need to be updated to
accomodate for this.
