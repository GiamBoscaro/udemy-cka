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

Ogni processo in esecuzione nel nodo può essere in ascolto in una porta. Per vedere a quali porte è assosciato un processo:

```bash
netstat -nplt
```

Per conoscere esattamente tutte le caratteristiche della rete a cui il nodo è connesso, è possibile utilizzare `ipcalc` (da installare con `apt`):

```bash
ipcalc <ip> # es: ip nodo 10.49.70.3/24
# OUTPUT:
Address:   10.49.70.3           
Netmask:   255.255.255.0 = 24   
Wildcard:  0.0.0.255            
=>
Network:   10.49.70.0/24 # network dei nodi     
HostMin:   10.49.70.1 # IP minimo
HostMax:   10.49.70.254 # IP massimo
Broadcast: 10.49.70.255         
Hosts/Net: 254  
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

## Network dei Servizi

In genere nel cluster i Pod non comunicano direttamente ma attraverso i Service. I Pod infatti sono effimeri, e vengono creati e distrutti spesso. Il loro IP quindi non è stabile, cambia nel tempo. Un servizio ha un IP statico ed è un oggetto virtuale condiviso in tutto il cluster (non appartiene ad un nodo), e viene gestito da `kube-proxy`. Quando un servizio viene creato `kube-proxy` gli assegna un IP che inoltra tutte le chiamate ricevute (alla porta del servizio) ai Pod collegati al esso.

Ci sono diversi modi in cui `kube-proxy` creare le regole di inoltro, dette *proxy modes*:

* `userspace`: `kube-proxy` rimane in ascolto di connessioni. Quando una nuova connessione viene richiesta, il `kube-proxy` chiude la connessione originale e ne crea una nuova, che gestisce l'inoltro delle richieste. Tutto passa quindi per `kube-proxy`. I log vengono salvati su `/var/log/kube-proxy`.
* `ipvs`: funziona come un load balancer, distribuendo le richieste ai servizi interni al cluster.
* `iptables`: le richieste vengono inoltrate direttamente al servizio destinatario tramite le iptables del sistema operativo dell'host. È più veloce ma il debugging è più difficile in quando i log sono visibili solo nel kernel dell'host.

La modalità può essere impostata durante la configurazione di `kube-proxy` usanto l'opzione `--proxy-mode`. Di default viene usata la modalità `iptables`.

*Nota*: il range di IP dei Servizi e dei Pod __non__ devono essere sovrapposti. Per impostare i range di IP dei Servizi gestiti dal `kube-proxy` utilizzare l'opzione `--service-cluster-ip-range`.

Le regole che vengono create dal `kube-proxy` per gestire i servizi sono visibili col comando:

```bash
iptables -L -t nat | grep <nome_servizio>
```

Per controllare i log del `kube-proxy` è possibile anche verificare i log all'interno del Pod:

```bash
kubectl logs <kube_proxy_pod> -n kube-system
```

## DNS in Kubernetes

I Pod si connettono tra di loro in genere attraverso i Servizi. Possono connettersi tramite l'IP del servizio ma è anche possibile (e consigliato) connettersi semplicemente attraverso il nome del servizio. Per poter fare ciò, Kubernetes utilizza un DNS che risolve il nome del servizio col suo IP.

Se la comunicazione avviene tra due namespace diversi, è necessario specificare il namespace oltre che al nome del servizio (`http://<servizio>.<namespace>`). In realtà tutti i servizi vengono poi raggrupati in un grande namespace detto `type` con valore `svc`. A loro volta, tutti i servizi (e anche tutti i Pod) vengono raggruppati nel namespace di root del cluster, `cluster.local`. L'uri completa per connettersi ad un servizio (*fully qualified domain*) sarebbe quindi:

```bash
http://<servizio>.<namespace>.svc.cluster.local
```

### DNS per i Pod

Di default, Kubernetes non crea alcun DNS Record per i Pod. Se richiesto, però, è possibile abilitare questa funzione. Il DNS crea un record di questo tipo:

```bash
http://<pod>.<namespace>.pod.cluster.local
```

dove però `<pod>` __non__ è il nome del Pod, ma semplicemente l'IP del Pod dove i punti sono sostituiti con dei meno (es: *10.244.2.5* diventa *10-244-2-5*).

### CoreDNS

Kubernetes deploya un DNS server per il cluster (due Pod in replica sul Master), in cui vengono associati tutti i domini dei servizi ai loro IP. Ogni nodo dovrà sapere come raggiungere il server DNS per risolve i domini. Questo può essere fatto aggiungendo il server a `/etc/resolv.conf`:

```bash
# ...
nameserver  10.96.0.10 # ip del DNS server
# ...
```

La configurazione di CoreDNS si trova in `etch/coredns/Corefile`:

```conf
.:53 {
    errors
    health {
        lameduck 5s
    } # root domain del dell'intero cluster
    kubernetes cluster.local in-addr.arpa ip6.arpa {
       pods insecure # abilita i record DNS per i Pod
       fallthrough in-addr.arpa ip6.arpa
       ttl 30
    }
    prometheus :9153
    forward . /etc/resolv.conf
    cache 30
    loop
    reload
}
```

Il file di configurazione consiste in vari plugin che CoreDNS esegue. Il plugin che gestire il DNS di Kubernetes è `kubernetes`. È possibile modificare o aggiungere diverse opzioni per modificare il comportamento del DNS (ad es: `pods` indica di creare i record DNS anche per i Pod).
Questa configurazione viene iniettata nel Pod di CoreDNS tramite una ConfigMap, visibile con:

```bash
kubectl describe cm coredns -n kube-system
```

Quando viene creato il Pod di CoreDNS, viene anche creato il servizio per poterlo raggiungere. L'IP assegnato al servizio corrisponde al `nameserver` da inserire in `/etc/resolv.conf`. Questa modifica viene effettuata all'interno del Pod automaticamente da `kubelet` ogni volta che uno di essi viene creato.

1. [CKA Course - Networking](https://github.com/kodekloudhub/certified-kubernetes-administrator-course/tree/master/docs/09-Networking)
2. [Weave Kubernetes Addo](https://www.weave.works/docs/net/latest/kubernetes/kube-addon/)
