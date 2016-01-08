# k8s charm

This is a repository for a k8s demo using layers to build a complex Kubernetes
charm and Juju to deploy and manage a Kubernetes cluster.

The k8s charm is built from the following reusable layers.

* [docker charm layer](https://github.com/juju-solutions/layer-docker) ->
<https://github.com/juju-solutions/layer-docker>
* [flannel charm layer](https://github.com/chuckbutler/flannel-layer) ->
<https://github.com/chuckbutler/flannel-layer>
* [etcd interface layer](https://github.com/chuckbutler/interface-etcd) ->
<https://github.com/chuckbutler/interface-etcd>
* [tls charm layer](https://github.com/mbruzek/layer-tls) ->
<https://github.com/mbruzek/layer-tls>
* [tls interface layer](https://github.com/mbruzek/interface-tls) ->
<https://github.com/mbruzek/interface-tls>
* [k8s charm layer](https://github.com/mbruzek/layer-k8s) ->
<https://github.com/mbruzek/layer-k8s>

## Steps to build and deploy a Kubernetes cluster:

```bash
git clone https://github.com/mbruzek/layer-k8s.git
cd layer-k8s
export JUJU_REPOSITORY=$HOME/charms
charm build -l DEBUG --force

cd ..
git clone https://github.com/mbruzek/k8s-demo.git && cd k8s-demo
juju deploy bundle.yaml
```
Notice how the new `juju deploy` command can take a v4 bundle!

What is going on behind the scenes is:
* The charm is installing the Docker container runtime.
* The charm is setting up networking with flannel, using etcd.
* The leader is creating a self signed Certificate Authority and setting that
on Juju's leader key value store.
* The followers are submitting a Certificate Signing Request (CSR) to the leader
using the peer relation.
* The leader picks up the CSR and signs the request sending a signed server
certificate back to the follower.
* The certificates are copied to the right spot and used by the Kubernetes
Docker containers to provide secure communication with clients and peers.
* The Kubernetes containers are being started and they also use etcd for
communication.
* The leader builds a package with the client tls bits (cerficiate and key)
along with the CA, and kubectl so Juju users can communicate securely with the
Kubernetes cluster.

The cluster deployment may take some time (8 to 10 minutes), please be paitent
and watch the `juju status` messages or the `juju debug-log` output until all
three nodes of the Kubernetes cluster are completely installed.  The
**kubectl_package.tar.gz** is generated on the leader after the relation to etcd
and after all the software is installed.

## Download kubectl

To communicate with the Kubernetes cluster you must first download certificate
credentials.

```bash
juju scp k8s/0:kubectl_package.tar.gz .
tar -xzvf kubectl_package.tar.gz
```

Where k8s/0 is the master, if that is not the master, than change the unit to
the master.

## Verify the Kubernetes cluster is up

> NOTE: The config, cert and keys in kubectl_package can be copied to 
$HOME/.kube/ directory to allow you to skip typing --kubeconfig flag on every
kubectl command, just remember to remove this configuration when you tear
down the cluster.

```bash
./kubectl --kubeconfig=config get pods
```
Shows the [pods](http://kubernetes.io/v1.0/docs/user-guide/pods.html) in the
Kubernetes cluster. A "pod" is a Kubernetes term for a collection of
applications running with a shared context. A pod models an
application-specific "logical host" in a containerized environment.

With no services started, this should return 3 default pods.

```bash
./kubectl --kubeconfig=config get nodes
```
Shows the [nodes](http://kubernetes.io/v1.0/docs/design/architecture.html) in
the Kubernetes cluster. The Kubernetes node has the services necessary to run
application containers and be managed from the master systems. The bundle calls
for 3 units, so you should see 3 units.

```bash
./kubectl --kubeconfig get services
```
Shows the [services](http://kubernetes.io/v1.0/docs/user-guide/services.html)
in the Kubernetes cluster. A Kubernetes Service is an abstraction which defines
a logical set of Pods and a policy by which to access them - sometimes called a
micro-service.

```bash
./kubectl --kubeconfig get rc
```
Shows the
[replication controllers](http://kubernetes.io/v1.0/docs/user-guide/replication-controller.html)
in the Kubernetes cluster.  A replication controller ensures that a specified
number of pod "replicas" are running at any one time. If there are too many,
it will kill some. If there are too few, it will start more.

There are currently no replication controller resources.

## Start Kubernetes services.

```bash
./kubectl --kubeconfig create -f frontend-controller.yaml
```
Starts the controller for the 2048 service.
```
./kubectl --kubeconfig create -f frontend-service.yaml
```
Starts the 2048 Kubernetes service (docker containers).
