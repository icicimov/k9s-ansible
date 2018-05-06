# k9s-ansible
Bare metal Kubernetes cluster with Ansible

# Purpose 

This repository is to support the [Kubernetes cluster step-by-step](https://icicimov.github.io/blog/kubernetes/Kubernetes-cluster-step-by-step/) blog but in automated way via Ansible.

This is little bit different from the blog page in that only `kubelet`, `flanneld` and `etcd` run as systemd services on the host, all other K8S components are created as static Pods via manifests. It still provisions 3 hosts as Masters and Workers in the same time.

The cluster `RBAC` and `Node` authorization are enabled for increased security just to mention some of the features. All K8S components authenticate to each other via certificate and the API communication internal and external is over SSL too. Docker is using the `overlay2` storage driver which should provide best file system performance.

This is a short list of Kubernetes and other components in the cluster and their current versions:

* os: debian-jessie (kernel 4.9.0 from backports)
* etcd: 2.3.8
* k8s: 1.8.8
* dashboard: 1.7.1
* heapster: 1.4.0
* kubedns: 1.14.7
* flanneld: 0.7.1
* docker_engine: 1.12.6-0~debian-jessie
* docker_storage_driver: overlay2
* haproxy: 1.7
* traefik: latest

It's been tested and working with Ansible 2.4 and Kubernetes 1.7 and 1.8.

All important settings are in the `defaults/main.yml` file. Some specific settings that we might want to enable or disable: 

```
k8s_cluster_dns_setup: true
k8s_cluster_dns_replicas: 1
k8s_cluster_dns_autoscale: false
k8s_kubectl_setup: false
k8s_heapster_setup: false
k8s_dashboard_setup: false
```

They control features like installing the Dashboard, Heapster for monitoring, kube-dns for DNS resolution inside the cluster (with optional auto-scaling and core-dns as alternative) and installing and setup of local `kubectl` and `kubeconfig` file for cluster CLI. The following variables are in control of the `Flanneld` overlay network, the Pods and Services network:

```
k8s_flanneld_subnet: '10.1.0.0/16'
k8s_pod_network: '10.2.0.0/16'
k8s_service_ip_range: '10.3.0.0/24'
k8s_service_ip: '10.3.0.1'
```

# Installation

Basic `hosts` inventory file and main Ansible playbook `k9s-provision.yml` are provided. The playbook consists of three parts:

* SSL certificates part, executed locally, that produces all the certificates needed for the Kubernetes services to identify them selfs to the clients and each other
* Cluster provisioning part that creates and configures all Kubernetes services on the hosts
* Kubernetes addons part, executed locally, that sets up the `kubectl` CLI tool and configures the local host user running Ansible for cluster access and (optionally) installs some addons like the essential Kube-DNS service for cluster internal DNS resolution, the Dashboard plugin and the Heapster addon (used with Helm)

The hosts are assumed to be running in local LAN with 192.168.0.0/24 CIDR.

After reviewing the repository and some parameters tuning to meet your need just run:

```
ansible-playbook -i hosts k9s-provision.yml
```

to provision the cluster. 

# Managing the cluster

When finished, assuming `k8s_kubectl_setup` has been set to `true`, you should see a `kubeconfig` file `~/.kube/config` with similar content to this:

```
apiVersion: v1
clusters:
- cluster:
    certificate-authority: /home/igorc/k9s-ansible/files/ssl/k9s.virtual.local/ca/ca.pem
    server: https://k9s-api.virtual.local
  name: k9s.virtual.local
contexts:
- context:
    cluster: k9s.virtual.local
    user: admin
  name: admin-context
current-context: admin-context
kind: Config
preferences: {}
users:
- name: admin
  user:
    client-certificate: /home/igorc/k9s-ansible/files/ssl/k9s.virtual.local/admins/admin.pem
    client-key: /home/igorc/k9s-ansible/files/ssl/k9s.virtual.local/admins/admin-key.pem
```

installed locally. Add the following in the local `/etc/hosts` to configure the DNS for the hosts and the API access point via Haproxy load balancer:

```
# K8S k9s.virtual.local cluster
192.168.0.151   k9s01.virtual.local k9s01
192.168.0.152   k9s02.virtual.local k9s02
192.168.0.153   k9s03.virtual.local k9s03
192.168.0.154   k9s-api.virtual.local k9s-traefik.virtual.local
```

To check the cluster health run:

```
kubectl get nodes -o wide
kubectl get pods -n kube-system
```

The `admin` user and all cluster SSL certificates will be installed under `k9s-ansible/files/ssl/` directory.

# Additional install

To install Traefik run:

```
kubectl apply -f files/manifests/traefik
```