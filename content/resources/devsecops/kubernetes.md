---
title: "Kubernetes"
date: 2026-01-20
---


## 1. Vannilla vs Custom kubernetes
- Vannilla (via kubeadm), its a official, standard way to boot strap a cluster and gives you the trust, most unopinoted kubernetes experiance.
- Lightweight/Custom (like k3s, MicroK8s, etc.): These are fantastic, simplified distributions that bundle everything together  - including networking addons etc.
- **Important**: Vannilla Kubernetes adopts decentralized approach, where components are loosly coupled.
- For example, Its supports multiple network plugins such as Calico, Cilium, Flannel, etc. Each has different requirements for IP addressing, routing, and networking architecture. So, instead of automatically assigning the range of ip addresses for pods by default, adminstrator has to specify it manually at the time of initialization `kubeadm init --pod-network-cidr=10.244.0.0/16`.

## 2. Reference Repository: https://github.com/silent-resistor/cka/


## 3. Installation procedure
### 3.1 Prerequisites and container runtime
- before installing kubernetes, your all nodes need a base software to actually run the containers like `containerd` or `CRI-O`
- [Container Runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)
- or [CRI-O](https://cri-o.io/)
- do it with sander's scripts [here](https://github.com/silent-resistor/cka/blob/master/setup-container.sh)

### 3.2 Installing core tools
- kubeadm, kubelet, kubectl in all of the nodes
- do it with sander's scripts [here](https://github.com/silent-resistor/cka/blob/master/setup-kubetools.sh)
- or manually with [official docs](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)

### 3.3 Initializing the cluster
- on the control plane node, run `kubeadm init`
- do it with sander's scripts [here](https://github.com/silent-resistor/cka/blob/master/setup-kubeadm-init.sh)
- or manually with [official docs](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
- 
- you can also pass some flags to kubeadm init to customize the cluster
  - `--pod-network-cidr` to specify the CIDR for the pod network
  - `--service-cidr` to specify the CIDR for the service network
  - `--apiserver-advertise-address` to specify the IP address to advertise the API server
- In case initialization fails, you can reset with the following commands and try again
  ```bash
  kubeadm reset -f
  sudo rm -rf /etc/cni/net.d
  sudo rm -rf $HOME/.kube/config
  kubeadm init
  ```
- If it's still failing or getting stuck, always run with verbose to see what's happening `kubeadm init --v=5`
- Most of the time, it fails at creating and configuring the kubelet service, so always inspect the kubelet service status
  ```bash
  systemctl status kubelet
  sudo journalctl -u kubelet --no-pager -n 100
  ```
- Once kubeadm init is done, it prints a few commands to run on the control plane node to set up kubectl with administrative access
  ```bash
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
  ```

### 3.4 Installing the network plugin
- install a network plugin like Calico, Cilium, etc.
- do it with sander's scripts [here](https://github.com/sandervanvugt/setup-network-plugin.sh)
- or manually with [official docs](https://kubernetes.io/docs/concepts/cluster-administration/addons/)

### 3.5 Joining worker nodes and control plane nodes
- If you wanted to get the join command for worker nodes, run `kubeadm token create --print-join-command` on the control plane node
- If you wanted to get the join command for control plane nodes, run `kubeadm token create --print-join-command --control-plane` on the control plane node
- or manually with [official docs](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/join-control-plane-node/)


## 4. Problems faced: (The Real Kubernetes Experience)
### 4.1 While intializing the cluster with `kubeadm init`, its getting stuck at `[control-plane-check] checking-kube-shedular at http://127.0.0.1:10259/livez` for very long time..

#### 4.1.1 Inspection1: The Container Detective Work
  - `systemctl status containerd` is `active (running)` but doing nothing useful.
  - `crictl --runtime-endpoint unix:///var/run/containerd/containerd.sock ps` shows no containers 
  - I restarted containerd after configuring the runtime endpoint with `sudo crictl config --set runtime-endpoint=unix:///var/run/containerd/containerd.sock`, but it didn't help (of course it didn't)
  - `crictl --runtime-endpoint unix:///var/run/containerd/containerd.sock logs` showing few failed container with the error as `rpc error: code = NotFound desc =an error occured when try to find the container`
  - A controlplane component may have crashed or exited when started by the container runtime. To troubleshoot, list all containers using preferred container runtime cli. 
  - `crictl --runtime-endpoint unix:///var/run/containerd/containerd.sock ps -a | grep kube | grep -v pause`
  - once found the failed container in here, inspect the logs of that container to understand the issue
  - `crictl --runtime-endpoint unix:///var/run/containerd/containerd.sock logs <container-id>`
  

#### 4.1.2 Inspection2: The cgroup Mystery
  - you may find that the kubelet does not know which systemd cgroup to use, please see the config at `/etc/containerd/config.toml`
    ```toml
    # sudo containerd config default | sudo tee /etc/containerd/config.toml
    # not sure, lets compare this with below
    version = 2
    [plugins]
    [plugins."io.containerd.grpc.v1.cri"]
    [plugins."io.containerd.grpc.v1.cri".containerd]
    discard_unpacked_layers = true
    [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]
    [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
    runtime_type = "io.containerd.runc.v2"
    [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
    ```
  - In kubernetes,`kubeadm` doesnt run the containers directly, it creates configuration files (static pod manifests) for the containers and waits for `kubelet` to read them and tell `containerd` to start the pods. If its hanging, `kubelet` is likely failing.
  
  4.1.2.1. **Case: The RPC Nightmare**
  - Since containerd is running fine, but doing nothing, the `kubelet` is likely crash looping. 
  - `sudo journalctl -u kubelet -n 100 --no-pager` to check kubelet's mental state
    ```
    rpc error: code = Unimpleted desk = failed to create containerd task: failed to start shim version (3): not implemented" pod="kube-system/kube-controller-manager-node1"
    ```
  - It means `kubelet` is trying to spinup the control plane pods, but `containerd` is rejecting request because it doesnt understand the "shim version3" instruction.
  - This almost always means installed version of `containerd` is too old to work with the modern version of Kubernetes tools installed. Modern Kubernetes (v1.26+) expects `containerd` to be atleast version of 1.6 or higher. Lets verify version of `containerd`
    ```bash
    containerd --version
    1.7.24

    # if the version is less than 1.6, you need to upgrade containerd
    # sudo apt update && sudo apt install -y containerd 
    # systemctl restart containerd
    # and reset the kubeadm, and restart
    ```
    - `containerd` background is correctly running the version of `1.7.24` (which onlu understand shim APIs upto version2), however, when it tries to sin up a kubernetes pod, it searches for your system's PATH and executing `containerd-shim-runc-v2` binary file that belongs to `containerd` 2.0+ which expects shim version 3.
    - Your system has orphaned,newer "shim" binary (likey left over from previous installation), hijacking the process. The 1.7 daemon tries to talk to 2.0 shim, which doesnt understand its "version3" instruction, so container creation fails.
    - Surgical fix: stop everything and kill the ghosts
    ```bash
    sudo systemctl stop containerd
    sudo killall conatinerd-shim-runc-v2
    which -a containerd-shim-runc-v2 | xargs sudo rm -f
    sudo rm -rf /usr/local/sbin/runc
    sudo rm -rf /usr/local/bin/runc
    sudo apt install --reinstall containerd
    sudo systemctl restart containerd
    sudo kubeadm reset -f
    sudo rm -rf /var/lib/cni/
    sudo rm -rf $HOME/.kube/config
    sudo kubeadm init
    ```
  
  4.1.2.2. **Case: The Swap Police**
  - The kubelet will absolutely refust to start if swap memory is enabled on your machine (unless you explicitly configured it to ignore swap, which is not standard), so disable it first. `sudo swapoff -a`
  - kubelet will absolutely refuse to start if swap memory is enabled. It's like that friend who won't come over if you have a messy apartment.
  - or permanently disable it by commenting out the swap entry in `/etc/fstab`

  4.1.2.3. **Case: The Nuclear Option**
  - Lets clear the battlefield with `kubeadm reset -f`, as intialization leaves behind a mess of certificates, partial configurations, data directories etc. 
    ```bash
    sudo kubeadm reset -f
    sudo rm -rf /etc/cni/net.d
    sudo rm -rf $HOME/.kube/config
    ```
  
### 4.2 How much its easier to configure the network addon like `calico` ? 
- We are handling the network configuration blueprints (yaml files) to the kubernetes API server, which lives on control node. Kubernetes takes those yaml files, and creates the workloads called a `DaemonSet` whose entire job is to automatically ensure that one copy of a specific pod (in this case calico routing agent) runs on every machine in the cluster.
- When you join your worker nodes later, the control plane will automatically push the calico pods down to them without you lifting the finger.
- Lets understand how to install the calico network addon.
  ```bash
  # Incase if you are cleanig up the network plugin and restarting the installation
  # Tear down the existing calico installation
  kubectl delete -f https://raw.githubusercontent.com/projectcalico/calico/v3.31.4/manifests/custom-resources.yaml
  kubectl delete -f https://raw.githubusercontent.com/projectcalico/calico/v3.31.4/manifests/tigera-operator.yaml
  # clean up any leftover network configs
  sudo rm -rf /etc/cni/net.d
  sudo rm -rf /var/lib/cni
  # watch calico pods are died
  watch kubectl get pods -n calico-system

  # **Install the Tigera Calico Operator:** Modern Calico uses `operator` pattern, which installs a manager that handles the lifecycle of network components automatically.
  kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.31.4/manifests/tigera-operator.yaml
  
  # **Apply the Calico Custom Resource:** This command tells the operator to actually start building the calico network using the `default` settings.
  kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.31.4/manifests/custom-resources.yaml
  
  # **Watch the network come alive:** It will take a minute or two for the calico operator to download the necessary contaier images and spin everything up.
  watch kubectl get pods -n calico-system
  ```

- I understand, sometimes, we may or for sure, face issues something like below when you ran the above commands:
  ```
  Error from server (NotFound): error when creating "https://raw.githubusercontent.com/projectcalico/calico/v3.31.4/manifests/custom-resources.yaml": the server could not find the requested resource (post installations.operator.tigera.io)
  resource mapping not found for name: "default namespace: "" from "https://raw.githubusercontent.com/projectcalico/calico/v3.31.4/manifests/custom-resources.yaml": no matches for kind "Goldmine" in versioin "operator.tigera.io/v1" to ensure CRDs are installed first.
  ```
- This error is a classic kubernetes timing issue. The specific message `ensure CRDs are installed first` tells exactly what went wrong
    - When we apply the first first file `tigera-operator.yaml`, it install custom resources definitions (CRDs), essentially teaching kubernetes API new vocabulary like "installation" or "APIServer" ( or apparently "Goldmine" or "wisker", depending upon exact minifest version you hit). So, this case we can follwoing stuff, to better understand the issue.
      ```bash
      # **Re-apply the operator:** lets make sure operator installed, lets use apply instead of create, generally a safer habit as it gracefully overwrites partial configurations
      kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.31.4/manifests/tigera-operator.yaml

      # **Verify the CRDs are registered:**  lets check if the k8s learning new vocobulary, If not, something else is the issue.
      kubectl get crds | grep tigera
      kubectl get pods -n tigera-operator
      # Apply the custom resource again only if you see the CRDs pods running like Goldmine or Wisker
      kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.31.4/manifests/custom-resources.yaml
      ```
- If still its custom resources are not being created, there are two possibiliteis
  - Possibility1: The Manager doesnt have blueprint yet, i.e there might be a issue with the manifest file `tigera-operator.yaml`
  - possibility2: **The Pod CIDR mismatch (The classsic `kubeadm` trap**
    - if the blueprint was applied, the manager might be stuck, Calico's default blueprint specifically look for cluster created with flag `--pod-network-cidr=192.168.0.0/16` during the `kubeadm inti`.
    - if you didnt use that exact flag, operator get confused and pauses indefinately
    - Lets understand this better, by knowing whats the manager complaining about 
    ```bash
    kubectl logs -n tigera-operator -l k8s-app=tigera-operator | tail -n 20
    # {"level": "info", "ts": "2025-10-15T10:12:34.567Z", "logger": "controller_windows", "msg": "Failed to check if resource is ready - will retry",
    # "name": "", "kind": "IPAMConfiguration", "Error": "the server is currently unable handle the request"}

    # Lets also view the current network configuration of the system
    kubectl describe installation default | grep -i -E "cidr|pod"
    #CIDR:     192.168.0.0/16
    ```
  - When we ran the command to intialize the cluster earlier, we havent specified the `--pod-network-cidr` flag, so kubernetes brain (controller manager) was never told to assign IP addresses to nodes. The tigera operator is crashing and throwing that `unable to handle the request` error because it is desperatly trying to setup the `192.168.0.0/16` network, but the cluster itself wasnt configured to allow it.
  - Since we just built this, the absolute fastest and cleanets way to fix this is a quick reset.
    ```bash
    # Reset the cluster
    sudo kubeadm reset -f
    sudo rm -rf /etc/cni/net.d
    sudo rm -rf $HOME/.kube/config
    
    # Reinitialize the cluster with the correct CIDR
    sudo kubeadm init --pod-network-cidr=192.168.0.0/16

    # Setup your local access
    mkdir -p $HOME/.kube
    sudo cp -f /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config

    # install calico
    kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.31.4/manifests/calico.yaml
    kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.31.4/manifests/custom-resources.yaml
    ```

### 4.3 How can I integrate a Kubernetes cluster to Jenkins? I want Jenkins to utilize this cluster for running its pipelines, where pods in the cluster act as ephemeral agents.
- Yeah understood, here is solution, works as: Jenkins will talk to the kubernetes cluster, spin up a temperory pod (acting as building agent), using specified image in the Jenkinsfile, run the build steps and then destroy the pod immediately when it finishes. 
- Here we are making better utilization of resources, by not permanently attaching the resources to the specific jenkins agent type. rather we are creating specified agent types (pods) on-demand.
- Also, this helps in scaling the build agents dynamically based on the workload. No need to contact infrastructure team for adjusting the resources on agents.
- Ok lets start, with creating a service account for jenkins to talk to the kubernetes cluster.
- Jenkins need permission to create and delete pods in your cluster. We will create the `ServiceAccount`, and give adminstrative rights for now so you dont hit permission roadbloacks while building.
  ```bash
  # Create the service account and cluster role binding
  cat <<EOF | kubectl apply -f -
  apiVersion: v1
  kind: ServiceAccount
  metadata:
    name: jenkins-agent
    namespace: default
  ---
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRoleBinding
  metadata:
    name: jenkins-agent-binding
  roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: ClusterRole
    name: cluster-admin
  subjects:
    - kind: ServiceAccount
      name: jenkins-agent
      namespace: default
  EOF

  # Since modern kubernetes (v1.24+), doesnt auto-generate permanent passwords for ServiceAccount, 
  # we will manually generate a long-live token that can be used by jenkins to login in and spin up pods here
  kubectl create token jenkins-agent --duration=87600h

  # Get your cluster address (ip:port), so jenkins knows exactly where to knock the door 
  kubectl cluster-info | grep "control plane"
  ```
- Now we have jump over to jenkins and configure the kubernetes plugin there.
    - Go to Manage Jenkins -> Plugins -> Available Plugins
    - Search for "Kubernetes" and install the plugin
    - Go to Manage Jenkins -> Configure System -> Cloud -> Add a new cloud -> Kubernetes
    - In the configuration
      - Kubernets URL: <cluster-ip:port> 
      - Disable kubernetes server certificate check
      - Credentials: Create new credential with the token we generated earlier, and name it something like `k8s-jenkins-token`
      - click `Test Connection` to verify the connection
      - Pod labels -> Add -> key: `jenkins`, value: `slave`
      - Pod Retention -> On Failure
      - Enable Garble Collection with timout around 5 mins
- Let's create a minimal image for the worker to run, so that the Jenkins JNLP (sidecar) and actual worker containers can be in the same Pod.
- Note: Please do not mix both containers (Jenkins JNLP and actual worker) in the same pod, as it will lead to the pod disconnecting from Jenkins when the actual work in this container consumes all resources and doesn't get a chance to communicate with Jenkins.
- Earlier I did this for convenience, and I gradually observed that pods leave no trace when they fail due to resource issues. I couldn't observe anything in Grafana; it was completely dark.
```bash
# Create a simple dockerfile for the worker container
cat <<EOF > Dockerfile
FROM ubuntu:22.04 as UBUNTU_BASE

ARG CONTAINER_USER=jenkins
ARG CONTAINER_USER_PASSWORD=jenkins
ARG ROOT_USER_PASSWORD=root
ARG CONTAINER_UID=1000
ARG CONTAINER_GID=1000
ENV LANG=C.UTF-8 LC_ALL=C.UTF-8 HOME=/home/${CONTAINER_USER}

# ------------- System packages and tools -------------
RUN apt update -y && apt install -y curl wget git vim python3 python3-pip
RUN apt clean -y && rm -rf /var/lib/apt/lists/*

# ------------ User & directories ---------
RUN groupadd -g ${CONTAINER_GID} ${CONTAINER_USER} && \
    useradd -u ${CONTAINER_UID} -g ${CONTAINER_GID} --create-home --shell /bin/bash ${CONTAINER_USER} && \
    echo "${CONTAINER_USER}:${CONTAINER_USER_PASSWORD}" | chpasswd && \
    echo "root:${ROOT_USER_PASSWORD}" | chpasswd && \
    mkdir -p /workspace && \
    chown -R ${CONTAINER_USER}:${CONTAINER_GID} /workspace
USER ${CONTAINER_USER}
WORKDIR /workspace
EOF

# Build the image
docker build -t ubuntu-jenkins-agent:latest -t ubuntu-jenkins-agent:1.0 -f Dockerfile .

# Push the image to registry
docker push ubuntu-jenkins-agent:latest
docker push ubuntu-jenkins-agent:1.0
```

- Incase your registry is on bare http, we need to configure containerd to allow insecure registry. Lets add below configuration to `/etc/containerd/config.toml` and restart the containerd `systemctl restart containerd`. Please note this works in case of older containerd versions.
```toml
version = 2
[plugins]
  [plugins."io.containerd.grpc.v1.cri".registry.mirrors."your-registry-ip:5000"]
    endpoint = ["http://your-registry-ip:5000"]
  [plugins."io.containerd.grpc.v1.cri".registry.configs."your-registry-ip:5000".tls]
    insecure_skip_verify = true
```
- We need to store the artifactory username and password inside kuberenetes as a specific type of secret, lets run this on control node
```bash
kubectl create secret docker-registry my-artifactory-secret \
  --docker-server=your-registry-ip:5000 \
  --docker-username=your-username \
  --docker-password=your-password \
  --docker-email=your-email

# lets verify the secret
kubectl get secret my-artifactory-secret -o yaml
```

- Now lets create a simple Jenkinsfile that spins up a kubernetes pod as an agent
```groovy
// Jenkins file
// Designed for Multi-branch Pipeline
// SCM is inherited from pipeline config
// Each parrallel job runs in a ephermal kubernetes pod.

// ========== Container Image ============
import groovy.transform.Field
@Field static final String REGISTRY = "your-registry-ip:5000"
static Map getImages() {
    return [
        ubuntu: "${REGISTRY}/ubuntu-jenkins-agent:latest"
    ]
}

// ========= XUnit Configuration =========
@Field static final Map XUNIT_CONFIG = [
  $class: 'XUnitPublisher',
  testTimeMargin: '3000',
  thresholds: [
    [$class: 'FailedThreshold', unstableThreshold: '1']
  ],
  tools: [[$class: 'JUnitType', pattern: 'tests_results/*.xml']]
]


// ======= Pipeline start here
pipeline {
  agent none
  options {
    buildDiscarder(logRotator(daysToKeepStr: '10'))
    timestamps()
  }
  parameters {
    booleanParam(name: 'RUN_BACKEND_TESTS', defaultValue: true, description: 'Run backend tests')
    booleanParam(name: 'RUN_FRONTEND_TESTS', defaultValue: true, description: 'Run frontend tests')
    
  }
  environment {
    GIT_SSL_NO_VERIFY = 'true'
  }
  stages {
    stage('TEsts') {
      parallel {
        stage('Backed Tests') {
          when { expression { params.RUN_BACKEND_TESTS } }
          agent { kubernetes { yaml podTemplate(
            memReq: '512Mi',
            memLim: '1Gi',
            cpuReq: '500m',
            cpuLim: '1'
          ) }}
          steps {
            runTests(testType: "backend")
          }
          post {
            always {
              stepI(XUNIT_CONFIG)
            }
          }
        }
        stage('Frontend Tests') {
          when { expression { params.RUN_FRONTEND_TESTS } }
          agent { kubernetes { yaml podTemplate(
            memReq: '512Mi',
            memLim: '1Gi',
            cpuReq: '500m',
            cpuLim: '1'
          ) }}
          steps {
            runTests(testType: "frontend")
          }
          post {
            always {
              stepI(XUNIT_CONFIG)
            }
          }
        }
      }

    }
  }
}

def podTemplate(Map overrides = [:]) {
  // Defautl resources
  @Field static final String DEFAULT_MEM_REQ = '512Mi'
  @Field static final String DEFAULT_MEM_LIM = '1Gi'
  @Field static final String DEFAULT_CPU_REQ = '500m'
  @Field static final String DEFAULT_CPU_LIM = '1'

  // Get values from overrides or use defaults
  String memReq = overrides.get('memReq', DEFAULT_MEM_REQ)
  String memLim = overrides.get('memLim', DEFAULT_MEM_LIM)
  String cpuReq = overrides.get('cpuReq', DEFAULT_CPU_REQ)
  String cpuLim = overrides.get('cpuLim', DEFAULT_CPU_LIM)
  String image = overrides.get('image', getImages().ubuntu)
  String gitSSLNoVerify = overrides.get('gitSSLNoVerify', 'true')

  return """\
apiVersion: v1
kind: Pod
spec: 
  containers:
    - name: jnlp
      image: jenkins/inbound-agent:jdk21
      resources:
        requests:
          memory: "256Mi"
          cpu: "100m"
        limits:
          memory: "512Mi"
          cpu: "500m"
      env:
        - name: GIT_SSL_NO_VERIFY
          value: ${gitSSLNoVerify}
    - name: worker
      image: ${image}
      tty: true
      imagePullSecrets:
      - name: my-artifactory-secret
      resources:
        requests:
          memory: ${memReq}
          cpu: ${cpuReq}
        limits:
          memory: ${memLim}
          cpu: ${cpuLim}
      volumeMounts:
        - name: workspace
          mountPath: /workspace
      env:
        - name: GIT_SSL_NO_VERIFY
          value: ${gitSSLNoVerify}
    volumes:
      - name: workspace
        emptyDir: {}
"""

// Lets run tests
def runTests(String testType) {

  container('worker') {
    String repoUrl = scm.userRemoteConfigs[0].url
    String credId = scm.userRemoteConfigs[0].credentialsId
    String branch = env.BRANCH_NAME

    withCredentials([usernamePassword(
      credentialsId: credId,
      usernameVariable: 'USER',
      passwordVariable: 'PASSWORD'
    )]) {
      sh """
        #!/bin/bash
        set -e
        git -c credential.helper='!f() { echo "username=\${USER}"; echo "password=\${PASSWORD}"; }; f' clone --depth 1 --single-branch --branch '${branch}' '${repoUrl}' .
      """
    }

    // checkout([
    //   $class: 'GitSCM'
    //   branches: scm.branches,
    //   userRemoteConfigs: scm.userRemoteConfigs
    //   extensions: [
    //     [$class: 'CloneOption', depth: 1, shallow: true, noTags: true],
    //     [$class: 'CleanBeforeCheckout']
    //   ]
    // ])

    echo "Running ${testType} tests"
    sh """
      # Add your test commands here
    """
  }
}
```
- Once you have pushed the branch, trigger the build on Jenkins, monitor the pods in the Kubernetes cluster to ensure they are running correctly.


### 4.4 Whats Helm ? How do you install monitoring system (Prometheus + Grafana) for your kubernetes epherphermal jenkins agents?
- Yes, i do understand the buzz word `helm`. Understand like, If K8s is a OS, Helm is app store (or package manager).
- The Problem: `Yaml Hell`
  - Think about what it takes to install a complex application on kubernetes from scratch, you cant just run one command, instead you have to write and apply dozen of yaml files like deployments, services, configmaps, secrets, etc.
  - And If you download someone else's 2000 lines of yaml to do this, you have to manually hunt through it to change the passwords, storage limits, and image versions.
  
- The Solution: `Helm`
  - Helm fixes this by bundling all those seperate yaml files together into a single package manager called `Chart`
  - Instead of hardcoding the values (like password's etc), directly into the yaml, the creator of helm chart replaces them with variables.
  - So, when you want install app using helm, we mainly interact with 3 components
    - **Chart:** The master blueprint, contains all the blank yaml templates for applications.
    - **values.yaml:** It is your control panel, a single, simple file where you define your specific variables. 
    - **The Release:** When you run the install command, helm takes `values.yaml`, injects your variables inot the chart's templetes, generates the final raw yaml in memory and pushes it to the kubernetes.

- Ok, that's great. Still have some silly questions in hand ?
  - Do you aware how do even install helm ?
    - lets refer this to install using [apt](https://helm.sh/docs/intro/install/#from-apt-debianubuntu)
    - Once done, lets do `helm version` to confirm the installation.
  - How do we find the helm packages ??
    - There is official, central search engine for every helm chart available, called [Artifact Hub](https://artifacthub.io/)
    - We can also use command line instead, but here in this case, each chart reffered as `hub`, and folder where it stays known to be a `repo`
    - But by default, helm doesn't know where any charts are stored. We have to add repositoroy on our own.
    - In this case, im adding repo by referring the [prometheus community](https://prometheus-community.github.io/helm-charts/)
    - `helm repo add prometheus-community https://prometheus-community.github.io/helm-charts`
    - `helm repo update`
    - `helm search hub prometheus`
  - How do we also know what values (values.yaml) to be given to helm command at the time of installing ?
    - Every chart creator is required to provide a default `values.yaml` file that acts as giant instruction manual, complete with comments explaining what every single varaible does.
    - To check on values of a particualr chart
    - `helm show values prometheus-community/kube-prometheus-stack > my-custom-values`
    - Once done, you can use this file to override the default values like password, limits etc at the time of app installation.
    - Here in this case, we are worried about value of `adminPassword` under `grafana` section.
  
  - How do we install the app using helm ?
    - `helm install my-monitoring prometheus-community/kube-prometheus-stack -f my-custom-values --namespace monitoring --create-namespace`
    - or if your upgrading the existing release with new values ?
    - `helm upgrade my-monitoring prometheus-community/kube-prometheus-stack -f my-custom-new-values --namespace monitoring`
    - Let Port forward svc/my-monitoring-grafana to access the grafana UI 
    - `kubectl port-foward svc/my-monitoring-grafana 3000:80 -n monitoring --address 0.0.0.0 &`
    - `curl http://localhost:3000` or use the control node ip and port 3000 to access the grafana UI
  
  - Port forwarding the service is for 5min dubug, its not at good for prod envs. Either we should use NodePort or Ingress.
    ```bash
    kubectl get svc -n monitoring
    kubectl edit svc monitoring-stack-grafana -n monitoring
    # Change 'type' from ClusterIP to NodePort in specs, add 'nodePort' in spec:ports
    # yeah save it.and verify the node port in the svc
    kubectl get svc -n monitoring monitoring-stack-grafana
    ```

  - If i wanted  inspect the yaml (or app) installed by helm, where do i need go and check ?
    - `helm get manifest my-monitoring -n monitoring`
    - `helm get values my-monitoring -n monitoring`
    - `helm get all my-monitoring -n monitoring`
  - How do we uninstall the app using helm ?
    - `helm uninstall my-monitoring -n monitoring` 
  - How do we list the releases installed by helm ?
    - `helm list -n monitoring`

### 4.5 All the terms Helm, artifacthub.io, CNCF, Kubernetes related ?
- To clear the sky completely, yes they are intimately related. They were built in a very specific order, usually because the previous invention created new, massive headache that needed solving..
- **Kubernetes (Born 2014)** - The OS.
  - Devoloped by Google (originally  based on their internal tool called Borg)
  - The Problem it solved: Running docker containers across multiple servers was a nightmare, kubernetes automated  the networking, restarting, and scaling of containers.
  - The Drawback: It was incredibly complex. Also because Google owned it, other massive companies like Microsoft, AWS were hesitant to adopt it, fearing google would change the rules or charge them for it later.
- **The CNCF (Born 2015) - The Switzerland of Tech**
  - Developed by Linux Foundation (with backing from Google, Microsoft, AWS etc)
  - The Problem it solved: To fix the trust issue, Google completely gave away the trademark and code for Kubernetes to a brand-new, non-profit organization called `Cloud Native Computing Foundation (CNCF)`
  - What is it today: The CNCF is essentially the neutral "government" or "umbrella" for cloud software. If a project is accepted by CNCF, it guarantees that no single corporation owns it, and it will remain open-source forever.
- **Helm (Born 2016) - The Package Manager**
  - Developed by a startup called `Deis` (which was later bought by microsoft) and donated to the CNCF.
  - The Problem it solved: As Kubernetes got widly popular, people realized that writing 2000 lines of raw kubernetes yaml by hand just to install a database was really awful (The YAML Hell). Helm was created to template that yaml into reusable "Charts"
  - The Drawback: Anyone colud make a helm chart, but there was no central place to put them, people were just hosting them on random github repositories. Finding a trustworthy chart was like trying to download software in the 1990s - you just had to trust random links on forums.
- **Artifcat Hub (Born 2020) - The Search Engine**
  - Developed by CNCF
  - The problem it solved: The CNCF realized the community needed a safe, centralized "Google Search" for helm charts and other kubernetes tools.
  - What it is today: Aritifact hub is a official CNCF registry, it scans thousands of github repos, checks the helm chars for securiy vlunerabilites, verifies the official publishers, and gives you one single website (https://artifacthub.io/) to search them.



### 4.6 What if the nodes in the cluster loaded heavily in such way that they stop responding to stop  responding to the SSH or send its `kubelet` heartbeat?
- This phenamena is known be a **Resource Starvation**
- Sometimes pods essentially become a massive black hole, sucking up 100% of CPU or RAM. Because it consuemed everything, the underlying linux OS didnt even have enough cpu time left to process your SSH keystrokes, and the `kubelet` didnt have enough power to tell the master node it was still alive.
- To prevent this from ever happening again, you do not just set limits on the node; you attack the problem from tow different angles.
- 1. **Defence1: Pod Resource Limits**
- 2. **Defence2: Kubelet Resource Reservation**
  - Kubernetes has a brillient built-in feature for this called **Node Allocatable Resources**
  - You configure the `kubelet` on your worker nodes to physically fence off a chunk of CPU and RAM that pods are legally not allowed to touch.
  - Configure below setting in each of worker nodes.
  ```bash
  sudo vi /var/lib/kubelet/config.yaml
  
  #Add below configuration:

  systemdReserved:
    cpu: "500m"
    memory: "1Gi"
  kubeReserved:
    cpu: "500m"
    memory: "1Gi"
  evictionHard:
    memory.available: "500Mi"
    nodefs.available: "10%"
  enforceNodeAllocatable:
    - pods

  #Restart the kubelet service:
  sudo systemctl restart kubelet
  ```


## 5. Specific Debugs 
### 5.1 A Control node suddenly went offline, with its kubelet service throwing error as `Apr 21 12:56.15 control_node1 kubelet[2111888]: E0421 12:56.152886 2111888 kubelet.go:3336] "No need to create a mirror pod, since failed to get node info from the cluster" err="node \">
- If you look at this error, it's easier to understand the root cause behind the disconnectivity of the control node.
- The kubelet service is throwing an error that it's not able to find current node information in the cluster info. This means that according to the cluster info from the API server, the node is not present in the cluster. Either someone modified the configuration or **changed the current hostname** of the server.
- If you have changed the hostname, please change it back to the original one as it was when the cluster was created and restart the kubelet service. 
- But if you think these may not be root cause. we can go over few possibilities.
#### Suspect1: Expired Certificates
- If the cluster was built using `kubeadm` exactly one year ago, your internal control plane certificates have likey expired. The API refuses to start if the TLS certs are not valid
  ```bash
  # check certificate expiry
  sudo kubeadm certs check-expiration
  # if certificates are expired, renew them
  sudo kubeadm certs renew all
  ```
#### Suspect2: It can be anything..
- Here we cant make use of `kubectl` as control node is completely down. We have to inspect the raw API server logs using `crictl`
  ```bash
  # Lets understand why api server is not responding ...
  sudo crictl logs --tail 100 --follow $(sudo crictl ps -a -q --name kube-apiserver | head -n 1)

  # If incase above api server shows issues with some kind 
  # 'Error while dialing: dial tcp 127.0.0.1:2379: operatin was canceled.
  # authentication error: `unable to authenticatee the request`, 
  # err=[invalid bearer token, service account has been invalidated]
  # The specific port, 2879 is default port for etcd (a k8s configuration db).
  # It means API server somehow, not able to authenticate with the etcd.
  # Why ? is etcd got down due to diskspace issue ? or OOM (out of memory) ? lets verify
  sudo df -h
  sudo free -h
  sudo crictl ps -a | grep etcd
  sudo crictl logs --tail 100 --follow $(sudo crictl ps -a -q --name etcd | head -n 1)

  # Incase if we see any error like `port bind address already in use`, 
  # it means either muliple instance of etcd is running, or some other process is using the same port.
  # Anyway lets restart the db again. We will not loose the data here as in 
  # harddrive at `/var/lib/etcd`.
  sudo crictl rm -f $(sudo crictl ps -a -q --name etcd)

  # If we do not see any error in the etcd logs, we do still see the apiserver auth issue.
  # It may be a reason of TLS certificate mismatch. 
  # Are both apiserver and etcd using same certs paths ?
  sudo cat /etc/kubernetes/manifests/etcd.yam | grep -i "trusted-ca-file\|cert-file\|key-file"
  sudo cat /etc/kuebernetes/manifests/kube-apiserver.yaml | grep -i "trusted-ca-file\|cert-file\|key-file"
  # If they are using same certs path. We can also supsect chance of curruption of crytographic contents
  # in the certificates file, Lets verify here..
  # whether apiserver server is presenting correct client cert to the etcd server...
  # etcd server has its own issuer placed at /etc/kubernetes/pki/etcd/ca.crt
  sudo openssl x509 -in /etc/kubernetes/pki/etcd/ca.crt -noout -text | grep -i "issuer"
  sudo openssl x509 -in /etc/kubernetes/pki/etcd/server.crt -noout -text | grep -i "issuer"
  sudo openssl x509 -in /etc/kubernetes/pki/apiserver-etcd-client.crt -noout -text | grep -i "issuer"
  # from above output, we have to understand issuer of each cert and make sure they are same.
  # we can also verify the trust of the certs using `openssl verify` command.
  sudo openssl verify -CAfile /etc/kubernetes/pki/etcd/ca.crt /etc/kubernetes/pki/etcd/server.crt
  sudo openssl verify -CAfile /etc/kubernetes/pki/etcd/ca.crt /etc/kubernetes/pki/apiserver-etcd-client.crt
  # If you suspect here something is really broken, lets regenerate the client cert for the apiserver.
  sudo mv /etc/kubernetes/pki/apiserver-etcd-client.crt /tmp/
  sudo mv /etc/kubernetes/pki/apiserver-etcd-client.key /tmp/
  sudo kubeadm init phase certs apiserver-etcd-client
  # now verify the trust again
  sudo openssl verify -CAfile /etc/kubernetes/pki/etcd/ca.crt /etc/kubernetes/pki/apiserver-etcd-client.crt
  # reboot api server
  sudo crictl stop $(sudo crictl ps -a -q --name kube-apiserver)
  sudo crictl start $(sudo crictl ps -a -q --name kube-apiserver)
  ```


### 5.2 ok, do you aware which type of distributed storage system is used by a freshly made vannila K8s cluster ? or what is the default storage class here ? How do we install one ?
- To give a short answer, Out of box, a freshly built k8s cluster has absolutely zero default distributed storage. 
- At this stage, K8s doesnt know how to store persistent data by default, it needs a **CSI (Container Storage Interface) provider.** plugin to be installed.
- Without CSI plugin, the only storage cluster natively understand is `hostPath` or `local` volume. These are strictly tied to physical node. 
- Just like Calico (CNI plugin), we need to install  a storage controller. Once its installed, it wiil automatically create a **Storage Class**. When you create a PVC, k8s will hand that request to the storage plugin, which will dynamically provision the virtual drive.
- **Longhorn** (originally created by Rancher/SUSE) is the absolute gold standard for team dont want to spend months learning complex storage engineeering.
- Here is straightforward guide to install longhorn on the cluster right away..
  - **Step1: The prerequisites.**
    - A CSI needs to actually talk to physical hard drives. Due to this, Linux OS on workers nodes need two standard packages to be installed, so they understand Longhorn storage language (iSCSI and NFS).
    ```bash
    # On every node
    sudo apt update
    sudo apt install open-iscsi nfs-common -y

    # or
    ansible kube_node -i host.ini -m apt -a "name=open-iscsi,nfs-common state=present" -k
    ```
  - **Step2: Install Longhorn.**
    ```bash
    # Install Longhorn
    kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/master/deploy/longhorn.yaml
    # Wait for Longhorn to be ready
    kubectl wait --for=condition=ready pod -l app=longhorn-manager -n longhorn-system --timeout=120s
    ```
  - **Step3: See the Magic (the UI)**
    - Longhorn has built in dashboard. To access it from your network, we have to configure the nodeport for the frontend service of longhorn-ui.
    ```bash
    kubectl patch service longhorn-frontend -n longhorn-system -p '{"spec":{"type":"NodePort","ports":[{"name":"http","port":80,"targetPort":80,"nodePort":30080,"protocol":"TCP"}]}}'
    ```
    - Now you can access the Longhorn UI at `http://<your-k8s-master-ip>:30080`




























