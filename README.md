# k8s-vagrant-virtualbox
Create a local kubernetes cluster using virtualbox. A modification of the great work [danielepolencic](https://github.com/danielepolencic) did in his [gist](https://gist.github.com/danielepolencic/ef4ddb763fd9a18bf2f1eaaa2e337544).

The vagrant file will do the following:
1.  Provision all local VMs using VirtualBox
2.  Update & patch the OS
3.  Install containerd
4.  Install k8s control plane
5.  Initialize cluster with Flannel CIDR block & install Flannel
6.  Join the worker nodes to the master
7.  Create and copy the SSH key to all virtual machines so you can SSH to any node from the Master. Add names & IPs to the local hosts file on each master and worker node. Create alias in vagrant home for kubectl, now you can use just k
8.  Make required Ubuntu OS mods for the cluster to function properly

## Dependencies

You should install [VirtualBox](https://www.virtualbox.org/wiki/Downloads) and [Vagrant](https://www.vagrantup.com/downloads.html) before you start.

## Make sure git is installed

Instal [git](https://git-scm.com/downloads) if you don't already have it.

## Open a shell and clone

```bash
$ git clone https://github.com/smart-cn/k8s-vagrant-virtualbox.git
$ cd k8s-vagrant-virtualbox
```

## Starting the cluster

You can create the cluster with:

```bash
$ vagrant up
```

## Clean up

You can delete the cluster with:

```bash
$ vagrant destroy -f
```

## SSH and other Commands

SSH to Master and Worker Nodes:

```bash
$ vagrant ssh k8master-vm
$ vagrant ssh k8worker1-vm
$ vagrant ssh k8worker2-vm
```

Get the status of the Nodes:

```bash
vagrant@k8master-vm:~$ k get nodes -o wide
NAME           STATUS   ROLES                  AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION       CONTAINER-RUNTIME
k8master-vm    Ready    control-plane,master   37m   v1.21.0   10.0.0.10     <none>        Ubuntu 18.04.5 LTS   4.15.0-128-generic   containerd://1.3.3
k8worker1-vm   Ready    <none>                 25m   v1.21.0   10.0.0.11     <none>        Ubuntu 18.04.5 LTS   4.15.0-128-generic   containerd://1.3.3
k8worker2-vm   Ready    <none>                 18m   v1.21.0   10.0.0.12     <none>        Ubuntu 18.04.5 LTS   4.15.0-128-generic   containerd://1.3.3
```

SSH to other Nodes in the cluster from the Master:

```bash
$ ssh k8master-vm
$ ssh k8worker1-vm
```
