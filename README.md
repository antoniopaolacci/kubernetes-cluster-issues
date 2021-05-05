## Kubernetes-cluster-issues

###### Kubernetes issues related to a cluster setup



If you are having trouble with 

```bat
~ $ kubectl version
Client Version: version.Info{Major:"1", Minor:"18", GitVersion:"v1.18.0", GitCommit:"9e991415386e4cf155a24b1da15becaa390438d8", GitTreeState:"clean", BuildDate:"2020-03-25T14:58:59Z", GoVersion:"go1.13.8", Compiler:"gc", Platform:"linux/arm"}
Unable to connect to the server: net/http: TLS handshake timeout```

or

```bat
~ $ kubectl get po
The connection to the server 192.168.1.65:6443 was refused - did you specify the right host or port?```



You must inspect error on:

```bat
sudo systemctl status kubelet.service```



![]()




```bat
$ curl https://192.168.1.65:6443/api/v1/nodes
curl: (35) OpenSSL SSL_connect: SSL_ERROR_SYSCALL in connection to 192.168.1.65:6443```



**Check the Certs**

After a few restart my kubelet and apiserver, no luck, I realize my kubernetes api-server pod was keep restarting. I went to check log, and it  show me https-related messages.

Apparently port 2379 is belongs to etcd , and its mentioned the cert has expired! Checking the certificate information, indeed etcd components certificates has expired.

```bat
$ sudo kubeadm alpha certs check-expiration
[check-expiration] Reading configuration from the cluster...
[check-expiration] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[check-expiration] Error reading configuration from the Cluster. Falling back to default configuration

W0505 20:30:58.449935   13377 configset.go:202] WARNING: kubeadm cannot validate component configs for API groups [kubelet.config.k8s.io kubeproxy.config.k8s.io]
CERTIFICATE                EXPIRES                  RESIDUAL TIME   CERTIFICATE AUTHORITY   EXTERNALLY MANAGED
admin.conf                 Mar 15, 2021 18:02 UTC   <invalid>                               no
apiserver                  Mar 15, 2021 18:02 UTC   <invalid>       ca                      no
apiserver-etcd-client      Mar 15, 2021 18:02 UTC   <invalid>       etcd-ca                 no
apiserver-kubelet-client   Mar 15, 2021 18:02 UTC   <invalid>       ca                      no
controller-manager.conf    Mar 15, 2021 18:02 UTC   <invalid>                               no
etcd-healthcheck-client    Mar 15, 2021 18:02 UTC   <invalid>       etcd-ca                 no
etcd-peer                  Mar 15, 2021 18:02 UTC   <invalid>       etcd-ca                 no
etcd-server                Mar 15, 2021 18:02 UTC   <invalid>       etcd-ca                 no
front-proxy-client         Mar 15, 2021 18:02 UTC   <invalid>       front-proxy-ca          no
scheduler.conf             Mar 15, 2021 18:02 UTC   <invalid>                               no

CERTIFICATE AUTHORITY   EXPIRES                  RESIDUAL TIME   EXTERNALLY MANAGED
ca                      Mar 13, 2030 18:02 UTC   8y              no
etcd-ca                 Mar 13, 2030 18:02 UTC   8y              no
front-proxy-ca          Mar 13, 2030 18:02 UTC   8y              no
```


**Renew the Certs**

To renew the cert manually, I will then need to run the command “kubeadm alpha certs renew <CERTIFICATE>”

```bat
$ sudo kubeadm alpha certs renew apiserver-etcd-client
[renew] Reading configuration from the cluster...
[renew] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[renew] Error reading configuration from the Cluster. Falling back to default configuration

W0505 20:51:20.524246    2659 configset.go:202] WARNING: kubeadm cannot validate component configs for API groups [kubelet.config.k8s.io kubeproxy.config.k8s.io]
certificate the apiserver uses to access etcd renewed```


When all certs has been renewed and the kubernetes cluster will come back online

 ```bat
 $ sudo kubeadm alpha certs check-expiration
[check-expiration] Reading configuration from the cluster...
[check-expiration] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'

CERTIFICATE                EXPIRES                  RESIDUAL TIME   CERTIFICATE AUTHORITY   EXTERNALLY MANAGED
admin.conf                 May 05, 2022 18:37 UTC   364d                                    no
apiserver                  May 05, 2022 18:50 UTC   364d            ca                      no
apiserver-etcd-client      May 05, 2022 18:51 UTC   364d            etcd-ca                 no
apiserver-kubelet-client   May 05, 2022 19:24 UTC   364d            ca                      no
controller-manager.conf    May 05, 2022 19:25 UTC   364d                                    no
etcd-healthcheck-client    May 05, 2022 18:35 UTC   364d            etcd-ca                 no
etcd-peer                  May 05, 2022 19:25 UTC   364d            etcd-ca                 no
etcd-server                May 05, 2022 19:25 UTC   364d            etcd-ca                 no
front-proxy-client         May 05, 2022 19:34 UTC   364d            front-proxy-ca          no
scheduler.conf             May 05, 2022 19:34 UTC   364d                                    no

CERTIFICATE AUTHORITY   EXPIRES                  RESIDUAL TIME   EXTERNALLY MANAGED
ca                      Mar 13, 2030 18:02 UTC   8y              no
etcd-ca                 Mar 13, 2030 18:02 UTC   8y              no
front-proxy-ca          Mar 13, 2030 18:02 UTC   8y              no
```


**Regen the kube config**

```bat
K8S_IP=$(kubectl config view -o jsonpath={.clusters[0].cluster.server} | cut -d/ -f3 | cut -d: -f1)
echo $K8S_IP
kubeadm init phase certs all --apiserver-advertise-address $K8S_IP
cp /etc/kubernetes/admin.conf ~/.kube/config`
```


```bat
$ kubectl get nodes
NAME           STATUS   ROLES    AGE    VERSION
rasp-node-01   Ready    master   416d   v1.18.0
```
