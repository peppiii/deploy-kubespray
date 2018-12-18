## Introduction
This is Kubernetes Deployment With kubespray

## Table of contents
<!--ts-->
   * [Schema](#schema)
   * [Install](#install)
   * [Clone kubespray](#clone-kubespray)
   * [Add hosts](#add-hosts)
   * [Network kubernetes](#network-kubernetes)
   * [Deploy cluster](#deploy-cluster)
   * [Add cluster](#deploy-cluster)
   * [Proxy pass nginx](#proxy-pass-nginx)
   * [Dashboard kubernetes](#dashboard-kubernetes)
   * [Dashboard disable skip rbac](#dashboard-disable-skip-rbac)
   * [Dashboard token rbac](#dashboard-token-rbac)
   * [Dashboard token kubeconfig](#dashboard-token-kubeconfig)
   * [Token](#dashboard-kubernetes)
   * [Contributing](#contributing)
	 * [Security Vulnerabilities](#security-vulnerabilities)
	 * [License](#license)
<!--te-->

## Schema
<p align="center">
<a href="https://github.com/peppiii/deploy-kubespray/blob/master/topologi/kubespray.png"><img src="https://github.com/peppiii/deploy-kubespray/blob/master/topologi/kubespray.png" width= 100% alt="Build Status"></a>

## Install
```bash
$ sudo apt-get update
$ sudo apt-get install software-properties-common
$ sudo apt-add-repository ppa:ansible/ansible
$ sudo apt-get update
$ sudo apt-get install ansible
$ easy_install pip
$ pip2 install jinja2 --upgrade
$ sysctl net.ipv4.ip_forward
$ sysctl -w net.ipv4.ip_forward=1
```

## Clone Kubespray
if you can kubespray you must clone to server this code
```bash
$ cd /root 
$ git clone https://github.com/kubernetes-incubator/kubespray.git
$ cd kubespray
$ pip install -r requirements.txt
```

## Add Hosts
Hosts : this is ansible for automation, you must add host ip this is't

```bash
$ cd inventory
$ cp -R sample/ cluster
$ nano hosts.ini

# ## Configure 'ip' variable to bind kubernetes services on a
# ## different ip than the default iface
# ## We should set etcd_member_name for etcd cluster. The node that is not a etcd member do not need to set the value, or can set the empty string value.
[all]
# node1 ansible_host=95.54.0.12  # ip=10.3.0.1 etcd_member_name=etcd1
# node2 ansible_host=95.54.0.13  # ip=10.3.0.2 etcd_member_name=etcd2
# node3 ansible_host=95.54.0.14  # ip=10.3.0.3 etcd_member_name=etcd3
# node4 ansible_host=95.54.0.15  # ip=10.3.0.4 etcd_member_name=etcd4
# node5 ansible_host=95.54.0.16  # ip=10.3.0.5 etcd_member_name=etcd5
# node6 ansible_host=95.54.0.17  # ip=10.3.0.6 etcd_member_name=etcd6

master1 ansible_ssh_host=172.29.235.43 ip=172.29.235.43
master2 ansible_ssh_host=172.29.235.44 ip=172.29.235.44
node1 ansible_ssh_host=172.29.235.45 ip=172.29.235.45
node2 ansible_ssh_host=172.29.235.46 ip=172.29.235.46
node3 ansible_ssh_host=172.29.235.47 ip=172.29.235.47

# ## configure a bastion host if your nodes are not directly reachable
# bastion ansible_host=x.x.x.x ansible_user=some_user

[kube-master]
node1
node2

[etcd]
node1
node2
node3

[kube-node]
node1
node2
node3

[k8s-cluster:children]
kube-master
kube-nodek8s-cluster/
```

## Network Kubernetes
```bash
$ cd group_vars
$ cd k8s-cluster
$ cat k8s-cluster.yml | grep network

you can change network
# Choose network plugin (cilium, calico, contiv, weave or flannel)
kube_network_plugin: flannel

# In “inventory/mycluster/group_vars/all.yml”  
uncomment the following line to enable metrics to fetch the cluster resource utilization data without this HPAs will not work (for ‘kubectl top nodes’ & ‘kubectl top pods’ commands to work)

# The read-only port for the Kubelet to serve on with no authentication/authorization. Uncomment to enable.
kube_read_only_port: 10255

```

## Deploy Cluster
```bash
$ cd ../../../
$ ansible-playbook -i inventory/cluster/hosts.ini cluster.yml
```

## Proxy Pass Nginx
```bash
you can setup gateway proxy pass, and must HTTPS not HTTP.
oh yeah i use letsencrypt, you can prefer for installation https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-16-04

example
server {
        listen 80;
        server_name kube.e-cbncloud.co.id;
        return 302 https://kube.e-cbncloud.co.id$request_uri;
}

server{
	listen 443 ssl;
	server_name kube.e-cbncloud.co.id;
	
	location / {
		proxy_pass https://172.29.235.43:31974;
		proxy_redirect       off;
        	proxy_set_header     Host $http_host;
        	proxy_set_header    X-Forwarded-For $proxy_add_x_forwarded_for;
	}
	
	ssl_certificate /etc/letsencrypt/live/kube.e-cbncloud.co.id/fullchain.pem; # managed by Certbot
        ssl_certificate_key /etc/letsencrypt/live/kube.e-cbncloud.co.id/privkey.pem; # managed by Certbot
        include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
        ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot
}
```


## Dashboard kubernetes
in kubemaster you can this code for terminal :
```bash
$ kubectl -n kube-system edit service kubernetes-dashboard
apiVersion: v1
kind: Service
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"labels":{"k8s-app":"kubernetes-dashboard","kubernetes.io/cluster-service":"true"},"name":"kubernetes-dashboard","namespace":"kube-system"},"spec":{"ports":[{"port":443,"targetPort":8443}],"selector":{"k8s-app":"kubernetes-dashboard"}}}
  creationTimestamp: 2018-11-22T05:07:36Z
  labels:
    k8s-app: kubernetes-dashboard
    kubernetes.io/cluster-service: "true"
  name: kubernetes-dashboard
  namespace: kube-system
  resourceVersion: "4614"
  selfLink: /api/v1/namespaces/kube-system/services/kubernetes-dashboard
  uid: 86a98b59-ee14-11e8-bfbe-fa163e22d184
spec:
  clusterIP: 10.233.5.10
  externalTrafficPolicy: Cluster
  ports:
  - nodePort: 31974
    port: 443
    protocol: TCP
    targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard
  sessionAffinity: None
  type: Clusterip
status:
  loadBalancer: {}
```
if you see <b> type: Clusterip </b> you must change to NodePort because that you can expose this ip

now you can see change for NodePort

```bash
NAME                   TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)         AGE
kubernetes-dashboard   NodePort   10.233.5.10   <none>        443:31974/TCP   4h17m
```
<b> Noted : you can not kubectl proxy again, because you have nodeport for dashboard kubernetes </b>

## Dashboard disable skip rbac
if you want to remove rbac, please  checking dashboard.yml in kubernetes --> etc/kubernetes/dashboard.yml
```bash
$ kubectl delete -f dashboard.yml

$ nano dashboard.yml

 - name: kubernetes-dashboard
        image: gcr.io/google_containers/kubernetes-dashboard-amd64:v1.10.0
        imagePullPolicy: IfNotPresent
        resources:
          limits:
            cpu: 100m
            memory: 256M
          requests:
            cpu: 50m
            memory: 64M
        ports:
        - containerPort: 8443
          protocol: TCP
        args:
          - --auto-generate-certificates
          - --authentication-mode=token          # Uncomment the following line to manually specify Kubernetes API server Host
          # If not specified, Dashboard will attempt to auto discover the API server and connect
          # to it. Uncomment only if the default does not work.
          # - --apiserver-host=http://my-address:port
          - --token-ttl=900
```
<b> Noted : you can add - --disable-skip=true for args </b>

```bash
$ kubectl create -f dashboard.yml
```

## Dashboard token rbac

## Dashboard token kubeconfing
