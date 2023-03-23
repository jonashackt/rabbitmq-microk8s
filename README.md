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





## Install RabbitMQ cluster operator

We want to check out the RabbitMQ operator:

https://operatorhub.io/operator/rabbitmq-cluster-operator

There's a great quickstart guide covering all the steps we need:

https://www.rabbitmq.com/kubernetes/operator/quickstart-operator.html



### 0. Install rabbitmq kubectl plugin via krew

Luckily there's a RabbitMQ kubectl plugin which will make working with RabbitMQ on Kubernetes much easier.

To install it we need to have [`krew` - the package manager for kubectl plugins](https://github.com/kubernetes-sigs/krew) in place. If you don't already have it installed, take [the following steps on a Mac](https://krew.sigs.k8s.io/docs/user-guide/setup/install/) (see the [docs for other OSses]https://krew.sigs.k8s.io/docs/user-guide/setup/install/)):

```shell
(
  set -x; cd "$(mktemp -d)" &&
  OS="$(uname | tr '[:upper:]' '[:lower:]')" &&
  ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/\(arm\)\(64\)\?.*/\1\2/' -e 's/aarch64$/arm64/')" &&
  KREW="krew-${OS}_${ARCH}" &&
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/${KREW}.tar.gz" &&
  tar zxvf "${KREW}.tar.gz" &&
  ./"${KREW}" install krew
)
```

Then add the following line to your `.bashrc` or `.zshrc`:

```shell
export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"
```

Now `kubectl krew` should be ready to use!

Finally install the rabbitmq kubectl plugin with:

```shell
kubectl krew install rabbitmq
```


### 1. Install the RabbitMQ cluster operator Helm

In the docs https://www.rabbitmq.com/kubernetes/operator/install-operator.html#helm-chart there is a reference to the bitnami Helm chart https://bitnami.com/stack/rabbitmq-cluster-operator/helm. We can install it using our own custom `Chart.yaml` to pin the version ([as described here](https://stackoverflow.com/questions/71765471/helm-how-to-pin-version-make-it-manageable-by-renovate-on-github/71765472#71765472)) in order to make this managable via Renovate bot. The `Chart.yaml` resides in `rabbitmq/install`:

```yaml
apiVersion: v2
type: application
name: rabbitmq-cluster-operator
version: 0.0.0 # unused
appVersion: 0.0.0 # unused
dependencies:
  - name: rabbitmq-cluster-operator
    repository: https://charts.bitnami.com/bitnami
    version: 3.2.7
```

Now we can install the RabbitMQ cluster operator with the following commands:

```shell
helm dependency update rabbitmq/install
helm upgrade -i rabbitmq-cluster-operator rabbitmq/install
```


### 2. Watch operator starting up

```shell
kubectl wait --for=condition=Ready pod -l app.kubernetes.io/name=rabbitmq-cluster-operator,app.kubernetes.io/instance=rabbitmq-cluster-operator --namespace default --timeout=120s
```