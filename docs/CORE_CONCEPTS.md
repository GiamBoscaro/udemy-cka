# CORE CONCEPTS

## ETCD

Key-Value Store per sistemi distribuiti. Contiene informazioni su Nodi, Pods, Segreti, Ruoli ecc. Tutte le informazioni richiamate da `kubectl get` arrivano dall'ETCD Server, come anche tutte le informazioni lette dall'API Server per conoscere lo stato del cluster.

Leggere e scrivere:

```bash
./etcdctl set key1 value1
./etcdctl get key1
```

Elenco di tutte le chiavi presenti:

```bash
etcdctl get / --prefix -keys-only
```

* ### Versione API ETCD

Ci sono due versioni supportate delle API di `etcd`, la versione 2 e 3. È necessario impostare correttamente la versione nelle variabili d'ambiente essendo i comandi molto diversi tra di loro.
Per impostare la versione:

```bash
export ETCDCTL_API=3
```

```bash
# Versione 2
etcdctl backup
etcdctl cluster-health
etcdctl mk
etcdctl mkdir
etcdctl set

# Versione 3
etcdctl snapshot save 
etcdctl endpoint health
etcdctl get
etcdctl put
```

*Nota*: se nessuna versione è specificata, verrà presa come default la versione 2.

* ### Installazione da zero

Installazione ed esecuzione:

```bash
wget -q --https-only "https://github.com/etcd-io/etcd/releases/download/v3.3.11/etcd-v3.3.11-linux-amd64.tar.gz"
tar xvzf etcd-v3.3.11-linux-amd64.tar.gz
./etcd
```

Configurare ed aggiornare etcd: <https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/>

Le opzioni più importanti per la configurazione sono:

```bash
# Url a cui è raggiungibile etcd
--advertise-client-urls https://${INTERNAL_IP}:2379
```

* ### Installazione con kubeadm

`kubeadm` istanzia un Pod che esegue `etcd` nel namespace `kube-system`.
Il Pod si chiamerà `etcd-master` e si può vedere chiamando:

```bash
kubectl get pods -n kube-system
```

I comandi posso essere inviati al Pod con:

```bash
kubectl exec etcd-master -n kube-system etcdctl <comando>
```

* ### ETCD Cluster

Per configurare un cluster di `etcd` è necessario impostare i seguenti parametri:

```bash
# Url delle varie istanze etcd nel cluster
--initial-cluster controller-0=https://${CONTROLLER_IP}:2380,controller-1=https://${CONTROLLER_IP}:2380,#...
```

*Nota*: la porta 2380 è per la comunicazione tra `etcd` e `etcd`, mentre la porta 2379 è tra `client` ed `etcd`.

## Kube-API Server

È la componente principale di Kubernetes. Si occupa di autenticare e validare le richieste, scaricare e aggiornare i dati nell'ETCD (ed è l'unico servizio a poterlo fare). Può comandare i `kubelet` e viene utilizzato da altri servizi per conoscere lo stato del cluster e agire di conseguenza, sempre richiedendo l'azione all'API server.
Ogni comando inviato via `kubectl` passa dal Kube-API Server. Le sue API possono anche essere richiamate con una classica chiamata GET e POST via API REST.

* ### Installazione da zero

Installazione:

```bash
wget https://storage.googleapis.com/kubernetes-release/release/v1.13.0/bin/linux/amd64/kube-apiserver
```

Le opzioni più importanti per la configurazione sono:

```bash
# Elenco dei server ETCD
--etcd-servers=https://${CONTROLLER_IP}:2379,https://${CONTROLLER_IP}:2379,#...
```

ed inoltre altre impostazioni varie per i certificati SSL.
La configurazione attuale dell'API Server è visibile col comando:

```bash
cat /etc/systemd/system/kube-apiserver.service
```

* ### Installazione con kubeadm

`kubeadm` istanzia un Pod che esegue `kube-apiserver` nel namespace `kube-system`.
Il Pod si chiamerà `kube-apiserver-master` e si può vedere chiamando:

```bash
kubectl get pods -n kube-system
```

Le specifiche del Pod che contiene l'API Server sono presenti in `/etc/kubernetes/manifests/kube-apiserver.yaml`.

## Kube Controller Manager

Gestisce vari controller, che monitorano lo stato dei componenti del sistema e cercano di mantenere sempre il sistema allo stato desiderato. I controller ottengono lo stato dei componenti ed eseguono le azioni passando sempre attraverso l'API Server.

* ### Installazione da zero

Installazione:

```bash
wget https://storage.googleapis.com/kubernetes-release/release/v1.13.0/bin/linux/amd64/kube-controller-manager
```

Le opzioni più importanti per la configurazione sono:

```bash
# Ogni quanti secondi viene verificato lo stato dei nodi
--node-monitor-period=5s
# Dopo quanti secondi un nodo viene considerato unreachable se l'health check fallisce
--node-monitor-grace-period=40s
# Tempo lasciato al Pod per tornare operativo prima che venga rimpiazzato
--pod-eviction-timeout=5m0s
# Elenco dei controller abilitati. Di default sono tutti abilitati. È consigliato tenerli sempre tutti abilitati
--controllers * # foo per abilitare, -foo per disabilitare
```

La configurazione attuale del Kube Controller Manager è visibile col comando:

```bash
cat /etc/systemd/system/kube-controller-manager.service
```

* ### Installazione con kubeadm

`kubeadm` istanzia un Pod che esegue `kube-controller-manager` nel namespace `kube-system`.
Il Pod si chiamerà `kube-controller-manager-master` e si può vedere chiamando:

```bash
kubectl get pods -n kube-system
```

Le specifiche del Pod che contiene il Controller Manager sono presenti in `/etc/kubernetes/manifests/kube-controller-manager.yaml`.

## Kube Scheduler

Decide a che Nodo assegnare i Pod, in funzione delle risorse richieste, risorse disponibili ecc. con l'aiuto di una funzione che permette di scegliere il Nodo compatibile migliore.
È possibile sviluppare ed utilizzare uno Scheduler personalizzato.

* ### Installazione da zero

Installazione:

```bash
wget https://storage.googleapis.com/kubernetes-release/release/v1.13.0/bin/linux/amd64/kube-scheduler
```

Le opzioni più importanti per la configurazione sono:

```bash
# Path della configurazione dello scheduler
--config=path/to/kube-scheduler.yaml
```

La configurazione attuale del Kube Scheduler è visibile col comando:

```bash
cat /etc/systemd/system/kube-scheduler.service
```

* ### Installazione con kubeadm

`kubeadm` istanzia un Pod che esegue `kube-scheduler` nel namespace `kube-system`.
Il Pod si chiamerà `kube-scheduler-master` e si può vedere chiamando:

```bash
kubectl get pods -n kube-system
```

Le specifiche del Pod che contiene lo Scheduler sono presenti in `/etc/kubernetes/manifests/kube-scheduler.yaml`.

## Kubelet

Gestisce le operazioni nei Nodi, come la registrazione dei Nodi nel Cluster, il deployment dei Pods all'interno del Nodo, l'invio dei report dello stato all'API Server.

* ### Installazione da zero

Installazione:

```bash
wget https://storage.googleapis.com/kubernetes-release/release/v1.13.0/bin/linux/amd64/kubelet
```

La configurazione attuale del Kube Scheduler è visibile col comando:

```bash
cat /etc/systemd/system/kubelet.service
```

* ### Installazione con kubeadm

`kubeadm` non installa `kubelet` nei nodi. È necessario sempre installare manualmente `kubelet`.

## Kube Proxy

Permette ai tutti i Pod di comunicare tra di loro. Ogni volta che un servizio viene creato, i Kube Proxy aggiornano le loro IP Table interne in modo da mettere in comunicazione i nodi definiti nel servizio, e di conseguenza i Pod contenuti nei nodi.

* ### Installazione da zero

Installazione:

```bash
wget https://storage.googleapis.com/kubernetes-release/release/v1.13.0/bin/linux/amd64/kube-proxy
```

La configurazione attuale del Kube Scheduler è visibile col comando:

```bash
cat /etc/systemd/system/kube-proxy.service
```

* ### Installazione con kubeadm

`kubeadm` istanzia un Pod per ogni nodo che esegue `kube-proxy` nel namespace `kube-system`.
Il Pod si chiamerà `kube-proxy-<id>` e si può vedere chiamando:

```bash
kubectl get pods -n kube-system
```

## References

1. [CKA Course - Core Concepts](https://github.com/kodekloudhub/certified-kubernetes-administrator-course/tree/master/docs/02-Core-Concepts)
2. [Kubernetes Documentation - Getting Started](https://kubernetes.io/docs/setup/)
3. [Kubernetes Documentation - Kubeadm Install](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
4. [etcd.io](https://etcd.io/docs/)
