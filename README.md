# kubernetes-the-kubeadm-way

Kubernetes on Vagrant using kubeadm.

## Requirements

* VirtualBox
* Vagrant

### Installing Requirements

#### macOS

Using `homebrew`

```bash
brew cask install virtualbox virtualbox virtualbox-extension-pack

brew cask install vagrant
```

## Using

### Provisioning with Vagrant

The default setup sets up a loadbalancer node, three master nodes and two worker nodes.

To stand up a new environment

```bash
vagrant up
```

Connecting to the nodes `vagrant ssh <node>`. Example:

```bash
vagrant ssh master-1
```

### Kubernetes setup

It's all set up using Vagrant provisoners! Yay!

### Smoke Tests

Let's set up an NGINX deployment and service as a smoke test. This can be run from `master-1` node or using the `admin.kubeconfig` in the repository folder after provisioning.

```bash
kubectl create deployment nginx --image=nginx
## deployment.apps/nginx created

kubectl scale deployment nginx --replicas=3
## deployment.apps/nginx scaled

kubectl expose deployment nginx --port=80 --target-port=80 --type NodePort
## service/nginx exposed

kubectl get service nginx -o yaml | sed -E "s/nodePort\:.*/nodePort: 30080/" | kubectl apply -f -
## Warning: kubectl apply should be used on resource created by either kubectl create --save-config or kubectl apply
## service/nginx configured

kubectl get pod,deployment,service
## NAME                        READY   STATUS    RESTARTS   AGE
## pod/nginx-f89759699-7lr85   1/1     Running   0          3m37s
## pod/nginx-f89759699-gn97b   1/1     Running   0          3m30s
## pod/nginx-f89759699-l5bjt   1/1     Running   0          3m30s
##
## NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
## deployment.apps/nginx   3/3     3            3           3m37s
##
## NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)        AGE
## service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP        43m
## service/nginx        NodePort    10.96.0.67   <none>        80:30080/TCP   104s

curl http://worker-1:30080 && curl http://worker-2:30080
## <!DOCTYPE html>
## <html>
## <head>
## <title>Welcome to nginx!</title>
## <style>
##     body {
##         width: 35em;
##         margin: 0 auto;
##         font-family: Tahoma, Verdana, Arial, sans-serif;
##     }
## </style>
## </head>
## <body>
## <h1>Welcome to nginx!</h1>
## <p>If you see this page, the nginx web server is successfully installed and
## working. Further configuration is required.</p>
##
## <p>For online documentation and support please refer to
## <a href="http://nginx.org/">nginx.org</a>.<br/>
## Commercial support is available at
## <a href="http://nginx.com/">nginx.com</a>.</p>
##
## <p><em>Thank you for using nginx.</em></p>
## </body>
## </html>
## ...

# Let's generate some logs and then check logging. This verifies kube-apiserver to kubelet RBAC permissions.
for (( i=0; i<50; ++i)); do
    curl http://worker-1:30080 &>/dev/null && curl http://worker-2:30080 &>/dev/null
done
kubectl logs deployment/nginx
## Found 3 pods, using pod/nginx-f89759699-7lr85
## 10.32.0.1 - - [26/Mar/2020:13:48:07 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.58.0" "-"
## 10.32.0.1 - - [26/Mar/2020:13:55:38 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.58.0" "-"
## 10.44.0.0 - - [26/Mar/2020:13:55:38 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.58.0" "-"
## ...
```

### Conclusion

Such Awesome! Much k8s!

## Cleanup

Destroy the machines and clean up temporary files from the repository.

```bash
vagrant destroy -f
git clean -xf
```

## Troubleshooting

For some reason I ran into an issue where I could not reach the guests via the network anymore. The reason was that the route was not being created on the host.

To create a route manually:

```bash
## You may need to verify the correct vboxnet interface. eg vboxnet0, vboxnet1 etc.
## You can find the interface associated to the network range with `VBoxManage list hostonlyifs`
sudo route -nv add -net 192.168.10 -interface vboxnet1
```
