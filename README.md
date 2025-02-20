# ros2
# Cilium + k3s Multicasting

# step1. Install k3s

curl -sfL https://get.k3s.io | sh -s - \
  --flannel-backend=none \
  --disable-kube-proxy \
  --disable servicelb \
  --disable-network-policy \
  --disable traefik \
  --cluster-init


# step2. K3s Config

sudo chmod 600 /etc/rancher/k3s/k3s.yaml
echo "export KUBECONFIG=/etc/rancher/k3s/k3s.yaml" >> $HOME/.bashrc
source $HOME/.bashrc

Or 

mkdir -p $HOME/.kube
sudo cp -i /etc/rancher/k3s/k3s.yaml $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
echo "export KUBECONFIG=$HOME/.kube/config" >> $HOME/.bashrc
source $HOME/.bashrc


# step3. Install Cilium

CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz


API_SERVER_IP=<IP>  # 193.196.39.140 
API_SERVER_PORT=<PORT>  # 6443 default
cilium install \
  --set k8sServiceHost=${API_SERVER_IP} \
  --set k8sServicePort=${API_SERVER_PORT} \
  --set kubeProxyReplacement=true

// How to get API_SERVER_IP
ubuntu@k3s-lat-test-master:~/k3s$ kubectl get nodes -o wide
NAME                  STATUS     ROLES                       AGE   VERSION        INTERNAL-IP      EXTERNAL-IP   OS-IMAGE             
k3s-lat-test-master   NotReady   control-plane,etcd,master   13m   v1.31.5+k3s1   193.196.39.140   <none>        Ubuntu 22.04.4 LTS

KERNEL-VERSION       CONTAINER-RUNTIME
5.15.0-113-generic   containerd://1.7.23-k3s2


# step4. Add worker node into cluster

K3S_TOKEN=<TOKEN>
API_SERVER_IP=<IP>  # master node IP 193.196.39.140
API_SERVER_PORT=<PORT>   //6443
curl -sfL https://get.k3s.io | sh -s - agent \
  --token "${K3S_TOKEN}" \
  --server "https://${API_SERVER_IP}:${API_SERVER_PORT}"

// how to get token
ubuntu@k3s-lat-test-master:~/k3s$ sudo cat /var/lib/rancher/k3s/server/token


# step5. Create ip-pool.yaml

#ip-pool.yaml
apiVersion: "cilium.io/v2alpha1"
kind: CiliumLoadBalancerIPPool
metadata:
  name: "first-pool"
spec:
  blocks:
    - start: "192.168.1.240" // server ip network segment
      stop: "192.168.1.249"

ubuntu@k3s-lat-test-master:~/k3s$ kubectl apply -f ip-pool.yaml
ciliumloadbalancerippool.cilium.io/first-pool created
ubuntu@k3s-lat-test-master:~/k3s$ kubectl get ippools
NAME         DISABLED   CONFLICTING   IPS AVAILABLE   AGE
first-pool   false      False         21              7s


# step6. Create values.yaml and announce.yaml

#announce.yaml
apiVersion: cilium.io/v2alpha1
kind: CiliumL2AnnouncementPolicy
metadata:
  name: default-l2-announcement-policy
  namespace: kube-system
spec:
  externalIPs: true
  loadBalancerIPs: true

kubectl apply -f announce.yaml


#values.yaml
k8sServiceHost: "193.196.39.140"
k8sServicePort: "6443"

kubeProxyReplacement: true

l2announcements:
  enabled: true

externalIPs:
  enabled: true

#增加 Cilium 與 Kubernetes API Server 的請求速率限制
k8sClientRateLimit:
  qps: 50     # (Queries Per Second）表示每秒最多 50 個請求
  burst: 200   # API 允許的瞬時最大請求數（突發流量）

operator:
  #如果只有一台 Node，取消註解這行
  #replicas: 1  
  rollOutPods: true

rollOutCiliumPods: true

ingressController:
  enabled: true
  default: true
  loadbalancerMode: shared
  nodeSelector:
    kubernetes.io/os: linux  #讓所有 Linux 節點（包括 Worker）運行 Cilium Ingress
  service:
    annotations: {}  # 刪除 `io.cilium/lb-ipam-ips`
      #io.cilium/lb-ipam-ips: 193.196.39.240     #如果真的需要手動指定 IP，選擇一個 193.196.39.x 或 193.196.39.x 範圍內的可用 IP



# step7. Check 

ubuntu@k3s-lat-test-master:~/k3s$ kubectl get services --all-namespaces
kube-system   cilium-ingress   LoadBalancer   10.43.70.124    192.196.39.151   80:32424/TCP,443:31854/TCP   26s

ubuntu@k3s-lat-test-master:~/k3s$ kubectl apply -f https://blog.stonegarden.dev/articles/2024/02/bootstrapping-k3s-with-cilium/resources/smoke-test.yaml
namespace/whoami created
deployment.apps/whoami created
service/whoami created
ingress.networking.k8s.io/whoami created

ubuntu@k3s-lat-test-master:~/k3s$ kubectl get service -n whoami
NAME     TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)        AGE
whoami   LoadBalancer   10.43.173.106   192.196.39.152   80:30169/TCP   8s
ubuntu@k3s-lat-test-master:~/k3s$ curl 192.196.39.152
Hostname: whoami-b69cc7dbb-85z4z
IP: 127.0.0.1
IP: ::1
IP: 10.0.0.64
IP: fe80::e444:bff:fe59:461b
RemoteAddr: 10.0.0.56:45630
GET / HTTP/1.1
Host: 192.196.39.152
User-Agent: curl/7.81.0
Accept: */*

ubuntu@k3s-lat-test-master:~/k3s$ kubectl get service -n kube-system cilium-ingress 
NAME             TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)                      AGE
cilium-ingress   LoadBalancer   10.43.70.124   192.196.39.151   80:32424/TCP,443:31854/TCP   2m30s

ubuntu@k3s-lat-test-master:~/k3s$ curl --header 'Host: whoami.local' 192.196.39.151
Hostname: whoami-b69cc7dbb-85z4z
IP: 127.0.0.1
IP: ::1
IP: 10.0.0.64
IP: fe80::e444:bff:fe59:461b
RemoteAddr: 10.0.0.112:41005
GET / HTTP/1.1
Host: whoami.local
User-Agent: curl/7.81.0
Accept: */*
X-Envoy-Internal: true
X-Forwarded-For: 193.196.39.140
X-Forwarded-Proto: http
X-Request-Id: 51bbe789-f295-4c18-9b18-f9f62da6c300


# step8. Install cilium dbg

sudo apt update && sudo apt install -y \
  clang-15 llvm-15 gcc-multilib \
  make libelf-dev iproute2 iptables \
  jq git bpfcc-tools libbpf-dev \
  python3 python3-pip

#install go 
wget https://go.dev/dl/go1.21.5.linux-amd64.tar.gz
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf go1.21.5.linux-amd64.tar.gz
rm go1.21.5.linux-amd64.tar.gz
echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.bashrc
echo 'export GOPATH=$HOME/go' >> ~/.bashrc
echo 'export PATH=$PATH:$GOPATH/bin' >> ~/.bashrc
source ~/.bashrc
go version


make cilium-dbg
sudo cp cilium-dbg/cilium-dbg /usr/local/bin/cilium-dbg
sudo chmod +x /usr/local/bin/cilium-dbg



# step9. Add multicast group

Follow
https://docs.cilium.io/en/latest/network/multicast/#enable-multicast


# step10. Apply ros2-cilium.yaml

ubuntu@k3s-lat-test-master:~/k3s$ kubectl apply -f ros2-cilium.yaml


# step11. Adding Config Map (optional)
If there is something wrong with the communicaton, use config map.











