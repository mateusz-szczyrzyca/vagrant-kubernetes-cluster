# vagrant-kubernetes-cluster

Bootstrap easy your own 3-node k8s cluster within vagrant for tests and development

[This is modified version of this original tutorial from kubernetes.io](https://kubernetes.io/blog/2019/03/15/kubernetes-setup-using-ansible-and-vagrant/)

[[https://github.com/username/repository/blob/master/vagrant-kubernetes-cluster.png|alt=cluster_scheme]]

## Purpose

Purpose of this project is to create your own cluster for **development** and **testing** purposes.

**Do not try to create such cluster for production usage!**

You can use [minikube](https://github.com/kubernetes/minikube) or [microk8s](https://microk8s.io/) (and many more similar projects) but using them usually does not reflect the "real" cluster conditions which you may encounter on various cloud providers or [manually created cloud clusters](https://github.com/kelseyhightower/kubernetes-the-hard-way)

The cluster uses vagrant with 4 separate VMs (this can be changed in `Vagrantfile`), 1 for master, 3 for nodes. 

They will be communicating between themselves by an internal vagrant network which simulates normal private network 
in the same datacenter. In this cluster, you have to attach [PersistentVolumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) by yourself (we will use NFS share for this) 
and configure, let's say, your own docker registry.

Because of this, this cluster reflects much better **real** cluster, rather than, for instance - minikube.

## Limitations

This whole cluster is intended to run on the same machine, hence:

- it's not HA cluster
- etcd resides only on master, normally it should be separate HA cluster too, ideally on complete separate instances
- TLS certs are self signed

## Security

This cluster is intended to be run only on VMs on host - it's depended on host security mostly, however, be aware that:

- docker daemon on master vm and nodes vm is listening on tcp interface which is highly unsecure
- docker registry is insecure registry (on your host if you follow this instruction)
- sudo on root is without password from user vagrant
- nfs share can be easily exposed outside by simple mistake
- public vagrant vm image can be malicious and harm your host
- public docker images can be malicius and harm your vms or host
- tls certs are self signed

It's highly recommend to bootstrap this cluster on host which is not directly exposed to the Internet via an public IP 
address (for example: within NAT network behind router/firewall). In such infrastructure, unsecure registers, shares 
and vms are less likely to be compromised.

## Requirements

- 8 vCPU cores recommended (2 vCPU per 1 VM), more is better
- min 16 GB ram for cluster purposes (4 GB per 1 VM), more is better
- vagrant with vbox provider (tested on vbox)
- ansible (to bootstrap cluster when VM are ready)
- docker and docker-compose, for deploy private insecure docker registry for the cluster, but you can use helm to deploy similar registry directly within the cluster
- NFS server (as [PersistentVolume](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) for pods), you need to setup your own local NFS server

## Prerequsities

Before you start, [I highly recommend to follow this great tutorial from Kelsey Hightower](https://github.com/kelseyhightower/kubernetes-the-hard-way) to understand how kubernetes 
cluster works and how to bootstrap it correctly from scratch. This gives you better understanding how this 
vagrant-kubernetes-cluster works.

The cluster will be using the following private IP addresses:

- 15.0.0.1 for host
- 15.0.0.5 for master
- 15.0.0.11-13 for nodes

Knowing ip addresses are not mandatory to bootstrap and operating this cluster - mostly you can use `vagrant ssh` or 
`vagrant scp` commands to interact with VMs, but it can be helpful when you have to debug something later.

This configuration was successfully tested on Manjaro Linux, as VM OS is used Ubuntu 18.04.3 LTS

## 1. Installations

Install (according to your distribution packages)

- docker (don't forget to add you to `docker` group to be able to use docker from user)
- ansible (I've tested 2.9.2 version with python 3.8)
- docker-compose
- [vagrant](https://www.vagrantup.com/docs/installation/) (with virtualbox provider, tested version 2.2.6)
- nfs-server (sometimes it's nfs-common package, depends on distribution)
- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
- [helm](https://helm.sh/docs/intro/install/)

WARNING: If you use Manjaro/Archlinux, while writting this article (just before 2020), repos contains 2.2.6 vagrant and 
virtualbox 6.1, hence quite new, [you have to do some workaround to make vagrant work with virtualbox](https://github.com/oracle/vagrant-boxes/issues/178)

If you already have vagrant, install `vagrant-scp` plugin by typing:

`$ vagrant plugin install vagrant-scp`

Basically you should have available vagrant ssh and vagrant scp commands to interact with VMs

## 2. Setting NFS share

One directory on host system will be used as an NFS share. In my case this is `/home/nfs`

Add the following line to `/etc/exports` on your host system:

`/home/nfs 15.0.0.0/16(rw,sync,crossmnt,insecure,no_root_squash,no_subtree_check)`

And later type: `# exportfs -avr`

And you should got:

`exporting 15.0.0.0/16:/home/nfs`

NOTE: Bear in mind if your host already has 15.0.0.0/16 network interface for different purpose - in such case you have to change playbooks and Vagrantfile. Futhermore, if your host is within a NAT network and this network already uses 15.0.0.0/16 addresses, that's not a good idea to keep using this address for NFS server (without firewall rules, you expose your NFS share to this network)

## 3. Setting INSECURE docker registry

I've used standard docker-compose template in this repo (`docker-registry` directory) - just go to this directory and type:

`$ docker-compose up -d`

To check if it works correctly you can use curl:

`$ curl -X GET http://localhost:5000/v2/_catalog`

And if it works correctly you should got:

`{"repositories":[]}`

Alternatively, you can use [docker-registry helm chart to deploy](https://hub.helm.sh/charts/stable/docker-registry)

## 4. Clone this repository

In my case, my vagrant directory is `~/vagrant/vagrant-kubernetes-cluster` and I will be using this directory in this tutorial. In my case kubernetes-cluster is a directory which contains file from this repository. Yoi can clone this by:

`$ git clone https://github.com/mateusz-szczyrzyca/vagrant-kubernetes-cluster/`

## 5. Provision VMs via vagrant

Now go to the repository, Vagrantfile should be there and write:

`$ vagrant up`

If everything is ok, vagrant should start 4 vms and ansible should perform configuration of these vms to work within the cluster.

You can check if VMs are working properly by typing vagrant status:

```bash
$ vagrant status
Current machine states:

k8s-master                running (virtualbox)
node-1                    running (virtualbox)
node-2                    running (virtualbox)
node-3                    running (virtualbox)

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run vagrant status NAME.
```
###
###
###
## 6. Interact with your cluster

When all machines are working and there were no errors during ansible configuration, type:

`$ vagrant scp k8s-master:/home/vagrant/.kube/config .`

It will copy `/home/vagrant/.kube/config` file to your host machine with certificates which is needed to be used by 
`kubectl`.

Now you can point to this file:

`$ export KUBECONFIG=$PWD/config`

And now kubectl should work properly:

`$ kubectl get nodes -o wide`

Now you should be able to see nodes in the cluster:

```bash
NAME         STATUS   ROLES    AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
k8s-master   Ready    master   20m   v1.17.0   10.0.2.15     <none>        Ubuntu 18.04.3 LTS   4.15.0-72-generic   docker://19.3.5
node-1       Ready    <none>   17m   v1.17.0   15.0.0.11     <none>        Ubuntu 18.04.3 LTS   4.15.0-72-generic   docker://19.3.5
node-2       Ready    <none>   15m   v1.17.0   15.0.0.12     <none>        Ubuntu 18.04.3 LTS   4.15.0-72-generic   docker://19.3.5
node-3       Ready    <none>   13m   v1.17.0   15.0.0.13     <none>        Ubuntu 18.04.3 LTS   4.15.0-72-generic   docker://19.3.5
```

It means the cluster is working properly!

## 7. Attach NFS share as PersistentVolume

Now it's time to use our NFS share as PV in our cluster.

To do this, we will install [nfs-client-provisioner](https://hub.helm.sh/charts/rimusz/nfs-client-provisioner) from helm chart. To install this, just type:

`$ helm install stable/nfs-client-provisioner --generate-name --set nfs.server="15.0.0.1" --set nfs.path="/home/nfs"`

If you changed your NFS server/path to different than in this tutorial, please modify this command accordingly.

Now we can check if we have a new pod:

```bash
$ kubectl get pods -o wide         
NAME                                                READY   STATUS    RESTARTS   AGE   IP               NODE     NOMINATED NODE   READINESS GATES
nfs-client-provisioner-1577712423-b9bcf4778-sw2xm   1/1     Running   0          10s   192.168.139.65   node-3   <none>           <none>
```

This pod adds a new storageclass `nfs-client`, check this using the command `$ kubectl get sc`:

```bash
NAME         PROVISIONER                                       RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
nfs-client   cluster.local/nfs-client-provisioner-1577712423   Delete          Immediate           true                   12m
```

We need to make this storageclass as default for our cluster, by commmand: 

`$ kubectl patch storageclass nfs-client -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'`

And now we can check again by `$ kubectl get sc` that this class is now default:

```bash
NAME                   PROVISIONER                                       RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
nfs-client (default)   cluster.local/nfs-client-provisioner-1577712423   Delete          Immediate           true                   12m
```

## 8. Using docker with this cluster

To attach to docker daemon exposed by master (or nodes), set this env variable:

`$ export DOCKER_HOST="tcp://15.0.0.5:2375"`

And now you can use cluster's docker:

`$ docker ps`

If you want to use your newly created docker image, use `docker build` as before, but when building is finished, check image id of newly created image:

`docker images`

Pick this image, and now use tag:

`$ docker tag 909252161370 15.0.0.1:5000/my-kubernetes-app`

And push to this registry, by:

`$ docker push 15.0.0.1:5000/my-kubernetes-app`

If you docker registry is working fine now you have new image pushed there, and this new image can be used in kubernetes yaml files in such manner:

```yaml
        image: 15.0.0.1:5000/my-kubernetes-app
```
