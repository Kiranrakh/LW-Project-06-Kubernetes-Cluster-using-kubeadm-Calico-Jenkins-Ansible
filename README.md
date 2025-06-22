# 🚀 Jenkins + Ansible on Kubernetes (EC2 Project)

## 📝 Project Overview

This project sets up a complete CI/CD automation pipeline using **Jenkins** and **Ansible**, deployed on a **Kubernetes cluster running on AWS EC2 instances**.

---

## 🎯 Key Objectives

- ✅ Deploy Jenkins on a Kubernetes cluster
- ✅ Build and push Docker images to Docker Hub
- ✅ Host Jenkins and Ansible in custom containers
- ✅ Use Ansible for automation tasks inside the Kubernetes cluster
- ✅ Use Jenkins UI to trigger builds and run jobs
- ✅ Understand how Kubernetes deployments, pods, services, nodeSelectors, and tolerations work together

---

## 🧱 Project Architecture

```text
         ┌────────────────────────┐
         │                        │
         │   AWS EC2 Instances    │
         │                        │
         └──────────┬─────────────┘
                    │
         ┌──────────▼────────────┐
         │  Kubernetes Master    │
         │ (Control Plane Node)  │
         └──────────┬────────────┘
                    │
      ┌─────────────┴──────────────┐
      │                            │
┌─────▼─────┐              ┌───────▼───────┐
│ Jenkins   │              │  Ansible Pod  │
│ Pod       │              │ (Container)   │
└─────▲─────┘              └───────────────┘
      │
      └────── Exposes NodePort: 30080 → Jenkins UI
````

---

## 🧪 Environment Setup Commands

### ✅ Master Node Setup (Run on master EC2)

```bash
sudo yum update -y
sudo yum install -y yum-utils device-mapper-persistent-data lvm2 git curl
sudo amazon-linux-extras enable docker
sudo yum install -y docker
sudo systemctl enable docker
sudo systemctl start docker
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/repodata/repomd.xml.key
EOF
sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
sudo systemctl enable kubelet
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
curl https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/calico.yaml -O
kubectl apply -f calico.yaml
kubectl get nodes
```

---

### ✅ Worker Node Setup (Run on each worker EC2)

```bash
sudo yum update -y
sudo yum install -y yum-utils device-mapper-persistent-data lvm2 git curl
sudo amazon-linux-extras enable docker
sudo yum install -y docker
sudo systemctl enable docker
sudo systemctl start docker
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/repodata/repomd.xml.key
EOF
sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
sudo systemctl enable kubelet
# Replace with your actual token and master IP
sudo kubeadm join <MASTER-IP>:6443 --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>
```

---

## 🐳 Docker Image Build & Push

### 🔨 Jenkins Dockerfile

```Dockerfile
FROM amazonlinux:2

ENV JENKINS_HOME=/var/jenkins_home
ENV JENKINS_VERSION=2.426.1
ENV JENKINS_URL=https://get.jenkins.io/war-stable/${JENKINS_VERSION}/jenkins.war

RUN yum update -y && \
    amazon-linux-extras enable java-openjdk11 && \
    yum install -y java-11-openjdk wget git curl unzip && \
    yum clean all

RUN useradd -m -d $JENKINS_HOME -s /bin/bash jenkins && \
    mkdir -p /opt/jenkins && \
    chown -R jenkins:jenkins $JENKINS_HOME /opt/jenkins

RUN wget -q -O /opt/jenkins/jenkins.war $JENKINS_URL

USER jenkins

EXPOSE 8080
WORKDIR $JENKINS_HOME
CMD ["java", "-jar", "/opt/jenkins/jenkins.war"]
```

Build & Push Jenkins Image:

```bash
docker build -t yourdockerhubuser/manual-jenkins:v1 -f Jenkins-Dockerfile .
docker login
docker push yourdockerhubuser/manual-jenkins:v1
```

---

### 🔧 Ansible Dockerfile

```Dockerfile
FROM amazonlinux:2

RUN yum update -y && \
    yum install -y python3-pip openssh-clients git && \
    pip3 install ansible

CMD ["/bin/bash"]
```

Build & Push Ansible Image:

```bash
docker build -t yourdockerhubuser/manual-ansible:v1 -f Ansible-Dockerfile .
docker login
docker push yourdockerhubuser/manual-ansible:v1
```

---

## 📦 Kubernetes Deployment

### 🔹 Create Namespace

```bash
kubectl create namespace devops-tools
```

### 🔹 Jenkins Deployment YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins
  namespace: devops-tools
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jenkins
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      nodeSelector:
        jenkins-node: "true"
      tolerations:
      - key: "node-role.kubernetes.io/control-plane"
        operator: "Exists"
        effect: "NoSchedule"
      containers:
      - name: jenkins
        image: yourdockerhubuser/manual-jenkins:v1
        ports:
        - containerPort: 8080
```

### 🔹 Jenkins Service YAML

```yaml
apiVersion: v1
kind: Service
metadata:
  name: jenkins-service
  namespace: devops-tools
spec:
  type: NodePort
  selector:
    app: jenkins
  ports:
  - port: 8080
    targetPort: 8080
    nodePort: 30080
```

Apply Jenkins resources:

```bash
kubectl apply -f jenkins-deployment.yaml
kubectl apply -f jenkins-service.yaml
```

---

### 🔹 Ansible Pod YAML

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ansible
  namespace: devops-tools
spec:
  containers:
  - name: ansible
    image: yourdockerhubuser/manual-ansible:v1
    imagePullPolicy: Always
    command: ["/bin/bash", "-c", "while true; do sleep 3600; done"]
```

Deploy Ansible pod:

```bash
kubectl apply -f ansible-pod.yaml
```

---

## 🌐 Access Jenkins UI

```bash
kubectl get svc -n devops-tools
```

Open in browser:

```
http://<your-ec2-public-ip>:30080
```

Get initial Jenkins admin password:

```bash
kubectl get pods -n devops-tools
kubectl exec -n devops-tools -it <jenkins-pod-name> -- cat /var/jenkins_home/secrets/initialAdminPassword
```

---

## 🧩 Node Labeling (if required)

```bash
kubectl label node <node-name> jenkins-node=true
```

Patch deployment with selector (if needed):

```bash
kubectl patch deployment jenkins -n devops-tools \
  --type='json' \
  -p='[{"op": "add", "path": "/spec/template/spec/nodeSelector", "value": {"jenkins-node": "true"}}]'
```

---

## 🔐 PEM & Docker Push from Local

If using Docker from your local machine to build and push to Docker Hub:

```bash
scp -i your-key.pem Dockerfile ec2-user@<master-ip>:/home/ec2-user/
```

Then SSH into EC2:

```bash
ssh -i your-key.pem ec2-user@<master-ip>
cd /home/ec2-user/
docker build ...
```

---

## ✅ Verification

```bash
kubectl get nodes
kubectl get pods -n devops-tools
kubectl logs -n devops-tools jenkins-xxx
```

---

## 🏁 Conclusion
