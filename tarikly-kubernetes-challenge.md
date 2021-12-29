## How to deploy an internal container registry on DigitalOcean Kubernetes Cluster

You gonna need:
- Digital Ocean account (https://www.digitalocean.com/)
- kubectl (https://kubernetes.io/docs/tasks/tools/)

*Attention! As referenced here (https://github.com/ContainerSolutions/trow/issues/283#issuecomment-1002388790), to make Trow working you need a Kubernetes DO working on versions prior to 1.20.X*

![image](https://user-images.githubusercontent.com/8581805/147695936-907d1cbb-b35a-464c-bc2c-d7cbc285fd31.png)

Onyou machine check Digital Ocean Kubernetes version:

![image](https://user-images.githubusercontent.com/8581805/147696260-fd13a187-eef3-4dd1-9448-2643a398fde6.png)



Clone trow repository
```
git clone https://github.com/ContainerSolutions/trow
```
Set namespace and create Trow resources
```
cd trow/quick-install
kubectl create ns registry
namespace=registry
sed "s/{{namespace}}/${namespace}/" trow.yaml | kubectl apply -f -
```

Approving certificate (it may take up to 2 minutes so you can appove the Certificate Signing Request created on previous step.)
```
kubectl certificate approve "trow.${namespace}"
```
Define certfile var as /tmp/ca-cert.XXXXXX
```
cert_file=$(mktemp /tmp/ca-cert.XXXXXX)
```

Saving cluster certficate as trow-ca-cert
```
kubectl config view --raw --minify --flatten \
  -o jsonpath='{.clusters[].cluster.certificate-authority-data}' \
  | base64 --decode | tee -a $cert_file
kubectl create configmap trow-ca-cert --from-file=cert=$cert_file \
  --dry-run=client -o json | kubectl apply -n "$namespace" -f -
```  
Copy certificate to cluster nodes
```
./copy-certs.sh "$namespace"
```
To test your registry from outside the cluster install ed and run configure-host.sh to setup Docker in your host:
```
apt-get install ed -y
./configure-host.sh --namespace="$namespace" --add-hosts;
```

Testing

Busybox:
```
docker pull busybox:latest
docker push trow.registry:31000/busybox:v1
kubectl run -i --tty busybox --image=trow.registry:31000/busybox:v1 -- sh
```

Nginx:
```
docker pull nginx:latest
docker tag nginx:latest trow.registry:31000/mynginx:test
docker push trow.registry:31000/mynginx:test
kubectl run trow-test --image=trow.registry:31000/mynginx:test
```
![image](https://user-images.githubusercontent.com/8581805/147698753-dd2af12d-1743-4add-9cba-586dcfc4fefd.png)


Resources:

https://github.com/digitalocean/Kubernetes-Starter-Kit-Developers

https://github.com/ContainerSolutions/trow


digitalocean-kubernetes-challenge
https://www.digitalocean.com/community/pages/kubernetes-challenge





## Name of Project 
* Deploy an internal container registry  
 
## Link to Your Github or Gitlab Repo
* https://github.com/tarikly/digitalocean-kubernetes-challenge

## Link to Your Project Writeup
* https://github.com/tarikly/digitalocean-kubernetes-challenge/blob/main/README.md

## Contact Info
* Tarikly
* https://www.linkedin.com/in/tarikly/

