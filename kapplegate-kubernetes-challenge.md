## Challenge
Install vcluster to test upgrades (eg. 1.20 to 1.21 DOKS version) of your cluster. With a virtual cluster, you can create a new Kubernetes cluster inside your existing DOKS cluster, test the application in this new vcluster, and then upgrade your original cluster if everything works well with the new version.

## Procedures

### Setup Kubernetes
After setting up my account and installing the doctl command line (via brew), I was off to get started with getting the infrastrcuture stood up.

First, I created a cluster using the following:
```bash
doctl kubernetes cluster create do-kapplegate --version 1.20.11-do.0  --region nyc1
```
This gives me 3 nodes at a version that I can upgrade from (to 1.21 later in the evaluation).

Normally I'd now need to grab the kubeconfig so that I can interact w/ the cluster from kubectl. However, it looks like that's included in the CLI command above (nice QoL feature). If I did need it, I could grab the config and add it to my local kubeconfig with:
```bash
doctl kubernetes cluster kubeconfig save do-kapplegate
```

One other shortcut I do to help with CLI interactions w/ kubectl is to alias it to `k`:
```bash
alias k=kubectl
```

Now I can check that I can see the cluster from the cli
```bash
k get nodes
NAME                               STATUS   ROLES    AGE     VERSION
do-kapplegate-default-pool-ubqz2   Ready    <none>   4m33s   v1.20.11
do-kapplegate-default-pool-ubqzl   Ready    <none>   2m35s   v1.20.11
do-kapplegate-default-pool-ubqzt   Ready    <none>   2m47s   v1.20.11
```

### Setup VCluster

First I'd need to install the VCluster CLI on my M1 (ARM) mac:

```bash
curl -s -L "https://github.com/loft-sh/vcluster/releases/latest" | sed -nE 's!.*"([^"]*vcluster-darwin-arm64)".*!https://github.com\1!p' | xargs -n 1 curl -L -o vcluster && chmod +x vcluster;
sudo mv vcluster /usr/local/bin;
```

I'd also need to grab Helm to help with the install:
```bash
brew install helm
```

Then I could go ahead and create the cluster:
```bash
% vcluster connect vcluster1 -n cluster1
[info]   Waiting for vcluster to come up...
[info]   Waiting for vcluster LoadBalancer ip...
[info]   Using vcluster vcluster1 load balancer endpoint: 157.230.203.55
[done] √ Virtual cluster kube config written to: ./kubeconfig.yaml. You can access the cluster via `kubectl --kubeconfig ./kubeconfig.yaml get namespaces`
```
Test cluster access
```bash
% k get nodes --kubeconfig ./kubeconfig.yaml
NAME                               STATUS   ROLES    AGE     VERSION
do-kapplegate-default-pool-ubqzl   Ready    <none>   3m44s   v1.20.11+k3s2
```

Now I threw a sample web application I'll use (Yelb) on the vcluster:
```bash
% k create -f https://github.com/mreferre/yelb/raw/master/deployments/platformdeployment/Kubernetes/yaml/yelb-k8s-loadbalancer.yaml --kubeconfig ./kubeconfig.yaml
service/redis-server created
service/yelb-db created
service/yelb-appserver created
service/yelb-ui created
deployment.apps/yelb-ui created
deployment.apps/redis-server created
deployment.apps/yelb-db created
deployment.apps/yelb-appserver created
```
Grab the LB IP to test the app:
```bash
% k get svc -A --kubeconfig ./kubeconfig.yaml
NAMESPACE     NAME             TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)                  AGE
default       kubernetes       ClusterIP      10.245.128.120   <none>          443/TCP                  10m
kube-system   kube-dns         ClusterIP      10.245.183.29    <none>          53/UDP,53/TCP,9153/TCP   10m
default       redis-server     ClusterIP      10.245.203.192   <none>          6379/TCP                 3m38s
default       yelb-db          ClusterIP      10.245.194.74    <none>          5432/TCP                 3m38s
default       yelb-appserver   ClusterIP      10.245.85.253    <none>          4567/TCP                 3m38s
default       yelb-ui          LoadBalancer   10.245.81.107    167.172.2.132   80:32014/TCP             3m38s
```

Now to try the upgrade of the host cluster
```bash
doctl kubernetes cluster upgrade do-kapplegate --version 1.21.5-do.0
Notice: Upgrading cluster to version 1.21.5-do.0
```

The upgrade did take quite awhile (3+ hours but I let it go over night). However, once it completed I was able to see the cluster was updated:
```bash
% k get nodes
NAME                               STATUS   ROLES    AGE   VERSION
do-kapplegate-default-pool-ubjj7   Ready    <none>   11h   v1.21.5
do-kapplegate-default-pool-ubjjc   Ready    <none>   11h   v1.21.5
do-kapplegate-default-pool-ubjju   Ready    <none>   11h   v1.21.5
```

The Vcluster was up the whole time and was available without skipping a beat. Also, since the controller of the vcluster is runnign as a pod on the worker nodes, even while the host cluster was unavailable during the upgrade, the guest cluster was fully responsive. Very cool stuff. 

## Conclusion
Both the Digital Ocean Kubernetes offering and the vcluster stack performed fantastic. Both are worthy of consideration for use-cases that can take advantage of them. I am finding a large amount of cluster sprawl in customers that I talk to. Especially when environments make it so easy to spin up clusters, the increased attack surface, resource under-utilization, and complexity compound exponentially. 

