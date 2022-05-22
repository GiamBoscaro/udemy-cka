# NETWORKING

## Network dei Nodi

L'infrastruttura di rete dei nodi nel cluster è al di fuori delle competenze di Kubernetes.
L'IP dei nodi può essere verificato con:

```bash
kubectl get nodes -o wide
# oppure
kubectl describe node <nome_nodo>
```

Conoscendo l'IP del Pod, è possibile vedere anche a quale interfaccia del network è connesso con:

```bash
ip link
# verificare a quale interfaccia è associato l'IP, ad esempio:
eth0@if16477: ......... inet 10.3.116.12 ........
```

da cui si può anche verificare il MAC Address dell'interfaccia:

```bash
ip link show eth0
```

Per verificare il MAC Address di un nodo che non è quello in cui stiamo eseguendo i comandi:

```bash
arp <nome_nodo>
```

Ogni processo in esecuzione nel nodo può essere in ascolto in una porta. Per vedere a quali porte è assicato un processo:

```bash
netstat -nplt
```

## Network dei Pod

Kubernetes non offre una soluzione integrata per gestire la comunicazione tra i Pod, ma ne definisce solo le regole:

1. Ogni Pod deve avere un IP.
2. Ogni Pod deve poter comunicare con tutti gli altri Pod nel nodo.
3. Ogni Pod deve poter comunicare con tutti gli altri Pod in tutti gli altri nodi, senza NAT.

Esistono varie soluzioni per gestire il network di un cluster Kubernetes: [flannel](https://github.com/flannel-io/flannel), cilium, [weaveworks](https://www.weave.works) ecc.

Tutte le soluzioni si basano sul concetto di Network Bridge e Interfacce Virtuali.

1. All'interno di ogni nodo, viene creato un network bridge per i Pod:

```bash
# uno per ogni nodo
ip link add v-net-0 type bridge
ip link set dev v-net-0 up # attiva l'interfaccia
```

2. Viene assegnato un IP al bridge di ogni nodo. L'IP è diverso per ogni nodo presente:

```bash
ip -n <namespace> addr add 10.244.1.1/24 dev v-net-0
# es: 10.244.x.1/24 con x = numero nodo
```

*Nota*: il namespace __non__ è riferito ai namespace di Kubernetes, ma a quello del virtual network. Specificandolo, il comando viene eseguito all'interno di quel namespace.

3. Collegare ogni container al bridge all'interno del nodo:

```bash
# per OGNI CONTAINER all'interno del nodo
ip link add <vnet_container> type veth peer name <vnet_container_br> # crea i connettori virtuali
ip link set <vnet_container> netns <container> # connette un capo del connettore al container
ip link set <vnet_container_br> netns <v_net_0 # connette l'altro capo al bridge network
ip -n <namespace> addr add <ip_container> dev <v-net-container> # assegna un indirizzo al container
ip -n <namespace> link set <v-net-container> up # attiva la connessione
```

Tutti i container all'interno del nodo potranno ora comunicare tra di loro, ma non con l'esterno

4. Collegare il network bridge dei nodi agli altri nodi:

```bash
ip route add <ip_bridge_network> <ip_nodo>
```

Questa procedura manuale può diventare complicata su un cluster con molti nodi. Si può quindi utilizzare un router per collegare assieme tutti i network bridge e creare un unico network bridge condiviso tra tutti i nodi:

![Bridge network con router](/assets/section-9/routing_bridge_network.PNG)

Il processo appena visto può essere scriptato ed eseguito manualmente ad ogni creazione di un Pod.

## CNI (Container Network Interface)

Lo script visto in precedenza può essere automatizzato in modo che avvenga ad ogni creazione di un Pod. Per fare ciò lo script deve seguire lo standard CNI (*Container Network Interface*). Lo script deve avere un comando `add` e `delete` per aggiungere o eliminare un container in un certo namespace:

```bash
./net-script.sh add <container> <namespace>
```

`kubelet`, che gestisce la creazione dei container, può quindi poi essere configurato per eseguire questo script ad ogni creazione di un container:

```bash
kubelet \
    --cni-conf-dir=/etc/cni/net.d # indica la configurazione di networking. Al suo interno c'è il nome dello script da eseguire
    --cni-bin-dir=/etc/cni/bin # la cartella contiene gli script
```

### Gestione degli IP

È compito del plugin che implementa la CNI assegnare e gestire gli IP dei Pod, ed essere certo di non assegnare lo stesso IP a due diversi Pod allo stesso momento.
CNI implementa due plugin per per l'*IPAM*:

* `host-local`: crea un file in ogni host/nodo che tiene traccia dei contPodainer presenti e degli IP assegnati ad essi.
* `DHCP`:

*Nota*: è lo script che gestisce la creazione del network per i Pod che deve implementare la chiamata allo script ipam. Lo script può anche gestire in modo dinamico la strategia di gestione degli ip, andando a leggere il file di configurazione della CNI in `/etc/cni/net.d/<script_plugin>.conf`:

```json
// /etc/cni/net.d/plugin.conf
{
    "cniVersion": "0.2.0",
    "name": "mynet",
    "type": "bridge",
    "bridge": "cnio",
    "isGateway": true,
    "ipMasq": true,
    "ipam": {
        "type": "host-local",
        "subnet": "10.22.0.0/16",
        "routes": [
            { "dst": "0.0.0.0/0" }
        ]
    }
}
```

La proprietà `ipam.type` indica la strategia che si vuole utizzare. È compito del plugin implementarla nel modo corretto.

### Weave

Una delle soluzioni più utilizzate per il networking dei Pod è [Weave](https://www.weave.works). Weave esegue un Daemon Pod in ogni nodo del cluster che creano un piccolo network tra di loro. I Pod di Weave comunicano tra di loro per tenere aggiornata una mappa di tutti i container in esecuzione sui rispettivi nodi, e i loro indirizzi IP. In questo modo, quando un container invia un messaggio diretto all'esterno del nodo, Weave lo intercetta e gestisce lui il trasporto del messaggio al nodo corretto.

Per installare Weave:

```bash
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

Il network di default di weave è `10.32.0.0/12`. Weave suddivide tutti gli IP disponibili tra i nodi presenti nel cluster. I Pod all'iterno del nodo potranno usare il range di IP assegnato al nodo.
Questo network potrebbe andare in conflitto con gli IP del cluster. Il Pod di Weave potrebbe andare in crash loop. Per verificare i log:

```bash
kubectl logs -n kube-system weave-net-xjd5z -c weave
```

È necessario modificare la variabile d'ambiente `IPALLOC_RANGE`. Per fare cioè, appendere al manifest di Weave la variabile:

```bash
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')&env.IPALLOC_RANGE=10.50.0.0/16"
```

1. [CKA Course - Networking](https://github.com/kodekloudhub/certified-kubernetes-administrator-course/tree/master/docs/09-Networking)
2. [Weave Kubernetes Addo](https://www.weave.works/docs/net/latest/kubernetes/kube-addon/)
