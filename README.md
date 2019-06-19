# kubernetes-kubeadm
Scripts to create kubernetes cluster, Deploy sample application, get familiar with use helm, moniroting using prometheus, grafana, heapster etc.

Steps to create the kubernetes cluster.

Create the servers on your cloud-provider or bare-metal data center
and make sure you have ssh access as a root user.

## Follow the below steps to create cluster using kubeadm.

**Step 1 Setting Up the Workspace Directory and Ansible Inventory File.**

```
$ mkdir ~/kube-cluster
$ cd ~/kube-cluster
```

Create the hosts file using `vim  ~/kube-cluster/hosts` and add following
configuration remember to replace master and node ips

```
master ansible_host=master_ip ansible_user=root

[workers]
worker1 ansible_host=worker_1_ip ansible_user=rooti
worker2 ansible_host=worker_2_ip ansible_user=root

[all:vars]
ansible_python_interpreter=/usr/bin/python3

```

**Step 2 Creating a Non-Root User on All Remote Servers.**

Run the playbook `initial.yml` which sets up the user and required permissions.

`ansible-playbook -i hosts ~/kube-cluster/initial.yml`

**Step 3 Installing Kubernetes' Dependencies**

Run the playbook `kube-dependencies.yml`

This playbook configures following things:

- Installs Docker, the container runtime.
- Installs apt-transport-https, allowing you to add external HTTPS sources to your APT sources list.
- Adds the Kubernetes APT repository's apt-key for key verification.
- Adds the Kubernetes APT repository to your remote servers' APT sources list.
- Installs kubelet and kubeadm.
- The second play consists of a single task that installs kubectl on your master node.

**Step 4 Setting Up the Master Node*.*

Execute the playbook by running:

```ansible-playbook -i hosts ~/kube-cluster/master.yml```

In the above playbook

```
- The first task initializes the cluster by running kubeadm init. Passing the argument --pod-network-cidr=10.244.0.0/16 specifies the private subnet that the pod IPs will be assigned from. Flannel uses the above subnet by default; we're telling kubeadm to use the same subnet.
- The second task creates a .kube directory at /home/ubuntu. This directory will hold configuration information such as the admin key files, which are required to connect to the cluster, and the cluster's API address.
- The third task copies the /etc/kubernetes/admin.conf file that was generated from kubeadm init to your non-root user's home directory. This will allow you to use kubectl to access the newly-created cluster.
- The last task runs kubectl apply to install Flannel. kubectl apply -f descriptor.[yml|json] is the syntax for telling kubectl to create the objects described in the descriptor.[yml|json] file. The kube-flannel.yml file contains the descriptions of objects required for setting up Flannel in the cluster.
```

To check the status of the master node, SSH into it with the following command:

`ssh ubuntu@master_ip`

Once inside the master node, execute:

`kubectl get nodes`

You will now see the following output:

```
Output
NAME      STATUS    ROLES     AGE       VERSION
master    Ready     master    1d        v1.14.0
```
The output states that the master node has completed all initialization tasks and is in a Ready state from which it can start accepting worker nodes and executing tasks sent to the API Server. You can now add the workers from your local machine.


**Step 5 Setting Up the Worker Nodes.**

Run the playbook worker.yml using following command.

```
 ansible-playbook -i hosts ~/kube-cluster/workers.yml
```
**Step 6 Verifying the Cluster.**

`ssh ubuntu@master_ip`

Then execute the following command to get the status of the cluster:

```
kubectl get nodes
You will see output similar to the following:

Output
NAME      STATUS    ROLES     AGE       VERSION
master    Ready     master    1d        v1.14.0
worker1   Ready     <none>    1d        v1.14.0
worker2   Ready     <none>    1d        v1.14.0
```
If all of your nodes have the value Ready for STATUS, it means that they're part of the cluster and ready to run workloads.

## Setting up CI/CD pipeline using Jenkins on Kubernetes.
