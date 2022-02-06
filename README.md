# Knative and microk8s

Install [multipass](https://multipass.run)
```
brew install multipass
```

Install `hyperkit` or `qemu`, do use virtual box it doesn't allow access from the host network bridge by default.

For qemu install `libvirt` and set as default driver
```
brew install libvirt
sudo multipass set local.driver=qemu
```

For hyperkit install `hyperkit` and set as default driver
```
brew install hyperkit
sudo multipass set local.driver=hyperkit
```

Using multipass create a new ubuntu VM

Create a multipass vm with 3 CPU, 2 GB, and 8GB of disk

```
multipass launch -n knative -c 3 -m 2G -d 8G
```

Set the primary name to `knative` to avoid always typing the name of the vm

```
multipass set client.primary-name=knative
```

Login into the vm
```
multipass shell
```

Install [microk8s])(https://microk8s.io/docs/getting-started) or from [github/ubuntu/microk8s](https://github.com/ubuntu/microk8s)
```
sudo snap install microk8s --classic
```

Join the group `microk8s`
```
sudo usermod -a -G microk8s $USER
sudo chown -f -R $USER ~/.kube
```

Logout to refresh groups
```
exit
```

Login into the vm again
```
multipass shell
```

Check status
```
microk8s status --wait-ready
```

Check access
```
microk8s kubectl get nodes
```

Set alias
```
alias kubectl='microk8s kubectl'
alias k='kubectl'
```

Enable dns
```
microk8s enable dns
```

Install Knative Serving from [knative.dev](https://knative.dev/docs/install/serving/install-serving-with-yaml)
TLDR;
```
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.2.0/serving-crds.yaml
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.2.0/serving-core.yaml

kubectl apply -f https://github.com/knative/net-kourier/releases/download/knative-v1.2.0/kourier.yaml
kubectl patch configmap/config-network \
  --namespace knative-serving \
  --type merge \
  --patch '{"data":{"ingress.class":"kourier.ingress.networking.knative.dev"}}'
```

Check the status of the knative network layer load balancer
```
kubectl --namespace kourier-system get service kourier
```
If the `EXTERNAL-IP` is in `pending` then you need a load balancer in your kubernetes cluster

You can use the metalb addon, with a small range of ip addresses, use `ip a` to inspect the ip address currently assign and assign IPs on the same subnet
```
microk8s enable metallb:192.168.205.250-192.168.205.254
```
> Yes, I know this is a hack but allows me to access the cluster from the host macOS 😅

Check again
```
kubectl --namespace kourier-system get service kourier
```
Output should look like this
```
NAME      TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)                      AGE
kourier   LoadBalancer   10.152.183.31   192.168.205.16   80:32564/TCP,443:32075/TCP   7m17s
```

Check knative is up
```
kubectl get pods -n knative-serving
```

Configure Knative DNS
```
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.2.0/serving-default-domain.yaml
```

Install the `kn` CLI
```
sudo curl -o /usr/local/bin/kn -sL https://github.com/knative/client/releases/download/knative-v1.2.0/kn-linux-amd64
sudo chmod +x /usr/local/bin/kn
```

Copy the kubeconfig to `$HOME/.kube/config`
```
microk8s config > $HOME/.kube/config
```

Create your first knative service
```
kn service create nginx --image nginx --port 80
```

Get the url of your new service
```
kn service describe nginx -o url
```

Curl the url
```
curl $(kn service describe nginx -o url)
```

You sould see the nginx output
```
Thank you for using nginx.
```

List the pods for your service
```
kubectl get pods
```

After a minute your pod should be deleted automatically (ie scale to zero)
```
NAME                                      READY   STATUS        RESTARTS   AGE
nginx-00001-deployment-5c94d6d769-ssnc7   2/2     Terminating   0          83s
```

Access the url again
```
curl $(kn service describe nginx -o url)
```

