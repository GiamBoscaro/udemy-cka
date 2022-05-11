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

*Nota*: per conoscere l'esatta versione da indicare, utilizzare il comando:

```bash
apt update
apt-cache madison kubeadm
```

Il comando elenca tutte le versioni disponibili di `kubeadm`. Selezionare l'ultima patch della minor desiderata.

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
kubeadm upgrade node
systemctl restart kubelet

kubectl uncordon <nome-nodo>
```

*Nota*: la procedura completa e dettagliata è visibile nella documentazione [Upgrading kubeadm clusters](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/).

## Backup e Ripristino

Gli oggetti di Kubernetes possono essere creati in modo imperativo o dichiarando dei manifest. Se tutto il cluster viene configurato tramite manifest, è facile eseguire un backup in quando basta pushare in una repository tutti i manifest. In caso sia necessario ripristinare il cluster, basta deployare tutti i manifest con `kubectl apply`.
Non è detto però che tutto il cluster sia stato creato in modo dichiarativo, alcuni oggetti potrebbero essere in esecuzione ma non documentati in un manifest.

### Backup delle Risorse con kubeapi-server

Se si vuole fare un backup del cluster, si può impostare nel *cron* uno script che esegue:

```bash
kubectl get all --all-namespaces -o yaml > all-deploy-services.yaml
```

*Nota*: questo script fa il backup solo di alcune risorse del cluster. Ad esempio, il contenuto dei volume non viene salvato.

*Nota*: esistono servizi automatizzati per il backup del cluster, come [Velero](https://velero.io).

### Backup di etcd

`etcd` cluster contiene tutte le risorse e lo stato del cluster. Al posto di fare il backup delle risorse in formato *yaml*, è possibile fare uno snapshot di tutto `etcd` database:

```bash
etcdctl snapshot save snapshot.db
# per verificare lo stato dello snapshot
etcdctl snapshot status snapshot.db
```

*Nota*: ricordarsi si specificare la versione dell'API di `etcd` prima dell'esecuzione di un qualsiasi comando:

```bash
export ETCDCTL_API=3
```

### Ripristino di etcd

Per ripristinare da uno snapshot:

```bash
service kube-apiserver stop
etcdctl snapshot restore snapshot.db --data-dir /var/lib/etcd-from-backup
```

Nel ripristino, `etcd` inizializza una nuova configurazione del cluster, quindi viene creata una nuova cartella per la configurazione (`data-dir`). È necessario indicare al servizio `etcd.service` la nuova cartella. Una volta riconfigurato il servizio, eseguire:

```bash
systemctl daemon-reload
service etcd restart
service kube-apiserver start
```

Il ripristino dovrebbe ora essere completo.

*Nota*: per tutti i comandi `etcdctl` è necessario specificare delle opzioni per l'autenticazione quando l'`etcd` ha il TLS abilitato:

```bash
ETCDCTL_API=3 etcdctl \
    snapshot save snapshot.db \
    --endpoints=https://127.0.0.1:2379 \
    --cacert=/etc/etcd/ca.crt \
    --cert=/etc/etcd/etcd-server.crt \
    --key=/etc/etcd/etcd-server.key
```

*Nota*: se si utilizza un cloud provider, probabilmente non si avrà accesso all'`etcd`, per cui l'unica soluzione rimane il backup delle risorse tramite `kubeapi-server`.

### Ripristino di etcd con kubeadm

Se si utilizza `kubeadm`, `etcd` viene eseguito in un Pod. Dopo aver ripristinato lo snapshot in una nuova cartella, è possibile aggiornare `etcd` semplicemente cambiando il suo file *yaml* in `/etc/kubernetes/manifests`. L'unica modifica da fare è:

```yaml
# ...
volumes:
- hostPath:
    path: /var/lib/<note_nuova_data_dir>
    type: DirectoryOrCreate
# ...
```

__non serve più modificare --data-dir__ tra le opzioni di avvio del servizio.
Una volta modificato il manifest, essendo `etcd` un DaemonSet, il *Master Node* noterà le modifiche e le applicherà al cluster.

### Backup e Ripristino da una posizione diversa dal Master Node

Si può effettuare il backup e ripristino dell'`etcd` da una qualsiasi posizione, l'importante è specificare l'endpoint e i certificati necessari all'esecuzione di `etcdctl`.

```bash
ETCDCTL_API=3 etcdctl \
    snapshot restore snapshot.db \
    --endpoints=https://<master_node>:2379 \
    --cacert=/etc/etcd/ca.crt \
    --cert=/etc/etcd/etcd-server.crt \
    --key=/etc/etcd/etcd-server.key \
    --data-dir="path/to/data/dir" \
    --initial-advertise-peer-urls="⟨http://<master_node>:2380"⟩ \
    --initial-cluster="default=⟨http://<master_node>:2380"⟩ \
    --initial-cluster-token="<token>" \
    --name="default"
```

Per sapere con sicurezza quali valori inserire nel comando, è consigliato controllare il manifest del Pod che esegue `etcd` nel *Master Node*:

```bash
kubectl get pod etcd-controlplane -n kube-system -o yaml
```

1. [CKA Course - Cluster Maintenance](https://github.com/kodekloudhub/certified-kubernetes-administrator-course/tree/master/docs/06-Cluster-Maintenance)
2. [The Kubernetes API](https://kubernetes.io/docs/concepts/overview/kubernetes-api/)
3. [Upgrading kubeadm clusters](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)
4. [Backing up an etcd Cluster](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#backing-up-an-etcd-cluster)
5. [etcd Recovery](https://github.com/etcd-io/website/blob/main/content/en/docs/v3.5/op-guide/recovery.md)