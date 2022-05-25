# INSTALL KUBERNETES WITH KUBEADM

## Deployment con Kubeadm

Per installare e configurare un cluster con `kubeadm` sono necessari alcuni prerequisiti:

1. SO Supportato: una tra le distribuzioni linux supportate.
2. 2GB di RAM per ogni macchina.
3. 2 CPU.
4. Un network attivo che colleghi tutti i nodi.
5. Hostname, MAC address ecc. devono tutti essere univoci.
6. [Le porte richieste](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#check-required-ports) devono essere aperte.
7. Swap __deve essere disabilitato__.

Se i prerequisiti sono rispettati è possibile proseguire con l'installazione:

* Abilitare il *bridged traffic* nelle IP tables:

```bash
sudo modprobe br_netfilter
```

* Configurare i nuovi parametri del Kernel:

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
```

* Installare la Container Runtime (es: [Docker](https://docs.docker.com/engine/install/#server)):

```bash
sudo apt-get update

sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

* Installare i componenti di Kubernetes:

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl

sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gp

echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

Una volta installati i componenti, si può iniziare ad installare e configurare il cluster. I seguenti comandi vanno eseguiti nel master node:

* Inizializzare il Control Plane:

```bash
# da eseguire come root
kubeadm init \
--pod-network-cidr=10.244.0.0/16 \ # IP utilizzati dai Pod
--apiserver-advertise-address=192.168.56.2 # IP dell'Api Server (in genere IP del master)
```

* Configurare `kubectl` per funzionare per gli utenti non root:

```bash
# da eseguire come normale user
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

* Installare il plugin per la gestione del network (CNI). Per esempio, per Weave:

```bash
# da eseguire come normale user
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

*Nota*: tutti i plugin e le istruzioni per l'installazione sono disponibili [nella documentazione ufficiale](https://kubernetes.io/docs/concepts/cluster-administration/addons/#networking-and-network-policy)

* Unire i nodi al network del cluster:

```bash
kubeadm join --token <token> <control-plane-host>:<control-plane-port> --discovery-token-ca-cert-hash sha256:<hash>
```

Il comando, con tutti i valori pre inseriti, viene mostrato alla fine del processo di inizializzazione (`kubedm init`).
Se non ci si ricorda il token, è possibile vederlo con:

```bash
kubeadm token list
# se è scaduto bisogna ricrearlo:
kubeadm token create
```

Per lo SHA265 del certificato:

```bash
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | \
   openssl dgst -sha256 -hex | sed 's/^.* //'
```

* Verificare lo stato dei nodi:

```bash
kubectl get nodes
```

1. [CKA Course - Install Kubernetes the Kubeadm Way](https://github.com/kodekloudhub/certified-kubernetes-administrator-course/tree/master/docs/11-Install-Kubernetes-the-kubeadm-way)
2. [Installing Kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
3. [Setup Files and Scripts Repo](https://github.com/kodekloudhub/certified-kubernetes-administrator-course)
4. [CNI Plugins](https://kubernetes.io/docs/concepts/cluster-administration/addons/#networking-and-network-policy)
