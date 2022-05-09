# CLUSTER MAINTENANCE

## Manutenzione e Aggiornamenti i Nodi

Se un nodo diventa irraggiungibile, i Pod al suo interno non sono più utilizzabili. Se fanno parte di un Replica Set, il servizio non dovrebbe essere compromesso, perchè ci sono altre repliche in altri nodi. Se invece il Pod era unico, il servizio potrebbe non funzionare e il Pod deve essere rischedulato in un altro nodo. Il cluster aspetta 5 minuti prima di segnare il nodo come irraggiungibile e schedulare i Pod in un altro nodo. È possibile cambiare questo timeout nel `kube-controller-manager`:

```bash
kube-controller-manager --pod-eviction-timeout=5mOs
```

Potrebbe essere però necessario spegnere o riavviare un nodo per effettuare operazioni di manuntenzione come degli aggiornamenti. In questo caso è consigliabile spostare tutti i Pod, e quindi tutto il carico di lavoro, su altri nodi prima di procedere con le operazioni. Per farlo è possibile seguire questo procedimento:

1. `drain` dei Pod: tutti i Pod in esecuzione nel nodo vengono spenti e rischedulati in altri nodi. Inoltre, il nodo viene segnato come *cordoned*, e quindi nessun Pod può più venire schedulato nel nodo. Questo permette di operare nel nodo in tutta sicurezza, in quanto nessun servizio è in esecuzione su di esso.

```bash
kubectl drain <nome-nodo>
```

Il `drain` potrebbe andare in errore se ci sono Daemon Sets attivi nel nodo (come molto probabilmente sarà). Per cui aggiungere l'opzione `--ignore-daemonsets`.

*Nota*: il *drain* funziona solo per Pod che sono controllati da un *ReplicaSet*, *Deployment*, *StatefulSet*, *DaemonSet*, *ReplicationController* e *Job*. Si può forzare il *drain* con `--force` ma i Pod *standalone* verranno cancellati completamente.

2. Una volta completate le operazioni, anche dopo un riavvio del nodo, esso continua a rimanere *cordoned*. Per poter permettere nuovamente la schedulazione:

```bash
kubectl uncordon <nome-nodo>
```

A questo punto, il nodo è di nuovo aperto per la schedulazione, ma i Pod che erano presenti precedentemente non verranno automaticamente riportati alla posizione originale.
Potrebbe essere utile anche rendere un nodo *cordoned* senza fare un *drain*. Per fare cioò

```bash
kubectl cordon <nome-nodo>
```

## Aggiornamento del Cluster

Kubernetes segue un sistema di versionamento standard. Tutti i componenti di Kubernetes dovrebbero essere aggiornati assieme e avranno la stessa versione. Altre dipendenze di Kubernetes come `etcd` e `coredns` potrebbero invece avere una versione differente. `kubeadm` segue invece il versionamento di Kubernetes.
Nessun componente può essere ad una versione superiore di `kube-apiserver`.

* `controller-manager` e `kube-scheduler`: possono essere ad una versione precedente.
* `kubelet` e `kube-proxy`: possono essere a due versioni precedenti.
* `kubectl`: può essere ad una versione precedente, ma anche ad una versione successiva.

Kubernetes supporta solo __le ultime tre versioni__, per cui è necessario tenere aggiornato il cluster, possibilmente all'ultima e penultima versione. È consigliato anche procedere all'aggiornamento procedendo a step, una minior version alla volta, senza saltare all'ultima versione direttamente.

Ci sono tre modi per aggiornare il cluster:

1. Automaticamente, se si utilizza un cloud provider.
2. Manualmente, tutti i componenti assieme con `kubeadm`.
3. Manualmente, componente per componente.

### Procedura di Aggiornamento

Il procedimento per aggiornare il cluster è il seguente:

1. Aggiornamento del *Master Node*. Durante l'aggiornamento il nodo non sarà raggiungibile, e non funzionerà nemmeno `kubectl`. Gli altri nodi rimarranno attivi e i Pod al loro interno continueranno a funzionare senza impattare l'applicazione.
2. Aggiornamento dei *Worker Nodes*. Si possono aggiornare tutti assieme, ma i servizi non saranno raggiungibile. Una strategia migliore è usare il `drain` e aggiornarli uno alla volta. È anche possibile creare direttamente dei nodi aggiornati, trasferire i Pod ed eliminare quelli vecchi. Questo è particolarmente facile se si utilizza un cloud provider.

### Aggiornamento con kubeadm

Per mostrare gli aggiornamenti disponibili:

```bash
kubeadm upgrade plan
```

*Nota*: `kubeadm` non aggiorna `kubelet`, che dovrà essere aggiornato manualmente successivamente.

Per procedere all'aggiornamento è necessario aggiornare prima `kubeadm`, sempre a step, una minor version alla volta. Per cui il procedimento completo è il seguente:

1. Aggiornare `kubeadm` del *Master* alla minor successiva:

```bash
apt-get upgrade -y kubeadm=1.12.0-00
```

*Nota*: `kubeadm` andrà poi aggiornato anche nei *Worker Nodes*.

2. Aggiornare il cluster alla minor successiva, uguale alla versione di `kubeadm`:

```bash
kubeadm upgrade apply 1.12.0
```

3. Aggiornare `kubelet` in ogni nodo. Se nel *Master* è presente `kubelet`, aggiornare prima questo:

```bash
apt-get upgrade -y kubelet=1.12.0-00
systemctl restart kubelet
```

Aggiornare poi i *Worker Nodes*, uno alla volta, usando `drain`. I *Worker Node* necessitano anche dell'aggiornamento di `kubeadm`:

```bash
kubectl drain <nome-nodo>

apt-get upgrade -y kubeadm=1.12.0-00
apt-get upgrade -y kubelet=1.12.0-00
kubeadm upgrade node config --kubelet-version v1.12.0
systemctl restart kubelet

kubectl uncordon <nome-nodo>
```

1. [CKA Course - Cluster Maintenance](https://github.com/kodekloudhub/certified-kubernetes-administrator-course/tree/master/docs/06-Cluster-Maintenance)
2. [The Kubernetes API](https://kubernetes.io/docs/concepts/overview/kubernetes-api/)
3. [Backing up an etcd Cluster](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#backing-up-an-etcd-cluster)
4. [etcd Recovery](https://github.com/etcd-io/website/blob/main/content/en/docs/v3.5/op-guide/recovery.md)