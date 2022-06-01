# DESIGN AND INSTALL A KUBERNETES CLUSTER

## Design del Cluster

Un cluster Kubernetes può essere

* On Premise: il cluster viene gestito interamente dall'organizzazione su server privati. È necessario configurare il cluster totalmente e mantenerlo in autonomia.
* Cloud: Vengono sfruttati i servizi cloud di vari provider (es: AWS, Google, Azure).
* Singolo Nodo: utilizzato generalmente a scopo educativo (es: minikube).
* Macchine fisiche: ogni nodo è una vera e propria macchina fisica.
* Macchine virtuali: ogni nodo è in una macchina virtuale, in un server che ne ospita anche altre.

Oltre ai nodi e quindi alla scelta del tipo di macchina da utilizzare, è necessario anche scegliere il tipo di storage:

* *SSD*: per applicazione ad alta performance.
* *Network Based Storage*: per applicazioni che richiedono molte connessioni in contemporanea.
* *PV*: per volumi persistenti che devono essere disponibili per molti Pod.

## Configurazione HA

In una configurazione HA (*High Availability*) il cluster ha diverse copie del nodo master. Nel master sono in esecuzione diversi processi, che vanno configurati in modo differente in questo tipo di setup:

* *API Server*: può funzionare in parallelo, ma deve esserci un Load Balancer davanti in modo da gestire le richieste correttamente. Una stessa richiesta non può essere fatta ad entrambi gli API Server.
* *Scheduler e Controller Manager*: Solo uno di questi può essere attivo, mentre gli altri rimangono in standby, pronti nel caso di fallimento. I processi attivi vengono scelti tramite la *lead election*.
Il primo processo che controlla il *Kube Controller Manager Endpoint* diventa leader. Vi sono diverse impostazioni per eleggere il leader:

```shell
--leader-elect true # è a true di default
--leader-elect-lease-duration 15s # quando dura il lease
--leader-elect-renew-deadline 10s # quanto tempo si ha per rinnovare il lease
--leader-elect-retry-period 2s # ogni quanto i processi provano a diventare leader
```

* *ETCD*: diverse istanze dell'ETCD possono essere in esecuzione allo stesso momento. È necessario indicare tutte le istanze presenti all'API Server tramite l'opzione `--etcd-servers`. L'ETCD può essere installato nello stesso nodo in cui sono presenti gli altri componenti (*stacked*) oppure in nodi separati.

## Configurazione HA per ETCD

L'ETCD è un semplice database chiave e valore, ma è distribuito. Questo significa che vi sono diverse istanze in esecuzione allo stesso momento, in cui si può leggere e scrivere. La lettura del database non è un problema in un sistema distribuito, ma la scrittura si, in quanto i dati devono essere sincronizzati tra tutte le istanze.
Viene quindi eletto un leader tra tutte le istanze. Le istanze subordinate che ricevono una richiesta di scrittura inoltrano la richiesta al leader. È il leader che scrive nel suo database e che invia la richiesta di scrittura anche a tutte le altre istanze per mantenerle sincronizzate. Il processo di scrittura è completo solo quando tutte le istanze danno conferma dell'avvenuta scrittura.
Se il leader crasha o si disconnette, viene eletto un nuovo leader. È possibile però che anche istanze subordinate vengano perse. Questo significa che il processo di scrittura non sarebbe mai concluso. In realtà, il processo di scrittura si assume completo se il *quorum* è stato raggiunto, cioè se almeno *N/2+1* istanze dell'ETCD hanno scritto il nuovo dato.
Nel caso di segmentazione del network o simili problemi di rete, potrebbe essere che le istanze vengano divise in due gruppi separati. Solo il gruppo che ha abbastanza istanze da raggiungere il *quorum* potrebbe continuare a lavorare. Per questo motivo è sempre consigliato __utilizzare un numero dispari__ di nodi, in modo che, nel caso avvenga una segmentazione, uno dei due gruppi possa continuare a lavorare.
Il numero minimo di nodi per avere effettivamente un sistema HA è 3, ma 5 nodi permettono di avere una maggiore tolleranza al fallimento. In genere, avere più di 5 nodi non porta a reali benefici.

1. [CKA Course - Desing and Install a Kubernetes Cluster](https://github.com/kodekloudhub/certified-kubernetes-administrator-course/tree/master/docs/10-Design-and-Install-Kubernetes-Cluster)
2. [Install Kubernetes the Hard Way - Video Tutorial](https://www.youtube.com/watch?v=uUupRagM7m0&list=PL2We04F3Y_41jYdadX55fdJplDvgNGENo)
3. [Install Kubernetes the Hard Way - Docs](https://github.com/mmumshad/kubernetes-the-hard-way)
