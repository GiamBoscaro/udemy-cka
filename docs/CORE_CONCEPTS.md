# CORE CONCEPTS

## ETCD

Key-Value Store per sistemi distribuiti. Contiene informazioni su Nodi, Pods, Segreti, Ruoli ecc. Tutte le informazioni richiamate da `kubectl get` arrivano dall'ETCD Server, come anche tutte le informazioni lette dall'API Server per conoscere lo stato del cluster.

Leggere e scrivere:

```shell
./etcdctl set key1 value1
./etcdctl get key1
```

Elenco di tutte le chiavi presenti:

```shell
etcdctl get / --prefix -keys-only
```

* ### Versione API ETCD

Ci sono due versioni supportate delle API di `etcd`, la versione 2 e 3. È necessario impostare correttamente la versione nelle variabili d'ambiente essendo i comandi molto diversi tra di loro.
Per impostare la versione:

```shell
export ETCDCTL_API=3
```

```shell
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

```shell
wget -q --https-only "https://github.com/etcd-io/etcd/releases/download/v3.3.11/etcd-v3.3.11-linux-amd64.tar.gz"
tar xvzf etcd-v3.3.11-linux-amd64.tar.gz
./etcd
```

Configurare ed aggiornare etcd: <https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/>

Le opzioni più importanti per la configurazione sono:

```shell
# Url a cui è raggiungibile etcd
--advertise-client-urls https://${INTERNAL_IP}:2379
```

* ### Installazione con kubeadm

`kubeadm` istanzia un Pod che esegue `etcd` nel namespace `kube-system`.
Il Pod si chiamerà `etcd-master` e si può vedere chiamando:

```shell
kubectl get pods -n kube-system
```

I comandi posso essere inviati al Pod con:

```shell
kubectl exec etcd-master -n kube-system etcdctl <comando>
```

* ### ETCD Cluster

Per configurare un cluster di `etcd` è necessario impostare i seguenti parametri:

```shell
# Url delle varie istanze etcd nel cluster
--initial-cluster controller-0=https://${CONTROLLER_IP}:2380,controller-1=https://${CONTROLLER_IP}:2380,#...
```

*Nota*: la porta 2380 è per la comunicazione tra `etcd` e `etcd`, mentre la porta 2379 è tra `client` ed `etcd`.

## Kube-API Server

È la componente principale di Kubernetes. Si occupa di autenticare e validare le richieste, scaricare e aggiornare i dati nell'ETCD (ed è l'unico servizio a poterlo fare). Può comandare i `kubelet` e viene utilizzato da altri servizi per conoscere lo stato del cluster e agire di conseguenza, sempre richiedendo l'azione all'API server.
Ogni comando inviato via `kubectl` passa dal Kube-API Server. Le sue API possono anche essere richiamate con una classica chiamata GET e POST via API REST.

* ### Installazione di Kube-API Server da zero

Installazione:

```shell
wget https://storage.googleapis.com/kubernetes-release/release/v1.13.0/bin/linux/amd64/kube-apiserver
```

Le opzioni più importanti per la configurazione sono:

```shell
# Elenco dei server ETCD
--etcd-servers=https://${CONTROLLER_IP}:2379,https://${CONTROLLER_IP}:2379,#...
```

ed inoltre altre impostazioni varie per i certificati SSL.
La configurazione attuale dell'API Server è visibile col comando:

```shell
cat /etc/systemd/system/kube-apiserver.service
```

* ### Installazione di Kube-API Server con kubeadm

`kubeadm` istanzia un Pod che esegue `kube-apiserver` nel namespace `kube-system`.
Il Pod si chiamerà `kube-apiserver-master` e si può vedere chiamando:

```shell
kubectl get pods -n kube-system
```

Le specifiche del Pod che contiene l'API Server sono presenti in `/etc/kubernetes/manifests/kube-apiserver.yaml`.

## Kube Controller Manager

Gestisce vari controller, che monitorano lo stato dei componenti del sistema e cercano di mantenere sempre il sistema allo stato desiderato. I controller ottengono lo stato dei componenti ed eseguono le azioni passando sempre attraverso l'API Server.

* ### Installazione di Kube Controller Manager da zero

Installazione:

```shell
wget https://storage.googleapis.com/kubernetes-release/release/v1.13.0/bin/linux/amd64/kube-controller-manager
```

Le opzioni più importanti per la configurazione sono:

```shell
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

```shell
cat /etc/systemd/system/kube-controller-manager.service
```

* ### Installazione di Kube Controller Manager con kubeadm

`kubeadm` istanzia un Pod che esegue `kube-controller-manager` nel namespace `kube-system`.
Il Pod si chiamerà `kube-controller-manager-master` e si può vedere chiamando:

```shell
kubectl get pods -n kube-system
```

Le specifiche del Pod che contiene il Controller Manager sono presenti in `/etc/kubernetes/manifests/kube-controller-manager.yaml`.

## Kube Scheduler

Decide a che Nodo assegnare i Pod, in funzione delle risorse richieste, risorse disponibili ecc. con l'aiuto di una funzione che permette di scegliere il Nodo compatibile migliore.
È possibile sviluppare ed utilizzare uno Scheduler personalizzato.

* ### Installazione di Kube Scheduler da zero

Installazione:

```shell
wget https://storage.googleapis.com/kubernetes-release/release/v1.13.0/bin/linux/amd64/kube-scheduler
```

Le opzioni più importanti per la configurazione sono:

```shell
# Path della configurazione dello scheduler
--config=path/to/kube-scheduler.yaml
```

La configurazione attuale del Kube Scheduler è visibile col comando:

```shell
cat /etc/systemd/system/kube-scheduler.service
```

* ### Installazione di Kube Scheduler con kubeadm

`kubeadm` istanzia un Pod che esegue `kube-scheduler` nel namespace `kube-system`.
Il Pod si chiamerà `kube-scheduler-master` e si può vedere chiamando:

```shell
kubectl get pods -n kube-system
```

Le specifiche del Pod che contiene lo Scheduler sono presenti in `/etc/kubernetes/manifests/kube-scheduler.yaml`.

## Kubelet

Gestisce le operazioni nei Nodi, come la registrazione dei Nodi nel Cluster, il deployment dei Pods all'interno del Nodo, l'invio dei report dello stato all'API Server.

* ### Installazione di Kubelet da zero

Installazione:

```shell
wget https://storage.googleapis.com/kubernetes-release/release/v1.13.0/bin/linux/amd64/kubelet
```

La configurazione attuale del Kube Scheduler è visibile col comando:

```shell
cat /etc/systemd/system/kubelet.service
```

* ### Installazione di Kubelet con kubeadm

`kubeadm` non installa `kubelet` nei nodi. È necessario sempre installare manualmente `kubelet`.

## Kube Proxy

Permette ai tutti i Pod di comunicare tra di loro. Ogni volta che un servizio viene creato, i Kube Proxy aggiornano le loro IP Table interne in modo da mettere in comunicazione i nodi definiti nel servizio, e di conseguenza i Pod contenuti nei nodi.

* ### Installazione di Kube Proxy da zero

Installazione:

```shell
wget https://storage.googleapis.com/kubernetes-release/release/v1.13.0/bin/linux/amd64/kube-proxy
```

La configurazione attuale del Kube Scheduler è visibile col comando:

```shell
cat /etc/systemd/system/kube-proxy.service
```

* ### Installazione di Kube Proxy con kubeadm

`kubeadm` istanzia un Pod per ogni nodo che esegue `kube-proxy` nel namespace `kube-system`.
Il Pod si chiamerà `kube-proxy-<id>` e si può vedere chiamando:

```shell
kubectl get pods -n kube-system
```

## Pods

Il Pod è il più piccolo oggetto presente in Kubernetes, e rappresenta una singola istanza dell'applicazione. Viene eseguito nei nodi del cluster, e può contenere uno o più container.
Kubernetes può istanziare vari Pod nello stesso nodo o in nodi diversi, in funzione del carico di lavoro e dello stato di funzionamento dei vari Pod.
I Pod vengono distrutti e creati ad ogni evenienza, cambiando il loro indirizzo IP ogni volta. I Pod sono quindi effimeri.

Per vedere la lista dei Pod presenti nel cluster:

```shell
kubectl get pods
# per avere informazioni più estese
kubectl get pods -o wide
```

Per avere dettagli e informazioni varie sul Pod appena creato:

```shell
kubectl describe pod <nome_pod> # in questo caso sarebbe myapp-pod
```

### Deploy di un Pod da CLI

Un Pod può essere creato e deployato dal terminale eseguendo:

```shell
kubectl run <nome_pod> --image <path/to/docker_image>
```

e per cancellarlo:

```shell
kubectl delete <nome_pod>
```

### Deploy di un Pod da specifica YAML

La configurazione di base per il deploy di un Pod utilizzando uno *yaml* è:

```yaml
# pod-definition.yaml
apiVersion: v1
kind: Pod
metadata:
    name: myapp-pod
    labels:
        # qualsiasi key e value
        app: myapp
        type: front-end
        # ...
spec:
    containers:
      - name: nginx-container
        image: nginx # se non preso da Docker Hub, specifica l'url del registro
        # è possibile inserire anche un comando da eseguire all'avvio del container
        command:
            - sleep
            - 1000
```

Per eseguire il deploy:

```shell
kubectl apply -f pod-definition.yaml
# oppure
kubectl create -f pod-definition.yaml # create va in errore se il Pod esiste già
```

### Generare un file YAML da un Pod in esecuzione

Se un Pod è in esecuzione, sono pochissime le proprietà che possono essere cambiate (es: l'immagine del container). È quindi necessario cancellare e ricreare il Pod. È utile quindi generare il file *yaml* del Pod attualmente in esecuzione, in modo da fare le modifiche necessarie e deployarlo facilmente:

```shell
kubectl get pod <nome_pod> -o yaml > pod-definition.yaml
```

*Nota*: se un Pod è stato creato da un Deployment, è possibile semplicemente modificare le specifiche del Deployment e Kubernetes si occuperà di rimuovere e ricreare i Pod automaticamente.

## Replica Sets

Il Replication Controller controlla che il numero desiderato di Pod sia in esecuzione nel cluster. Si occupa di sostituire i Pod in errore e crearne di nuovi per mantenere lo stato del cluster desiderato.
Il Replication Controller è in realtà la vecchia implementazione del controller, mentre il nuovo si chiama Replica Set.

Per vedere i Replication Controller in esecuzione:

```shell
kubectl get replicationcontroller
```

mentre per i ReplicaSet:

```shell
kubectl get replicaset
```

### Deploy di un Replication Controller da specifica YAML

La configurazione di base per il deploy di un Replication Controller utilizzando uno *yaml* è:

```yaml
# rc-definition.yaml
apiVersion: v1
kind: ReplicationController
metadata:
    name: myapp-rc
    labels:
        app: myapp
        type: front-end
spec:
    replicas: 3
    template:
    ### POD DEFINITION ###
        metadata:
        name: myapp-pod
        labels:
            app: myapp
            type: front-end
        spec:
            containers:
            - name: nginx-container
                image: nginx
    ######################
```

Per eseguire il deploy:

```shell
kubectl apply -f rc-definition.yaml
```

### Deploy di un ReplicaSet da specifica YAML

La configurazione di base per il deploy di un ReplicaSet utilizzando uno *yaml* è:

```yaml
# replicaset-definition.yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
    name: myapp-replicaset
    labels:
        app: myapp
        type: front-end
spec:
    replicas: 3
    selector: 
        matchLabels:
            type: front-end
    template:
    ### POD DEFINITION ###
        metadata:
        name: myapp-pod
        labels:
            app: myapp
            type: front-end
        spec:
            containers:
            - name: nginx-container
              image: nginx
    ######################
```

La differenza principale tra ReplicaSet e Replication Controller è la presenza dei `selector`. Il tag `selector` permette di selezionare quali Pod il ReplicaSet deve gestire, oltre a quelli definiti nello *yaml*. È utile per controllare Pod che sono già in esecuzione nel cluster.
In realtà anche Replication Controller permette di inserire la tag `selector`, che però è facoltativa e assume di default i valori presenti sul tag `labels`. Inoltre nel Replication Controller non è possibile usare `matchExpression`.

Per eseguire il deploy:

```shell
kubectl apply -f replicaset-definition.yaml
```

### Scaling di un ReplicaSet

Ci sono due modi per aumentare (o diminuire) le repliche di un Pod. È possibile modificare il file *yaml* e deployarlo:

```shell
kubectl replace -f replicaset-definition.yaml
# oppure
kubectl apply -f replicaset-definition.yaml
```

In alternativa è possibile usare il comando `scale`:

```shell
kubectl scale --replicas=6 -f replicaset-definition.yaml
```

che andrà anche ad aggiornare automaticamente il file.
Se non si conosce la posizione del file *yaml*, è possibile scalare anche con:

```shell
kubectl scale --replicas=6 replicaset <nome_replicaset>
```

*Nota*: questo comando non aggiorna il file *yaml* ma esegue direttamente lo scaling in runtime.

## Deployments

Un Deployment permette di eseguire varie repliche di un Pod, come un ReplicaSet. Ha però anche molte funzionalità in più, come ad esempio la possibilità di aggiornare facilmente le immagini dei container facendo dei *rollout* e dei *rollback* in caso di errore. Il Deployment è l'oggetto principale per eseguire Pods di applicazioni __Stateless__.
Un Deployment crea automaticamente il suo ReplicaSet, e il ReplicaSet crea poi i Pod. Per vedere tutti gli oggetti creati da Kubernetes:

```shell
kubectl get all
```

mentre per vedere solo i deployment:

```shell
kubectl get deployments
```

### Deploy di un Deployment da specifica YAML

La configurazione di un Deployment è:

```yaml
# deployment-definition.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
    name: myapp-deployment
    labels:
        app: myapp
        type: front-end
spec:
    replicas: 3
    selector: 
        matchLabels:
            type: front-end
    template:
    ### POD DEFINITION ###
        metadata:
        name: myapp-pod
        labels:
            app: myapp
            type: front-end
        spec:
            containers:
            - name: nginx-container
              image: nginx
    ######################
```

Come si nota è molto simile al ReplicaSet.
Per eseguire il deploy:

```shell
kubectl apply -f deployment-definition.yaml
```

È possibile aggiornare l'immagine di un deployment in esecuzione direttamente da cli usando:

```shell
kubectl set image deployment <nome_deployment> nginx=nginx:1.18
```

### Generare un file YAML con Dry Run

Potrebbe essere complicato creare da zero un file *yaml* da terminale. È possibile però auto generare le specifiche *yaml* eseguendo una `dry-run` con `kubectl create`, ad esempio:

```shell
kubectl create deployment --image=nginx nginx --dry-run=client -o yaml > nginx-deployment.yaml
```

## Stateful Sets

Gli *Stateful Sets* sono simili ai *Deployments* ma vengono utilizzati quando lo stato del Pod è importante, e tutte le repliche devono avere lo stesso stato (ad esempio: nei database, tutte le repliche devono avere i dati nel database sincronizzati tra di loro).
Una strategia di base è quella di avere un Pod Master e diversi Pod Slave/Read Only. Il master si occupa di scrivere i dati nel database, mentre dalle repliche è possibile solo che leggere.

Una strategia per creare delle repliche di Pod stateful è quella di creare prima un Master, quando questo è pronto, creare il primo Slave copiando i dati dal Master. Successivamente, creare altri Slave, sempre attendendo che lo Slave precedente sia pronto, e copiando i dati da esso (non dal Master, per evitare di sovraccaricarlo). Questo procedimento __non__ può essere seguito con un Deployment. Stateful Set invece assicura che:

* I Pod vengano creati con ordine, attendendo che quello precedente sia Ready.
* I Pod abbiano dei nomi statici, cioè `<nome-pod>-<numero-incrementale>`, dove il numero va da 0 ad n. Il Pod con numero 0 sarà il Master.

### Deploy di uno StatefulSet da specifica YAML

La configurazione di uno StatefulSet è simile a quella di un Deployment, cambia solo il `kind` e la richiesta di un *Headless Service*:

```yaml
# deployment-definition.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
    name: myapp-deployment
    labels:
        app: myapp
        type: front-end
spec:
    replicas: 3
    selector: 
        matchLabels:
            type: front-end
    serviceName: mysql-h # servizio headless
    podManagementPolicy: OrderedReady # OrderedReady/Parallel
    template:
    ### POD DEFINITION ###
        metadata:
        name: myapp-pod
        labels:
            app: myapp
            type: front-end
        spec:
            containers:
            - name: nginx-container
              image: nginx
    ######################
```

`podManagementPolicy` indica che strategia di deployment usare per l'esecuzione dei Pod:

* `OrderedReady`: è l'opzione di default. I Pod vengono creati con ordine, uno per volta.
* `Parallel`: i Pod vengono creati in parallelo, ma avranno comunque sempre un nome statico (a differenza dei Pod dei Deployment, che hanno un nome casuale).

Il servizio headless permette di contattare i Pod in modo diretto senza conoscere il loro IP. In questo modo l'applicazione può connettersi al Master quando deve scrivere, e connettersi al normale servizio del database (quello che effettua il load balancing tra le repliche readonly) quando deve scaricare dei dati (vedi [Headless Service](#headless-service)).

### Volumi per Stateful Sets

Impostando un volume nel modo classico su un StatefulSet, risulta che ogni Pod utilizza la stessa PVC e quindi lo stesso PV. Non è in genere questa la soluzione che si vuole adottare, inoltre non tutti i volumi supportano la modalità `ReadWriteMany`.

```yaml
apiVersion: v1
kind: StatefulSet
# ...
template:
    metadata:
        labels:
            app: mysql
    spec:
    containers:
        - name: mysql
        image: mysql
        volumeMounts:
        - mountPath: "/var/lib/mysql"
            name: data-volume
    volumes:
        # la PVC è stata definita all'esterno, è statica e ogni Pod la utilizzerà per il volume dei dati
        - name: data-volume
        persistentVolumeClaim:
            claimName: data-volume
```

Si vuole invece avere una PVC e quindi un PV diverso associato ad ogni Pod dello StatefulSet. Per questo si può utilizzare i `volumeClaimTemplates`:

```yaml
apiVersion: v1
kind: StatefulSet
# ...
template:
    metadata:
        labels:
            app: mysql
    spec:
    containers:
        - name: mysql
        image: mysql
        volumeMounts:
        - mountPath: "/var/lib/mysql"
            name: data-volume
volumeClaimTemplates:
  - metadata:
      name: data-volume
    spec:
        accessModes:
            - ReadWriteOnce
        storageClassName: google-storage
        resources:
            requests:
                storage: 500Mi

```

In questo modo, quando i Pod vengono creati, viene creato in automatico anche la PVC e il PV associati ad esso. Se il Pod viene ricreato, il volume __non__ viene cancellato, ed anzi ad ogni Pod dello StatefulSet viene associato sempre lo stesso volume, mantenendo quindi i dati replicati al suo interno.

## Services

Permette di connettere i Pod tra di loro e con l'esterno.

Per vedere i servizi in esecuzione nel cluster:

```shell
kubectl get services
```

I servizi possono essere contattati sia per nome che per IP. Una volta definite le specifiche *yaml*, creare il servizio con:

```shell
kubectl apply -f service-definition.yaml
```

si può anche creare un servizio andando ad esporre un Deployment col comando:

```shell
kubectl expose deployment <nome_deployment> --port <porta> --type LoadBalancer --name <nome_servizio>
```

### NodePort

Mappa una porta del Nodo ad una porta del Pod. In questo modo le richieste inviate dall'esterno all'IP e porta del nodo vengonono indirizzate al servizio in esecuzione nel Pod. Il servizio viene lanciato all'interno del cluster come un server virtuale con un suo IP interno.

Le porte da configurare nel servizio sono tre:

* TargetPort: la porta del Pod a cui reindirizzare le richieste.
* Port: la porta del servizio da cui provengono le richieste inviate al Pod.
* NodePort: La porta del Nodo che ricevere le richieste dall'esterno. La porta del Nodo deve essere compresa tra 30000 e 32767.

![Porte di un Servizio](../assets/section-2/service_ports.PNG)

La configurazione di un Service NodePort è:

```yaml
# service-definition.yaml
apiVersion: v1
kind: Service
metadata:
    name: myapp-service
spec:
    type: NodePort
    ports:
      - targetPort: 80 # facoltativo, di default è uguale a port
        port: 80
        nodePort: 30008 # facoltativo, di defeault viene assegnata una porta casuale nel range 30000 e 32767
    selector:
        app: myapp
        type: front-end
```

I Pod a cui il servizio invia le richieste viene selezionato tramite i `selector`. Verranno selezionati solo i Pod che hanno __tutti__ i label corrispondenti (sia chiave che valore).

Ora è possibile ricevere una risposta dal servizio all'interno del nodo chiamando l'indirizzo pubblico del nodo alla porta 30008. Se sono presenti più Pod corrispondenti ai `labels` scelti, la richiesta verrà inviata a uno dei Pod del nodo in modo casuale.
Se i Pod corrispondenti sono presenti in nodi diversi, verrà creato automaticamente un servizio che copre tutti i nodi necessari, e i Pod diventano raggiungibili facendo una richiesta a qualsiasi degli IP esterni dei nodi selezionati, alla porta NodePort impostata.

![Servizio in diversi Nodi](../assets/section-2/multi_node_service.PNG)

### ClusterIP

Il servizio crea un indirizzo virtuale visibile solo all'interno del Cluster. Questo permette la connessione tra Pod all'interno del cluster, senza esporli all'esterno. I vari Pod inviano richieste al servizio tramite il suo nome, e il servizio si occupa di inoltrare la richiesta a uno dei Pod a cui la richiesta è indirizzata, casualmente.

La configurazione di base di un Service ClusterIP è:

```yaml
# service-definition.yaml
apiVersion: v1
kind: Service
metadata:
    name: back-end
spec:
    type: ClusterIP
    ports:
      - targetPort: 80 # porta alla quale il backend è esposto, cioè la porta dei Pod a cui vengono inviate le richieste.
        port: 80 # porta alla quale il servizio è esposto, cioè quella a cui va contattato il servizio.
    selector:
        app: myapp
        type: back-end
```

ClusterIP è il servizio di default quando `type` non è definito.

### LoadBalancer

Funziona come NodePort, ma distribuisce il traffico proveniente dall'esterno ai vari nodi quando il servizio copre più nodi. Il servizio LoadBalancer è utilizzabile solamente quando il cluster risiede in un Cloud Provider compatibile, in quando in servizio LoadBalancer si integra col cloud provider per configurare automaticamente un Load Balancer.
Se il cluster viene creato da zero su una macchina senza alcun supporto da un provider, LoadBalancer si comporterà esattamente come NodePort e configurare manualmente un Load Balancer (come *haproxy* o *nginx*) che distribuisca correttamente le richieste ai vari nodi.

La configurazione di un servizio LoadBalancer è come quella di NodePort:

```yaml
# service-definition.yaml
apiVersion: v1
kind: Service
metadata:
    name: myapp-service
spec:
    type: LoadBalancer
    ports:
      - targetPort: 80 # facoltativo, di default è uguale a port
        port: 80
        nodePort: 30008 # facoltativo, di defeault viene assegnata una porta casuale nel range 30000 e 32767
    selector:
        app: myapp
        type: front-end
```

### Headless Service

Un servizio headless non svolge alcun load balancing, ma semplicemente assegna un record DNS a tutti i Pod connessi al servizio. In questo modo, è possibile chiamare un Pod preciso in modo diretto, senza sapere il suo IP ne il record DNS del Pod (che è creato a partire dall'IP, ma con `-` al posto dei `.`, vedi [DNS in Kubernetes](NETWORKING.md#DNS-per-i-Pod)). La definizione di un servizio headless richiede di impostare `clusterIP: None`:

```yaml
# mysql-h.yaml
apiVersion: v1
kind: Service
metadata:
    name: mysql-h
spec:
    clusterIP: None # indica che il servizio è headless
    ports:
        port: 3306
    selector:
        app: mysql
```

Una volta creato il servizio headless, __in generale non vengono creati automaticamente i record DNS__ dei Pod. È necessario specificare nei Pod del due nuove proprietà, `subdomain` e `hostname`:

```yaml
apiVersion: apps/v1
kind: Deployment
# ...
template:
    metadata:
    name: myapp-pod
    labels:
        app: myapp
        type: front-end
    spec:
        subdomain: mysql-h
        hostname: mysql-pod
        containers:
        - name: nginx-container
            image: nginx
# ...
```

In questo caso, __tutte__ le repliche vengono create __con lo stesso hostname e subdomain__. Questo può andare bene nel caso di Deployment e ReplicaSet, ma __non nel caso dei StatefulSet__, dove i Pod devono avere ogniuno il suo record DNS.

Nel caso di *StatefulSet*, __non devono essere specificati `hostname` e `subdomain__`. Invece, va specificata la proprietà `serviceName`, che indica il servizio headless che gestirà i record DNS. In questo modo, i record DNS vengono creati automaticamente e con `hostname` diversi, con il formato:

```shell
<nome-pod-stateful-set>.<nome-servizio-headless>.<namespace>.<tipo-oggetto>.<cluster>
# esempio
mysql-0.mysql-h.default.svc.cluster.local
```

```yaml
# ...
spec:
    replicas: 3
    selector: 
        matchLabels:
            type: mysql
    serviceName: mysql-h # servizio headless
    template:
# ...
```

## Namespaces

I namespace permettono di suddividere gli oggetti di kubernetes in insiemi. All'interno del namespace gli oggetti vengono chiamati col loro nome, mentre per contattare dall'esterno un oggetto di un namespace si utilizza la notazione *name.namespace* (oppure la notazione estesa *name.namespace.object_type.cluster_domain*, ad esempio: `db-service.dev.svc.cluster.local`).
Ogni namespace può limitare le risorse utilizzate dagli oggetti del kluster e può assegnare diversi permessi.

Di default, Kubernetes assegna tutti gli oggetti al namespace *default*. Inoltre, allo startup, crea altri due namespace: *kube-system* (che contiene i servizi interni) e *kube-public* (dedicato agli oggetti da esporre a tutti gli utenti).

Tutti i comandi di `kubectl` mostrano gli oggetti del *default* namespace. Per specificare un namespace diverso, utilizzare l'opzione `--namespace` o `-n`:

```shell
kubectl get pods --namespace <nome_namespace>
```

Per vedere invece gli oggetti di  tutti i namespace:

```shell
kubectl get pods --all-namespaces
```

### Deploy di un Namespace da CLI

Per creare un namespace direttamente da cli:

```shell
kubectl create namespace <nome_namespace>
```

### Deploy di un Namespace da specifica YAML

Per creare un namespace da una specifica *yaml*:

```yaml
# namespace-dev.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
```

e deployarlo nel cluster con:

```shell
kubectl create -f namespace-dev.yaml
```

### Creare oggetti in un Namespace specifico

Per creare un oggetto in un preciso namespace, utilizzare:

```shell
kubectl create -f spec.yaml --namespace <nome_namespace>
```

Se si vuole invece specificare nel file *yaml* a quale namespace appartiene l'oggetto, aggiungere il namespace tra i `metadata`:

```yaml
# pod-definition.yaml
apiVersion: v1
kind: Pod
metadata:
    name: myapp-pod
    namespace: dev
    labels:
        app: myapp
        type: front-end
spec:
    containers:
      - name: nginx-container
        image: nginx
```

*Nota*: non tutti gli oggetti di Kubernetes possono essere assegnati ad un namespace. Alcuni oggetti sono *Cluster Scoped*, come i Nodi, Persistent Volumes, Cluster Roles e i Namespace stessi.

### Cambiare il Namespace di default

Se si vuole cambiare il namespace che viene utilizzato di default da Kubernetes:

```shell
kubectl config set-context $(kubectl config current-context) --namespace=<nome_namespace>
```

in questo modo tutti i comandi inviati saranno riferiti al namespace selezionato. Il context si riferisce al cluster a cui siamo collegati.

### Limitare le risorse di un Namespace con ResourceQuota

Per limitare le risorse di un namespace, utilizzare l'oggetto ResourceQuota:

```yaml
# resource-quota-dev.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
    name: compute-quota
    namespace: dev
spec:
    hard:
        pods: "10" # numero massimo di pod in esecuzione nel namespace
        requests.cpu: "4" # massime richieste di cpu tra tutti i pod nel namespace
        requests.memory: 5Gi # massima richiesta di memoria tra tutti i pod del namespace 
        limits.cpu: "10" # cpu massime utilizzate tra tutti i pod del namespace
        limits.memory: 10Gi  # memoria massima utilizzata tra tutti i pod del namespace
```

e deployarlo con:

```shell
kubectl apply -f compute-quota.yaml
```

`requests` dovrebbe essere impostato sulle risorse minime che il Pod necessita per funzionare correttamente. Se queste risorse vengono impostate troppo alte, è possibile che non si riesca a trovare un nodo su cui deployare il Pod che garantisca le risorse richieste. `limits` rappresenta le risorse massime che possono essere utilizzate. `limits` deve sempre essere maggiore di `requests`.

## Imperative vs Declarative

Sono i due diversi approcci per gestire il cluster e inviare comandi.

### Imperative

Utilizzando il metodo imperativo, vengono usati comandi ben specifici per creare, aggiornare e distruggere gli oggetti:

```shell
# CREATE
kubectl run # crea ed esegue un pod
kubectl create # crea un oggetto tramite opzioni o leggendo un file yaml
kubectl expose # crea un servizio per esporre un oggetto
# UPDATE
kubectl edit # per modificare le specifiche di un oggetto in esecuzione. Le modifiche NON vengono riportate nel file yaml originale salvato localmente
kubectl replace # rimpiazza la configurazione di un oggetto in esecuzione con una nuova definizione yaml. In questo caso le modifiche sono sincronizzate tra file locale e memoria di kubernetes
kubectl scale # modifica il numero di repliche di un oggetto
kubectl set
```

Con questo metodo, i comandi sono digitati nella console e solo l'operatore che effettua l'operazione conosce la sequenza di comandi eseguita o il modo in cui certi oggetti in esecuzione sono stati creati.
Inoltre, creare una configurazione complessa con comandi imperativi richiede di scrivere comandi molto lunghi e complessi.

Per facilitre le operazioni, è possibile utilizzare i file di configurazione *yaml* e creare, aggiornare e distruggere gli oggetti indicati nei file.

### Declarative

I comandi esposti precendentemente possono fallire se le condizioni non sono corrette (es: cerco di creare un oggetto che esiste già).
Il comando `kubectl apply` è un comando intelligente che in funzione dello stato del cluster capisce se deve creare, aggiornare o rimuovere gli oggetti:

```shell
kubectl apply -f <file_name>
```

Al posto di un singolo file è anche possibile puntare ad un intera cartella per eseguire tutti i file *yaml* presenti in essa.

Per applicare correttamente i comandi necessari, Kubernetes confronta la nuova configurazione con quella in esecuzione, ed inoltre con un JSON rappresentante l'ultima configurazione applicata (utilizzata sopratutto per capire se qualche campo è stato cancellato dal file locale).
L'ultima configurazione applicata è salvata direttamente sulla definizione dell'oggetto nella memoria del cluster, su un `annotations`, e viene creata solo utilizzando il comando `kubectl apply`.

## Risorse Personalizzate

Tutte gli oggetti visti fin'ora sono inclusi nell'installazione di Kubernetes. È possibile però creare risorse/oggetti personalizzati, tramite un *Custom Resource Definition*:

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
    name: flighttickets.flights.com
spec:
    scope: Namespaced # indica se l'oggetto creato sarà namespaced o no
    group: flights.com
    names:
        kind: FlightTicket
        singular: flightticket
        plural: flighttickets
        shortnames:
          - ft
    versions:
      - name: v1
        served: true
        storage: true
    schema:
        openAPIV3Schema:
            # Open API 3 schema per tutte le proprietà che si possono inserire nell'oggetto
```

Una volta definita e creata la risorsa con `kubectl create -f <crd-file>`, sarà possibile utilizzarla nel cluster:

```yaml
apiVersion: flights.com/v1
kind: FlightTicket
# ...
```

L'oggetto creato verrà inserito nell'`etcd`, ma non farà nulla. Questo perchè manca ancora la definizione del Controller, che riceve monitora la risorsa ed effettua le operazioni e le chiamate API necessarie a svolgere il compito dell'oggetto (es: in questo caso, a prenotare effettivamente il biglietto aereo).
Un controller è un codice che controlla continuamente lo stato del cluster ed in particolare certi tipi di risorse (`kind`). Quando il `kube-apiserver` richiede l'esecuzione di un certo comando per quella risorsa, è il Controller che dovrà rispondere effettuando le operazioni necessarie (es: `create` prenota il biglietto, `delete` annulla la prenotazione).
I controller possono essere scritti in qualsiasi linguaggio di programmazione, ma Kubernetes offre [un template scritto in Go](https://github.com/kubernetes/sample-controller) per facilitare la creazione dei Controller.
Il Controller può essere eseguito all'interno di un Pod nel cluster.

### Operator Framework

Gli operatori sono dei plugin di Kubernetes per gestire le risorse personalizzate e i suoi componenti. Il framework permette di automatizzare facilmente molte funzioni estendendo le API di Kubernetes.

## References

1. [CKA Course - Core Concepts](https://github.com/kodekloudhub/certified-kubernetes-administrator-course/tree/master/docs/02-Core-Concepts)
2. [Kubernetes Documentation - Getting Started](https://kubernetes.io/docs/setup/)
3. [Kubernetes Documentation - Kubeadm Install](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
4. [kubectl Usage Conventions](https://kubernetes.io/docs/reference/kubectl/conventions/)
5. [etcd.io](https://etcd.io/docs/)
