# SCHEDULING

Lo Scheduler decide a che Nodo assegnare i Pod, in funzione delle risorse richieste, risorse disponibili ecc. con l'aiuto di una funzione che permette di scegliere il Nodo compatibile migliore.

## Scheduling Manuale

Nella definizione *yaml* di un Pod è presente il campo `nodeName`. Questo non viene inserito dall'utente ma da Kubernetes, è indica il nodo in cui è deployato il Pod. Lo Scheduler cerca i Pod che non hanno questa proprietà impostata e li assegna ad un nodo.
Uno Pod che non ha un nodo assegnato rimane nello stato *Pending*. È possibile assegnare manualmente il Pod ad un nodo semplicemente inserendo `nodeName` manualmente nella specifica *yaml*. Questa operazione può essere esegiota solo durante la __creazione__ del Pod.

```yaml
# pod-definition.yaml
apiVersion: v1
kind: Pod
metadata:
    name: nginx
    labels:
        name: nginx
spec:
    containers:
      - name: nginx
        image: nginx
    ports:
      - containerPort: 8080
    nodeName: node02
```

Se il Pod è già stsato creato, per assegnarlo ad un nodo è necessario utilizzare un Binding per richiedere al cluster di assegnare il Pod ad un certo nodo:

```yaml
apiVersion: v1
kind: Binding
metadata:
  name: nginx
target:
  apiVersion: v1
  kind: Node
  name: node02
```

La definizione *yaml* va trasformata in JSON e inviata tramite curl all'API di Kubernetes:

```shell
curl --header "Content-Type: application/json" --request POST --data '{"apiVersion":"v1", "kind":"Binding", ...}' http://$SERVER/api/v1/namespaces/default/pods/$PODNAME/binding
```

## Label, Selectors e Annotations

È possibile assegnare qualsiasi label agli oggetti di Kubernetes, in modo da porteli categorizzare, filtrare e selezionare tramite i selector successivamente. I labels vanno inseriti nel tag `metadata` dell'oggetto.

Per selezionare degli oggetti in funzione dei `labels`:

```shell
kubectl get pods --selector app=my-app
```

Per contare velocemente tutti gli oggetti con un certo label:

```shell
kubectl get pods -l app=my-app --no-headers | wc -l
```

Un Service o ReplicaSet utilizza i campi `selector` e `matchLabels` per selezionare i Pod che deve gestire. Il Service o ReplicaSet prenderà in carico i Pod che contengono __tutti__ i `labels` presenti in `matchLabels`. I Pod, viceversa, possono avere anche più `labels` diversi, ma per essere presi in carico devono avere almeno i `labels` che il servizio sta cercando.

Le `annotations` sono altri metadati che vengono usati solo a scopo informativo per inserire alcune informazioni nella definizione dell'oggetto. Molte volte vengono create da Kubernetes durante l'esecuzione.

## Taint e Tolerations

Sono delle regole che permettono di limitare il deploy di Pod in alcuni nodi. Ad un certo nodo viene applicata una `taint` (un "repellente"). Di default tutti i Pod sono incapacitati dal repellente, per cui nessun Pod verrà deployato nel nodo dallo Scheduler. Alcuni Pod possono però essere abilitati ad essere deployati sul nodo aggiungendo ai Pod voluti una `toleration`, cioè il Pod diventa tollerante al repellente.

*Nota*: questo sistema __NON__ assicura che un certo Pod verrà deployato su un nodo a cui il Pod è tollerante. Il repellente assicura però che il nodo non avrà Pod in esecuzione che non sono tolleranti al suo repellente. Se ci si vuole assicurare che un Pod venga eseguito in un certo nodo, bisogna invece utilizzare la funzionalità di *Affinity*.

Anche il *Master Node* del cluster potrebbe ricevere dei Pod, ma è tainted da Kubernetes quando il cluster viene creato. È possibile applicare un repellente ai Pod in modo che questi vengano deployati anche nel nodo master, ma è buona norma non farlo. Per conoscere il Taint del master:

```shell
kubectl describe node kubemaster | grep Taint
```

### Taints

Per applicare un `taint` ad un nodo:

```shell
kubectl taint nodes <nome_nodo> key=value:taint-effect
```

Per rimuovere un `taint` aggiungere un `-` alla fine:

```shell
kubectl taint nodes <nome_nodo> key=value:taint-effect-
```

I `taint-effect` hanno dei valori ben definiti:

* `NoSchedule`: il Pod non verrà deployato nel nodo.
* `PreferNoSchedule`: lo Scheduler cercherà di non deployare il Pod sul nodo.
* `NoExecute`: il Pod non verrà deployato nel nodo, ma anche i Pod già presenti verranno fermati e rimossi dal nodo.

un esempio può essere:

```shell
kubectl taint nodes node1 app=blue:NoSchedule
```

### Tolerations

Le `tolerations` vengono applicate nelle specifiche di un Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
    name: myapp-pod
spec:
    containers:
      - name: nginx-container
        image: nginx
    tolerations:
      - key: "app"
        operator: "Equal"
        value: "blue"
        effect: "NoSchedule"
```

*Nota*: tutti i valori definiti per la `toleration` devono essere stringhe tra doppi apici.

I valori possibili per il campo `operator` sono:

* Equal: se la `key` ha un valore `value`.
* Exists: se la `key` esiste. Non deve essere specificato alcun `value`.

## Node Selectors

Pod che hanno un grosso carico computazionale potrebbero essere assegnati a nodi con poche risorse, e viceversa. Con il NodeSelector è possibile schedulare un Pod su un ben preciso nodo in funzione dei suoi label:

```yaml
# pod-specification.yaml
apiVersion: v1
kind: Pod
metadata:
    name: myapp-pod
spec:
    containers:
      - name: data-processor
        image: data-processor
    nodeSelector:
        size: Large # deve essere un label assegnato al nodo
```

Per applicare un `label` ad un nodo:

```shell
kubectl label nodes <node-name> <label-key>=<label-value>
```

NodeSelector è molto limitato in quanto può selezionare solo nodi con un certo label, non può selezionare combinazioni di labels o escluderne alcune. Per questo è necessario usare Node Affinity.

## Node Affinity

Node Affinity permette di creare regole di selezione molto più complesse per selezionare i nodi in cui schedulare i Pod.

Per schedulare un Pod in un nodo che combacia con più valori possibili del `label`, ad esempio:

```yaml
# pod-definition.yaml
apiVersion: v1
kind: Pod
metadata:
 name: myapp-pod
spec:
 containers:
 - name: data-processor
   image: data-processor
 affinity:
   nodeAffinity:
     requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: size
            operator: In
            values: # il Pod viene deployato in un nodo con uno dei due valori
            - Large
            - Medium
```

I valori possibili per gli operatori sono:

* *In*
* *NotIn*
* *Exists*
* *DoesNotExist*

I tipi di Affinity sono:

* `requiredDuringSchedulingIgnoredDuringExecution`: il Pod deve essere schedulato su un nodo che rispetta le regole di affinità.
* `preferredDuringSchedulingIgnoredDuringExecution`: lo scheduler cercherà di rispettare le regole di affinità ma non è garantito.

dove:

* *DuringScheduling*: si intende quando il Pod deve essere creato e schedulato in un nodo.
* *DuringExecution*: quando un Pod è già in esecuzione in un nodo.
* *required*: se non c'è un nodo disponibile, il Pod rimane nello stato Pending. Viene utilizzato quando la selezione del nodo è cruciale.
* *preferred*: se non c'è un nodo disponibile, viene schedulato su un altro nodo a caso.

Al momento tutte le regole di affinità ignorano i Pod in esecuzione (*IgnoredDuringExecution*). Questo significa che se viene rimosso un label dal nodo, un Pod che aveva affinità col nodo ma non ce l'ha più continua comunque a funzionare.

### Combinare Taints e Affinity

Una combinazione corretta di Taints e Affinity permette di schedulare un Pod esattamente nel nodo che ci interessa, evitando al contempo che altri Pod interferiscano.

1. Applicare al nodo una `taint` corrispondente alla `toleration` del Pod. Questo assicura che altri Pod non possano essere deployati su quel nodo. Ma il Pod interessato potrebbe essere schedulato in un altro nodo senza alcun `taint`.
2. Applicare un `affinity` al Pod corrispondente ai `labels` del nodo. In questo modo lo scheduler dovrà deployare il Pod nel nodo che ha affinità con esso.
3. Con questa combinazione, il Pod verrà sempre deployato nel nodo voluto, ed altri Pod estranei rimarranno sicuramente esclusi dal nodo.

## Requisiti e Limiti Risorse

Le risorse di un nodo sono CPU, Memoria e Disco. Lo Scheduler si occupa di deployare i Pod in nodi che abbiano abbastanza risorse disponibili per eseguirli. Se nessun nodo è libero, il Pod rimarra in Pending. Controllando gli eventi di un Pod è possibile verificare con precisione quale risorsa non è sufficiente. Per fare ciò:

```shell
kubectl describe pod <nome_pod> | grep "Events"
```

### Requisiti Risorse

Di default, Kubernetes non assegna alcun requisito di risorse di default. Per specificare dei requisiti di risorse in un Pod, si utilizza il campo `resources`:

```yaml
# pod-definition.yaml
apiVersion: v1
kind: Pod
metadata:
# ...
spec:
  containers:
    - name: my-app
      # ...
  resources:
    requests:
      memory: "1Gi"
      cpu: 1
```

* *CPU*: 1 CPU corrisponde ad un *Hyperthread*, oppure una *vCPU* (simile ad un core). La definizione varia in base al cloud provider utilizzato. Il valore minimo applicabile è `1m` (`100m` corrispondono a 0.1 CPU).
* *Memory*: si specifica in *bytes* e suoi multipli. I multipli nel sistema internazionale (*Kilobyte*, *Megabyte*, *Gigabyte*) sono abbreviati con `K`, `M`, `G`, mentre i multipli in potenza di due (*Kibibyte*, *Mebibyte*, *Gibibyte* ) sono abbreviati con `Ki`, `Mi`, `Gi`.

### Limiti Risorse

Il consumo di risorse può aumentare o diminuire durante l'esecuzione del Pod. Se il consumo aumenta troppo, potrebbe soffocare gli altri Pod ed anche i processi interni al nodo. Per questo motivo è necessario limitare le risorse utilizzabili da un Pod. Di default Kubernetes non applica alcun limite alle risorse. È possibile e consigliato quindi specificare il limite massimo di risorse utilizzabili da un Pod nel suo *yaml*:

```yaml
# pod-definition.yaml
apiVersion: v1
kind: Pod
metadata:
# ...
spec:
  containers:
    - name: my-app
      # ...
  resources:
    limits:
      memory: "2Gi"
      cpu: 2
```

Se un Pod cerca di utilizzare più CPU del previsto, la CPU va in throttling ma il Pod rimane in esecuzione. Se invece il Pod utilizza più memoria del previsto per un certo lasso di tempo, verrà terminato.

I limiti sono assegnati al Pod, per cui nel caso di Pod con molteplici container al suo interno, questi container dovranno condividere le risorse e i suoi limiti.

### LimitRange

È possibile impostare un valore di default per `limits` e `requests` utilizzando LimitRange.

```yaml
# mem-limit-range.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: mem-limit-range
spec:
  limits:
  - default:
    memory: 512Mi
    defaultRequest:
      memory: 256Mi
    type: Container
```

```yaml
# cpu-limit-range.yaml
apiVersion: v1
kind: LimitRange
metadata:
    name: cpu-limit-range
spec:
    limits:
      - default:
        cpu: 512Mi
        defaultRequest:
            cpu: 256Mi
        type: Container
```

## DaemonSets

Un DeamonSets funziona come un ReplicaSet, ma replica il Pod in ogni singolo nodo presente nel cluster, anche quando altri nodi vengono aggiunti successivamente.
È utile per avere in ogni singolo nodo dei Pod di utility come un monitor o un logger o log viewer. Anche il `kube-proxy` è un DeamonSets, perchè è necessario in ogni singolo nodo.

Le specifiche *yaml* sono praticamente uguali ad un ReplicaSet:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: monitoring-daemon
  labels:
    app: nginx
spec:
  selector:
    matchLabels:
      app: monitoring-agent
  template:
    metadata:
     labels:
       app: monitoring-agent
    spec:
      containers:
      - name: monitoring-agent
        image: monitoring-agent
```

Non è possibile creare un Daemon Set con `kubectl create`, ma è possibile comunque sfruttare i comandi da terminale con un piccolo trucchetto:

1. Creare un Deployment (`kubectl create deployment <nome_deployment> --dry-run=client -o yaml > ds.yaml`)
2. Rimuovere i campi `replicas`, `status`, `strategy` e sostituire a `kind: DaemonSet`.
3. Eseguire con `kubectl apply -f ds.yaml`

## Pod Statici

Ogni singolo nodo esegue al suo interno `kubelet`. Il servizio non può operare tutte le funzionalità di un cluster, la sua funzione è di creare e rimuovere i Pod.

Un singolo nodo può pertanto gestire Pod anche senza essere connesso ad un cluster. Questi Pod, gestiti autonomamente dal nodo, vengono detti Pod statici.

Il nodo controlla periodicamente la cartella `etc/kubernetes/manifests`, ed esegue tutte le specifiche *yaml* presenti al suo interno. Se un file viene cancellato, anche il Pod viene rimosso. Se un file viene modificato, il Pod viene ricreato.

*Nota*: `kubelet` può gestire solo e unicamente i Pod, per cui tutti i manifesti *yaml* devono essere del `kind` Pod.

Il nodo può gestire contemporaneamente Pod statici e Pod del cluster. I Pod statici saranno visibili anche dal `kube-apiserver` e quindi visibili utilizzando `kubectl get pods`, ma saranno in sola lettura e non potranno essere modificati.

I Pod Statici sono utili per creare nuovi nodi Master, ed è la stessa strategia che usa `kubeadm`. Per creare un nuovo master, basta installare `kubelet` nella nuova istanza e inserire nella cartella dei manifest le specifiche *yaml* di tutti i componenti principali di Kubernetes, come `etcd`, `kube-apiserver`, `controller-manager`. Questi verranno istanziati come Pod nel `kube-system`.

### Cambiare la cartella dei Manifest

All'esecuzione di `kubectl` è possibile passare un parametro per modificare la cartella di default contentente i file *yaml*:

```shell
# kubelet.service
/usr/local/bin/kubelet --pod-manifest-path=/etc/kubernetes/manifests
```

I file del servizio sono posizionati in `/lib/systemd/system/`.
In alternativa, è possibile indicare un file di configurazione esterno:

```shell
# kubelet.service
/usr/local/bin/kubelet --config=kubeconfig.yaml
```

```yaml
# kubeconfig.yaml
staticPodPath: /etc/kubernetes/manifests
```

Questo approccio è quello usato da `kubeadm` durante il setup del cluster. Per trovare la posizione del file su un nodo in cui `kubelet` è già in esecuzione:

```shell
ps aux | grep kubelet
```

Nell'output che comprare, trovare l'opzione `config=/path/to/config.yaml` che indicherà la posizione del file di configurazione.

## Scheduler Multipli

È possibile creare e deployare degli Scheduler personalizzati. Diversi Scheduler possono coesistere contemporaneamente nel cluster, ed è possibile impostare per ogni Pod a quale Scheduler deve essere affidato.

Per vedere gli Scheduler attivi:

```shell
kubectl get pods --namespace=kube-system
```

### Deploy di uno Scheduler

Il deploy di un nuovo Scheduler viene effettuato come con `kube-scheduler`.

```shell
ExecStart=/usr/local/bin/kube-scheduler
    --config=/etc/kubernetes/config/kube-scheduler.yaml
    --scheduler-name=my-custom-scheduler # nome dello scheduler
```

Per personalizzare il nuovo Scheduler, è possibile modificare il suo file *yaml* e deployarlo:

```yaml
apiVersion: vl
kind: Pod
metadata:
    name: my-custom-scheduler
    namespace: kube-system
spec:
    containers:
    command:
    - kube-scheduler
    - --address=127.0.0.1
    - --kubeconfiq=/etc/kubernetes/scheduler.conf
    - --leader-elect=true
    - --scheduler-name=my-custom-scheduler
    - --lock-object-name=my-custom-scheduler
    image: k8s.gcr.io/kube-scheduler-amd64:v1.11
    name: kube-scheduler
```

Il file esegue i comandi definiti nella proprietà `command`, in particolare:

* `leader-elect`: necessario quando copie multiple dello scheduler sono in esecuzione in diversi nodi master. Solo una delle copie può funzionare alla volta. Se ci fossero problemi al Pod leader, un altra copia prenderebbe il suo posto. Questa opzione è necessaria quando si hanno più nodi master, mentre può rimanere a false quando il master è solo uno.
* `lock-object-name`: necessario quando `leader-elect` è `true` e sono presenti più nodi master. Permette di indicare durante l'elezione del leader che il nuovo Scheduler è diverso da quello di default e da eventuali altri Scheduler.
* `scheduler-name`: nome del nuovo scheduler

### Utilizzo del nuovo Scheduler

Per indicare ad un Pod di utilizzare il nuovo scheduler:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - image: nginx
    name: nginx
  schedulerName: my-custom-scheduler
```

*Nota*: Se lo Scheduler non è configurato correttamente, il Pod rimarrà nello status di Pending.

### Eventi e Logs

Per verificare il corretto funzionamento dello Scheduler è possibile visionare gli Eventi del cluster e i logs dello Scheduler:

Per mostrare tutti gli eventi avvenuti nel Namespace corrente:

```shell
kubectl get events
```

Per vedere i logs dello Scheduler:

```shell
kubectl logs my-custom-scheduler -n kube-system
```

## References

1. [CKA Course - Scheduling](https://github.com/kodekloudhub/certified-kubernetes-administrator-course/tree/master/docs/03-Scheduling)
2. [Scheduler Code Hierarchy Overview](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-scheduling/scheduling_code_hierarchy_overview.md)