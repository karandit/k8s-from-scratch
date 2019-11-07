# 01 Install Docker

    03  vi /etc/apt/apt.conf.d/proxy.conf

---
/etc/apt/apt.conf.d/proxy.conf

    Acquire::http::Proxy "http://192.168.28.1:5555";
    Acquire::https::Proxy "http://192.168.28.1:5555";
---

    04  apt update
    05  apt upgrade
    06  apt install docker.io -y
    43  mkdir /etc/systemd/system/docker.service.d
    44  vim /etc/systemd/system/docker.service.d/http-proxy.conf

---
/etc/systemd/system/docker.service.d/http-proxy.conf

    [Service]
    Environment="HTTP_PROXY=http://192.168.28.1:5555"
    Environment="HTTPS_PROXY=http://192.168.28.1:5555"
---
    42  docker pull hello-world
    48  systemctl daemon-reload
    49  systemctl show --property Environment docker
    50  sudo systemctl restart docker

# 02 Download k8s 1.9.11

    13  export http_proxy=http://192.168.28.1:5555
    14  export https_proxy=http://192.168.28.1:5555
    11  wget https://storage.googleapis.com/kubernetes-release/release/v1.9.11/kubernetes-server-linux-amd64.tar.gz
    18  tar -xzf kubernetes-server-linux-amd64.tar.gz
    19  cd kubernetes/server/bin/
    20  ls
    21  cp kubelet kubectl kube-apiserver kube-controller-manager kube-scheduler kube-proxy /usr/bin/
    22  cd
    23  kubectl
    24  mkdir -p /etc/kubernetes/manifests
    25  swapoff -a

# 03 Install kubelet

    32  kubelet --pod-manifest-path /etc/kubernetes/manifests/ &> /etc/kubernetes/kubelet.log &
    33  ps -au | grep kubelet
    34  head /etc/kubernetes/kubelet.log

---
kubelet-test.yaml

    apiVersion: v1
    kind: Pod
    metadata:
      name: kubelet-test
    spec:
      containers:
      - name: alpine
        image: alpine
        command: ["/bin/sh", "-c"]
        args: ["while true; do echo Baaaaang; sleep 15; done"]
---

    35  vim /etc/kubernetes/manifests/kubelet-test.yaml
    36  kubectl get po
    37  # no server
    52  docker ps
    53  docker logs 918e553c398c

# 04 Install etcd 3.2.26

    57  wget https://github.com/etcd-io/etcd/releases/download/v3.2.26/etcd-v3.2.26-linux-amd64.tar.gz
    60  tar -xzf etcd-v3.2.26-linux-amd64.tar.gz
    62  cp etcd-v3.2.26-linux-amd64/etcd /usr/bin/
    63  cp etcd-v3.2.26-linux-amd64/etcdctl /usr/bin/
    70  etcd --listen-client-urls http://0.0.0.0:2379 --advertise-client-urls http://localhost:2379 &> /etc/kubernetes/etcd.log &
    68  etcdctl cluster-health
    71  ps -au | grep etcd
    72  etcdctl cluster-health

# 05 Install apiserver

    73  kubectl get all --all-namespaces
    74  # update the drawing
    75  # let's install the api-server
    76  ip a
    77  # be careful with the CIDR
    81  kube-apiserver --etcd-servers=http://localhost:2379 --service-cluster-ip-range=192.168.0.0/16 --bind-address=0.0.0.0 --insecure-bind-address=0.0.0.0 &> /etc/kubernetes/apiserver.log &
    86  ps -au | grep apiserver
    85  cat /etc/kubernetes/apiserver.log
    87  curl http://localhost:8080/api/v1/nodes
    88  curl http://localhost:8080/api/v1/
    89  # update drawing with the apiserver

# 06 Configure kube config
    90  kubectl cluster-info
    91  kubectl config view
    92  kubectl config set-cluster kube-from-scratch --server=http://localhost:8080
    93  kubectl config view
    94  kubectl config set-context kube-from-scratch --cluster=kube-from-scratch
    95  kubectl config view
    96  kubectl config use-context kube-from-scratch
    97  kubectl config view
    98  cat .kube/config
    99  kubectl cluster-info

# 07 Install the scheduler
    100  kubectl get all --all-namespaces
    101  pkill -f kubelet
    104  kubelet --register-node --kubeconfig=".kube/config" &> /etc/kubernetes/kubelet.log &
    105  ps -au | grep kubelet
    106  head /etc/kubernetes/kubelet.log
    107  kubectl get no
    108  # kubelet is the node
    109  docker ps
    110  # no pods
    112  cp /etc/kubernetes/manifests/kubelet-test.yaml .
    114  mv kubelet-test.yaml kube-test.yaml

---
kube-test.yaml

    apiVersion: v1
    kind: Pod
    metadata:
      name: kube-test
      labels:
        app: kube-test
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - name: http
          containerPort: 80
          protocol: TCP
---

    115  vim kube-test.yaml
    116  kubectl create -f kube-test.yaml
    117  kubectl get po
    118  # it is pending, why ???????
    119  # we can watch it all day
    120  # with one node it should be an easy task, but this is a distributed system
    121  # let's start the scheduler
    122  kube-scheduler --master=http://localhost:8080/ &> /etc/kubernetes/scheduler.log &
    123  ps -au | grep kube-scheduler
    124  head /etc/kubernetes/scheduler.log
    125  kubectl get po
    126  # yipppi, it is running
    129  # update the drawing withe the scheduler

# 08 Install the controller manager

    127  kubectl delete po --all
    128  kubectl get po
    133  ls
    134  cp kube-test.yaml replica-test.yaml

---
replica-test.yaml

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: replica-test
    spec:
      replicas: 3
      selector:
        matchLabels:
          app: replica-test
      template:
        metadata:
          name: replica-test
          labels:
            app: replica-test
        spec:
          containers:
          - name: nginx
            image: nginx
            ports:
            - name: http
              containerPort: 80
              protocol: TCP
---

    135  vim replica-test.yaml
    136  kubectl create -f replica-test.yaml
    137  kubectl get deploy
    138  kubectl get po
    139  kube-controller-manager --master=http://localhost:8080 &> /etc/kubernetes/controller-manager.log &
    140  ps -au | grep controller
    141  head /etc/kubernetes/controller-manager.log
    142  kubectl get deploy
    143  kubectl get po
    144  # controller manager makes k8s self healing, resilient

# 09 Install the proxy

    145  ls
    146  cp replica-test.yaml service-test.yaml

---
service-test.yaml

    apiVersion: v1
    kind: Service
    metadata:
      name: replica-test
    spec:
      type: ClusterIP
      selector:
        app: replica-test
      ports:
      - name: http
        port: 80
---

    147  vim service-test.yaml
    150  kubectl create -f service-test.yaml
    152  kubectl get svc -o wide
    155  curl 192.168.21.152:80
    156  # it is never going to work !!!!!!
    157  kube-proxy --master=http://localhost:8080/ &> /etc/kubernetes/proxy.log &
    158  ps -au | grep proxy
    159  head /etc/kubernetes/proxy.log
    160  kubectl get svc -o wide
    161  curl 192.168.21.152:80
