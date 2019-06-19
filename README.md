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

**Step 4 Setting Up the Master Node*.**

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

Created a debian 9 server with with sudo user.

Follow below steps to install jenkins.

ssh to server and execute following commands..

```
 sudo apt install default-jre -y
 wget -q -O - https://pkg.jenkins.io/debian/jenkins.io.key | sudo apt-key add -
 sudo sh -c 'echo deb http://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list'
 sudo apt update
 sudo apt install jenkins -y
 sudo service jenkins start
```
Add the jenkins to docker group

```
Install Helm on the server.

```
wget https://get.helm.sh/helm-v2.14.1-linux-amd64.tar.gz
tar -zxvf helm-v2.0.0-linux-amd64.tgz 
mv linux-amd64/helm /usr/local/bin/helm
```

Install ansible on the server

```
sudo apt ansible -y
```

sudo groupadd docker
sudo usermod -aG docker jenkins
```
Open the URL **https://<Jenkins-server-ip>:8080**

Create user on jenkins dashboard and install following plugins.
 - Cloudbees docker
 - Kubernetes
 - git
 - ghprb

```
sudo -i -u jenkins
mkdir .kube ; $ touch .kube/config
```
copy the contents of /etc/kubernetes/admin.conf from master node to ~/.kube/config

helm init --upgrade

Create a jenkins pipeline and post the following script

```
node{
// define variables
  def Namespace = "default"
  def ImageName = "sayarapp/sayarapp"
  def Creds = "2dfd9d0d-a300-49ee-aaaf-0a3efcaa5279"
  try{
// pull and clone from the sample git repository
  stage('Checkout'){
      git 'https://mAyman2612@bitbucket.org/mAyman2612/ci-cd-k8s.git'
      sh "git rev-parse --short HEAD > .git/commit-id"
      imageTag= readFile('.git/commit-id').trim()
}
// Run unit tests
stage('RUN Unit Tests'){
      sh "npm install"
      sh "npm test"
  }
// Docker build and push to docker registry
  stage('Docker Build, Push'){
    withDockerRegistry([credentialsId: "${Creds}", url: 'https://index.docker.io/v1/']) {
      sh "docker build -t ${ImageName}:${imageTag} ."
      sh "docker push ${ImageName}"
        }
}
// Call the Ansible playbook to deploy on k8s
    stage('Deploy on K8s'){
sh "ansible-playbook /var/lib/jenkins/ansible/sayarapp-deploy/deploy.yml  --user=jenkins --extra-vars ImageName=${ImageName} --extra-vars imageTag=${imageTag} --extra-vars Namespace=${Namespace}"
    }
     } catch (err) {
      currentBuild.result = 'FAILURE'
    }
}
```
 
Access the application running in Kubernetes using:

```
kubectl get svc
```
To get the IP/Port of the application

Now curl http://<public-node-ip>:<node-port>.

In this way we installed jenkins on Debian server and configured the CI/CD pipeline
to deploy apps on kubernetes.

## 3. Create a development namespace.

Use kubectl command

```
kubectl create ns development
namespace development created
```

## 4. Deploy guest-book application in the development namespace.

Run the following kubectl commands to deploy the application.
```
  kubectl apply -f https://k8s.io/examples/application/guestbook/redis-master-deployment.yaml
  kubectl apply -f https://k8s.io/examples/application/guestbook/redis-master-service.yaml
  kubectl apply -f https://k8s.io/examples/application/guestbook/redis-slave-deployment.yaml
  kubectl apply -f https://k8s.io/examples/application/guestbook/redis-slave-service.yaml
  kubectl apply -f https://k8s.io/examples/application/guestbook/frontend-deployment.yaml
  kubectl apply -f https://k8s.io/examples/application/guestbook/frontend-service.yaml
  kubectl get services
  kubectl get service frontend
 
```


Access the guestbook application on the URL http://35.229.140.13:31218/

