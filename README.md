# rabbitmq-microk8s
Example project showing how to run RabbitMQ on MicroK8s


## Install MicroK8s

On Linux - especially Ubuntu - installation of MicroK8s is super easy: https://microk8s.io/docs/getting-started

If you're on a Mac, then [the following steps apply](https://microk8s.io/docs/install-macos):

```shell
brew install ubuntu/microk8s/microk8s
```

On MacOS MicroK8s is based on [Multipass](https://multipass.run/), which manages and runs Ubuntu VMs on every OS' native hypervisor (for Mac this is QEMU and HyperKit). So the next command will prompt you to download Multipass:

```shell
microk8s install
```

After successfull completion there should be something printed something like this:

```shell
2023-03-22T09:13:05+01:00 INFO Waiting for automatic snapd restart...
microk8s (1.26/stable) v1.26.1 from Canonical✓ installed
microk8s-integrator-macos 0.1 from Canonical✓ installed
MicroK8s is up and running. See the available commands with `microk8s --help`.
```

Now we should turn on some services that we need:

```shell
microk8s enable dns
```

## Access MicroK8s

> MicroK8s bundles its own version of kubectl for accessing Kubernetes.

Just checking:

```shell
microk8s kubectl get nodes
```

This works, but you may want to use your own `kubectl` including [kubectx or kubens](https://github.com/ahmetb/kubectx) and [k9s](https://k9scli.io/). This is also possible (see https://microk8s.io/docs/working-with-kubectl).

First output the `kubeconfig` with:

```shell
microk8s config
```

If you don't have a K8s cluster configured, you can copy the output directly into your K8s config

```shell
mkdir ~/.kube
microk8s config > ~/.kube/config
```

> If you have already configured other Kubernetes clusters, __you should merge the output from the microk8s config with the existing config__ 

Watch out for existing lines like:

```shell
apiVersion: v1
clusters:
## and
contexts:
## and 
current-context: xyz-cluster
## and
kind: Config
preferences: {}
users:
```

You shouldn't override them!





## Install RabbitMQ

We want to check out the RabbitMQ operator:

https://operatorhub.io/operator/rabbitmq-cluster-operator

There are 3 steps to take:

### 1. Install Operator Lifecycle Manager (OLM)

```shell
curl -sL https://github.com/operator-framework/operator-lifecycle-manager/releases/download/v0.24.0/install.sh | bash -s v0.24.0
```