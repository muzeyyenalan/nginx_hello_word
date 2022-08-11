# nginx_hello_word

ubuntuserver kubernetes cluster with "hello word"

# **Installing Kubernetes on Ubuntu 20.04 running on AWS EC2 Instances and deploying an Nginx server on Kubernetes**

# **Part 1 Configuring Security Groups**

***Ports for the Control-plane (Master) Node(s)***

```
1. TCP 6443      → For Kubernetes API server
2. TCP 2379–2380 → For etcd server client API
3. TCP 10250     → For Kubelet API
4. TCP 10259     → For kube-scheduler
5. TCP 10257     → For kube-controller-manager
6. TCP 22        → For remote access with ssh
7. UDP 8472      → Cluster-Wide Network Comm. — Flannel VXLAN
```

***Ports for the Worker Node(s)***

```
1. TCP 10250       → For Kubelet API
2. TCP 30000–32767 → NodePort Services†
3. TCP 22          → For remote access with ssh
4. UDP 8472        → Cluster-Wide Network Comm. — Flannel VXLAN
```

# **Part 2 Creating instances**

Choose `t2.medium`( worker,master) as instance type, choose your own key, select existing security groups and choose the master security group that we just created and finally hit on `launch instance(worker and master)`

Then ssh into both of them

ssh -i "xxxx.pem" ubuntu@ec2-54-82-5-115.compute-1.amazonaws.com `<master instance>`

ssh -i "xxxx.pem" ubuntu@ec2-54-82-5-115.compute-1.amazonaws.com `<worker instance>`

# **Part 3 Working with instances**

1-for hostname

```
sudo hostnamectl set-hostname master
```

2-hostname actived

```
bash
```

3-ubuntu server for update

```
sudo apt-get update
```

4-we can install helper packages for Kubernetes.

```
sudo apt-get install -y apt-transport-https gnupg2
```

5-installing the helper packages

```
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
```

```
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee -a /etc/apt/sources.list.d/kubernetes.list
```

6-We need to update the system again after we are done with the helper packages

```
sudo apt-get update
```

7-Continue with the Kubernetes installation including Docker

```
sudo apt-get install -y kubectl kubeadm kubelet kubernetes-cni docker.io
```

8-Now we have to start and enable Docker service.

```

sudo systemctl start docker
sudo systemctl enable docker
```

9-For the Docker group to work smoothly without using sudo command, we should add the current user to the `Docker group`

```
sudo usermod -aG docker $USER
```

10-Now we run the following command so that the changes take affect immediately.

```
newgrp docker
```

11-As a requirement, update the `iptables` of Linux Nodes to enable them to see bridged traffic correctly. Thus, you should ensure `net.bridge.bridge-nf-call-iptables` is set to `1` in your `sysctl` config and activate `iptables` immediately.

```
cat << EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
```

We are done with the initial configuration of the master node. Now we will repeat all 10 steps for the worker node. The only difference would be the name of the node.

1-Change the terminal to the worker node instance and change the host name

```
sudo hostnamectl set-hostname worker
```

2-write "bash"

```
bash
```

3-Before installing Kubernetes packages, we should update the system.

```
sudo apt-get update
```

4-we can install helper packages for Kubernetes.

```
sudo apt-get install -y apt-transport-https gnupg2
```

5-Continue installing the helper packages

```
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
```

```
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee -a /etc/apt/sources.list.d/kubernetes.list
```

6-We need to update the system again after we are done with the helper packages.

```
sudo apt-get update
```

7-Continue with the Kubernetes installation including Docker

```
sudo apt-get install -y kubectl kubeadm kubelet kubernetes-cni docker.io
```

8-Now we have to start and enable Docker service.

```
sudo systemctl start docker
sudo systemctl enable docker
```

9-For the Docker group to work smoothly without using sudo command, we should add the current user to the `Docker group`.

```
sudo usermod -aG docker $USER
```

10-Now we run the following command so that the changes take affect immediately.

```
newgrp docker
```

11-As a requirement, update the `iptables` of Linux Nodes to enable them to see bridged traffic correctly. Thus, you should ensure `net.bridge.bridge-nf-call-iptables` is set to `1` in your `sysctl` config and activate `iptables` immediately.

```
cat << EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
```

# **Part 4 — Setting Up Master Node for Kubernetes**

1-We will pull the packages for Kubernetes beforehand

```
sudo kubeadm config images pull
```

2-By default, the Kubernetes cgroup driver is set to system, but docker is set to systemd. We need to change the Docker cgroup driver by creating a configuration file `/etc/docker/daemon.json` and adding the following line then restart deamon, docker and kubelet:

```
vim /etc/docker/daemon.json
```

```
echo '{"exec-opts": ["native.cgroupdriver=systemd"]}' | sudo tee /etc/docker/daemon.json
```

```
sudo systemctl daemon-reload
```

```
sudo systemctl restart docker
sudo systemctl restart kubelet
```

3-At this stage, the `kubeadm` will prepare the environment for us. For this we need the private IP of the master node.

```
sudo kubeadm init --apiserver-advertise-address=<172.31.17.170 >--pod-network-cidr=10.244.0.0/16    #Use your master node’s private IP
```

4-At this point note down the last part of the output that includes the `kubeadm join `:

like

```
#kubeadm join 172.31.17.170:6443 --token zycl0r.nr9mi4cwwrcc9d19
    --discovery-token-ca-cert-hash sha256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

5-Now, we need to run the following commands to set up local `kubeconfig` on master node.

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

6-Activate the `Flannel` pod networking.

```
kubectl apply -f https://github.com/coreos/flannel/raw/master/Documentation/kube-flannel.yml
```

7-Finally check to see that the Master node is ready

```
kubectl get nodes ( nodes must be ready)
```

# **Part 5 — Adding the Worker Node to the Cluster**

1-By default, the Kubernetes cgroup driver is set to system, but docker is set to systemd. We need to change the Docker cgroup driver by creating a configuration file `/etc/docker/daemon.json` and adding the following line then restart deamon, docker and kubelet:

```
vim /etc/docker/daemon.json
```

```
echo '{"exec-opts": ["native.cgroupdriver=systemd"]}' | sudo tee /etc/docker/daemon.json
```

```
sudo systemctl daemon-reload
```

```
sudo systemctl restart docker
```

```
sudo systemctl restart kubelet
```

2-Remember that we noted `sudo kubeadm join…` command previously. We will now run that command to have them join the cluster. Do not forget to add `sudo` before the command.

```
sudo kubeadm join <172.31.21.161:6443 — token 7iwh5m.v8pqnnhjl18l81xh — discovery-token-ca-cert-hash sha256:f7xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx> #you will paste kubeadm join commad for master node 4
```

Now let’s go to the master node and get the list of nodes. If we did everything correctly, then we should see the new worker node in the list.

```
kubectl get nodes
```

for detailed version of the nodes

```
kubectl get nodes -o wide
```

# **Part 6 — Deploying a Simple Nginx Server with "Hello Word"on Kubernetes**

1-You will create "Hello Word " deployment file

```
vim  hello-word-deployment.yaml
```

---

```
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30001
  selector:
    app: nginx
```

---

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  nginx.conf: '
events {
}
http {
   server {
       listen 80;
       location / {
           return 200 "Hello world!";
       }
   }
}
'
```

---

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx:latest
        name: nginx
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
          - name: config-vol
            mountPath: /etc/nginx/
      volumes:
        - name: config-vol
          configMap:
            name: nginx-config
            items:
              - key: nginx.conf
                path: nginx.conf
```

2-for the apply deployment

```
kubectl apply -f  hello-word-deployment.yaml
```

3-go to browser `<workernode publicIP :30001>`
