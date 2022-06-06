# SECURITY

## Principi di Sicurezza

### Host

I server in cui viene hostato il cluster dovrebbero seguire le regole basilari di sicurezza:

* Disabilitare password di root
* Nessun accesso remoto tramite password
* Accesso solo tramite SSH

### Kubernetes

Quando si accedere al cluster, si accede al `kubeapi-server`. È possibile accedere con diversi meccanismi di autenticazione:

* Utente e Password
* Utente e Token
* Certificati
* Provider esterni (es: LDAP)
* Account di servizio (per le macchine)

Una volta autenticato, l'utente sarà autorizzato a effettuare solo certe operazioni. Le autorizzazioni possono essere fornite in vari modi:

* Ruolo e Gruppo
* Attributi
* Nodo
* Webhook

Tutte i componenti del cluster comunicano con il `kubeapi-server` tramite certificati TLS.
I Pod del cluster possono, di default, comunicare tra di loro in tutto il sistema. Questa comunicazione può essere limitata tramite policy di rete.

## Autenticazione

Vi sono varie tipologie di utenti che si connetto al cluster:

* Amministratori: gestiscono e mantengono il cluster.
* Sviluppatori: sviluppatori che devono programmare e deployare i servizi.
* Utenti finali: l'autenticazione viene gestita dall'applicazione effettivamente in esecuzione nel cluster. Non dipende dalla sicurezza di Kubernetes.
* Altri servizi esterni: servizi e bot che devono accedere al sistema per alcune integrazioni.

Kubernetes __non__ gestisce le utente, ma solo i *Service Account* (usati dai servizi che si devono integrare nel cluster).

### File Statico con Password

È possibile utilizzare un file `.csv` contenente la lista degli utenti. Il file deve essere organizzato in questo modo:

```csv
password,username,userid,groupid
```

*Nota*: il gruppo è opzionale.

Il file viene letto da `kubeapi-server` per gestire l'autenticazione.
Per indicare a `kubeapi-server` quale file utilizzare per l'autenticazione, usare l'opzione:

```shell
# kube-apiserver.service
--basic-auth-file=user-details.csv
```

È necessario riavviare il servizio perchè le modifiche abbiano effetto. Se si utilizza `kubeadm`, è possibile inserire il comando sul manifest in `/etc/kubernetes/manifests/kube-apiserver.yaml` e il rispettivo Pod verrà ricreato automaticamente.

Per autenticarsi durante le chiamate:

```shell
curl -v -k https://master-node-ip:6443/api/v1/pods -u "user:password"
```

### File Statico con Token

Al posto di avere un file di password, è possibile usare un file di token:

```csv
token,username,userid,groupid
```

che va indicato al `kubeapi-server` con l'opzione `--token-auth-file`.

L'autenticazione viene effettuata con un Bearer token nell'header della chiamata:

```shell
curl -v -k https://master-node-ip:6443/api/v1/pods -H "Authorization: Bearer <token>"
```

### Autenticazione di Base con kubeadm

Se si utilizza `kubeadm`, è necessario fornire i file di utenti (o token) tramite un volume. Modificare il manifest `kube-apiserver.yaml` di conseguenza:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-apiserver
    - --authorization-mode=Node,RBAC
    # ....
    # indico la path in cui è presente  il file
    - --basic-auth-file=/tmp/users/user-details.csv
    image: k8s.gcr.io/kube-apiserver-amd64:v1.11.3
    name: kube-apiserver
    # monto il volume dal file system
    volumeMounts:
    - mountPath: /tmp/users
      name: usr-details
      readOnly: true
  volumes:
  # indico la cartella nel file system in cui sono salvati i file
  - hostPath:
      path: /tmp/users
      type: DirectoryOrCreate
    name: usr-details
```

Devono poi essere definiti i ruoli:

```yaml
kind: Role
# role base access control
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""] # "" incica il gruppo core API
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
---
# user1 deve poter vedere i pod in default
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: user1 # case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role #Role  ClusterRole
  name: pod-reader # creato precedentemente
  apiGroup: rbac.authorization.k8s.io
```

A questo punto l'utente può accedere al cluster tramite autenticazione.

*Nota*: questi tipi di autenticazioni di base sono state deprecate dalla versione 1.19 ed eliminate in release successive.

*Nota*: nessun tipo di autenticazione di base è adatta ad un ambiente di produzione.

## Certificati TLS

Nel cluster vi sono varie comunicazioni che vanno protette:

* Tra *Master Node* e *Worker Nodes*.
* Tra gli amministratori del cluster e il *Master Node*
* Tra tutti i componenti del cluster.

Tutte queste connessioni vengono protette tramite TLS.

### Componenti di Kubernetes

I componenti "server" che richiedono un certificato TLS sono:

* `kubeapi-server`: tutte le chiamate passano per il `kubeapi-server`, che è quindi il componente server più importante.
* `etcd`: riceve chiamate dal `kube-apiserver`. Il `kubeapi-server` è quindi anche un client visto sotto il punto di vista di `etcd`.
* `kubelet`: riceve istruzioni dal `kube-apiserver`. Il `kubeapi-server` è quindi anche un client per lui.

*Nota*: il `kubeapi-server` può utilizzare gli stessi certificati generati come server come client, quando deve chiamare `etcd` e `kubelet`. In alternativa, è possibile generare dei certificati appositi per queste connessioni.

mentre i componenti "client" che richiedono il TLS sono:

* `kubectl`: l'amministratore del cluster utilizza `kubectl` per inviare comandi al `kubeapi-server`.
* `kube-scheduler`: lo *Scheduler* invia comandi al `kubeapi-server` per cui è un suo client.
* `kube-controller-manager`: il *Controller Manager* è un client di `kubeapi-server`.
* `kube-proxy`: il *Kube Proxy* è un client di `kubeapi-server`.

Oltre a questi certificati, vi sono anche i certificati della *Certificate Authority*. Kubernetes richiede almeno una CA, ma se ne possono anche avere più di una.

![Componenti che richiedono certificati nel cluster](../assets/section-7/cluster_certs.PNG)

*Nota*: Le public key hanno in genere l'estensione `crt` o `pem`, mentre le private key hanno l'estensione `key` oppure `-key.pem`.

### Creare il Certificato della CA

Per prima cosa, viene generato il certificato della *Certificate Authority*. In questo caso, il certificato è self-signed:

```shell
# Genera la chiave privata
openssl genrsa -out ca.key 2048
# Genera la sign request
openssl req -new -key ca.key -subj "/CN=KUBERNETES-CA" -out ca.csr # CN=Common Name, il nome del servizio
# Firma la richiesta con la chiave privata e genera il certificato (chiave pubblica)
openssl x509 -req -in ca.csr -signkey ca.key -out ca.crt
```

Questo certificato verrà usato per firmare tutte gli altri certificati generati.

__NB__: è necessario distribuire il certificato CA a __tutti__ i componenti che dovranno autenticarsi nel cluster.

### Creare il Certificato dell'Amministratore

Si può procedere poi a creare gli altri certificati. Il certificato per l'amministratore sarà un certificato per client, essendo l'amministratore un client che si connette al cluster:

```shell
openssl genrsa -out admin.key 2048
# Genera il certificato per l'amministratore e lo inserisce nel gruppo dei masters
openssl req -new -key admin.key -subj "/CN=kube-admin/O=SYSTEM:MASTERS" -out admin.csr
# firmato con la chiave della CA
openssl x509 -req -in admin.csr -CA ca.crt -CAkey ca.key -out admin.crt
```

E importante ricordasi di insererire `/O=SYSTEM:MASTERS` in modo che l'utente venga assegnato al gruppo degli amministratori.

*Nota*: il CN può essere uno qualsiasi, ma è importante assegnare un nome parlante perchè è questo il nome utilizzato per l'autenticazione e comparirà nei logs.

Per autorizzare una chiamata `curl` con un certificato TLS:

```shell
curl https://kube-apiserver:6443/api/v1/pods \
    --key admin.key 
    --cert admin.crt 
    --cacert ca.crt
```

*Nota*: l'opzione `--cacert` è necessaria perchè il certificato è self-signed. Un certificato proveniente da una vera CA dovrebbe già essere contenuto nel sistema. Se si vuole, è possibile inserire anche il `ca.crt` self-signed tra i certificati riconosciuti nel sistema, copiandolo nella cartella `/etc/pki/tls/certs` (o `/etc/ssl/certs`).

### Creare i Certificati dei Componenti di Sistema

I componenti di sistema sono a loro volta dei client di `kubeapi-server`. Vengono creati allo stesso modo del certificato di amministratore, ma è necessario ricordarsi del prefisso `SYSTEM:` nel CN. Si procede quindi alla generazione dei certificati per `kube-scheduler`, `kube-controller-manager`, `kube-proxy`:

```shell
openssl genrsa -out scheduler.key 2048
openssl req -new -key scheduler.key -subj "/CN=system:kube-scheduler" -out scheduler.csr
openssl x509 -req -in scheduler.csr -CA ca.crt -CAkey ca.key -out scheduler.crt
```

### Creare i Certificati per etcd

I certificati vengono generati allo stesso modo di tutti gli altri. In un setup in cui anche `etcd` è replicato, è necessario generare i certificati per ogni replica.
I certificati vanno poi specificati all'avvio del servizio, oppure se si utilizza `kubeadmin`, nel manifest `etcd.yaml`:

```yaml
command:
    - etcd
    --key-file=/path-to-certs/etcdserver.key
    --cert-file=/path-to-certs/etcdserver.crt
    --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
    # se ci sono repliche:
    --peer-cert-file=/path-to-certs/etcdpeer1.crt
    --peer-client-cert-auth=true
    --peer-key-file=/etc/kubernetes/pki/etcd/peer.key
    --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
```

### Creare i Certificati per KubeAPI Server

I certificati per il `kubeapi-server` vengono sempre creati con lo stesso procedimento. Il `kubeapi-server` è il componente che riceve chiamate da tutti gli altri componenti. Diversi servizi e componenti potrebbero chiamare il `kubeapi-server` con un diverso nome dal CN. Per questo motivo è necessario configurare anche gli alias. Per fare cioè è necessario creare un file di configurazione prima della creazione della chiave:

```conf
[req]
req_extensions= v3_req
[v3_req]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation,
subjectAltName = @alt_names
[alt_names]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.sve
DNS.4 = kubernetes.default.svc.cluster.local
IP.1 = 10.96.0.1
IP.2 = 172.17.0.87
```

e poi generare la chiave specificando il file di configurazione:

```shell
openssl genrsa -out apiserver.key 2048
openssl req -new -key apiserver.key -subj "/CN=kube-apiserver" -out apiserver.csr --config openssl.cnf
openssl x509 -req -in apiserver.csr -signkey apiserver.key -out apiserver.crt
```

È possibile anche generare i certificati client di `kubeapi-server`, usati per le connessioni a `kubelet` e `etcd`. Tutti questi certificati vanno specificati all'avvio del servizio:

```shell
ExecStart=/usr/local/bin/kube-apiserver \\
# Certificati client per etcd
--etcd-cafile=/var/lib/kubernetes/ca.pem \\
--eted-certfile=/var/lib/kubernetes/apiserver-etcd-client.crt \\
--etcd-keyfile=/var/lib/kubernetes/apiserver-etcd-client.key \\
# Certificati client per kubelet
--kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \\
--kubelet-client-certificate=/var/lib/kubernetes/apiserver-eted-client.crt \\
--kubelet-client-key=/var/lib/kubernetes/apiserver-etcd-client.key \\
# Certificati kubeapi server
--client-ca-file=/var/lib/kubernetes/ca.pem \\
--tls-cert-file=/var/lib/kubernetes/apiserver.crt \\
--tls-private-key-file=/var/lib/kubernetes/apiserver.key \\
```

### Creare i Certificati per ogni Kubelet

Ogni nodo del cluster deve avere un certificato. Il certificato verrà nominato col nome del nodo, e verrà aggiunto a `kubelet-config.yaml`:

```yaml
kind: KubeletConfiguration
apiVersion: kubelet.contig.k8s.io/v1beta1
authentication:
    x509:
        clientCAFile: "/var/lib/kubernetes/ca.pem"
authorization:
    mode: Webhook
clusterDomain: "cluster.local"
clusterdNS:
    - "10.32.0.10"
tlsCertfile: "/var/lib/kubelet/kubelet-node01.crt"
tlsPrivateKeyFile: "/var/lib/kubelet/kubelet-node01.key"
```

Kubelet comunica anche con il KubeAPI Server come client, per cui è necessario generare anche i certificati client per ogni nodo. Il KubeAPI Server deve conoscere quale nodo lo sta contattando, per cui il CN deve seguire la nomenclatura `SYSTEM:NODE:<nome_nodo>`:

```shell
openssl genrsa -out node01-client.key 2048
openssl req -new -key node01-client.key -subj "/CN=SYSTEM:NODE:NODE01/O=SYSTEM:NODES" -out node01-client.csr
openssl x509 -req -in node01-client.csr -CA ca.crt -CAkey ca.key -out node01-client.crt
```

Per avere i permessi corretti, il nodo deve essere dentro il gruppo `SYSTEM:NODES`.

### Vedere i Dettagli dei Certificati

È importante tenere sotto controllo e organizzati tutti i certificati del cluster, magari su una tabella che contenga tutti i dettagli di ogni certificato (vedi [certs-checker.xlsx](/assets/section-7/kubernetes-certs-checker.xlsx)).
Per vedere i dettagli di un certificato nel cluster:

```shell
openssl x509 -in /path/to/certificate -text -noout
```

Prendere nota sopratutto dei seguenti parametri:

* *Issuer*: è la CA. Se si utilizza `kubeadm`, il valore sarà `CN=kubernetes`.
* *Validity*: il periodo di validità del certificato.
* *Subject*: il *Common Name* del servizio che si sta certificando.
* *Alternative Name*: tutti i nomi alternativi del CN.

### Debugging dei Problemi TSL

Se vi sono dei problemi nel cluster riguardanti la connettività, è possibile controllare i log dei vari servizi per verificare che non siano problemi di certificati.

Se i servizi sono stati installati da zero nella macchina, si possono vedere i log di sistema con:

```shell
journalctl -u etcd.service -l
```

Se invece si utilizza `kubeadm`:

```shell
kubectl logs etcd-master
```

Se i problemi riguardano direttamente `kubeapi-server` o `etcd` potrebbe non funzionare il comando `kubectl`, per cui è necessario andare a controllare i log direttamente nei container docker:

```shell
docker ps -a
docker logs <id_container>
```

## API per la Generazione dei Certificati

Il *Master Node* del cluster contiene i certificati CA, per cui il *Master* è anche la *Certificate Authority* del cluster.
Ogniqualvolta bisogna generare un nuovo certificato (ad esempio, per un nuovo amministratore), è necessario che un utente già certificato entri nel *Master* e firmi la nuova CSR con il certificato CA presente nel server.

Kubernetes espone però un API che permette di firmare i certificati senza accedere al nodo e firmandoli manualmente, ma passando solo attraverso `kubectl`. Questa procedura è gestita dal *Controller Manager*, e in particolare dai controller *CRS-APPROVING* e *CSR-SIGNING*. Prima di poter utilizzare l'API, devono essere configurate le chiavi CA nel `kube-controller-manager.yaml` (o nel relativo servizio nel caso fosse installato nel sistema senza `kubeadm`):

```yaml
spec:
    containers:
        command:
        - kube-controller-manager
        - --cluster-signing-cert-file=/etc/kubernetes/pki/ca.crt
        - --cluster-signing-key-file=/etc/kubernetes/pki/ca.key
        # ...
```

La procedura per la creazione di un nuovo certificato tramite API è la seguente:

1. Il nuovo utente crea una chiave e la sua csr:

```shell
openssl genrsa -out user.key 2048
openssl req -new -key user.key -subk "/CN=user" -out user.csr
```

2. L'amministratore del cluster crea un oggetto *Certificate Signing Request* da inviare al cluster:

```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: user
spec:
  signerName: kubernetes.io/kube-apiserver-client # a chi è diretta la richiesta
  groups:
  - system:authenticated
  usages:
  - client auth
  request:
    /path/to/csr #in base64
```

*Nota*: la csr deve essere codificata in base64:

```shell
cat user.csr | base64 | tr -d "\n"
```

*Nota*: la proprietà `signerName` è obbligatoria dalla versione 1.19. Indica a chi è diretta la richiesta di firma. In genere è semppre `kubernetes.io/kube-apiserver-client`, che dirma i certificati per accedere al `kubeapi-server` come client.

3. Tutti gli amministratori già certificati possono vedere, revisionare la richiesta e approvarla:

```shell
kubectl get csr
kubectl certificate approve <nome_certificato> # es: user
# per rifiutarlo:
kubectl certificate deny <nome_certificato>
```

4. Il certificato firmato può essere visto con:

```shell
kubectl get csr <nome_certificato> -o yaml
```

Il certificato firmato sarà visibile nella proprietà `certificate` in `base64`:

```yaml
# ...
status:
  certificate:
Ef6IgJGsre... # in base 64
# ...
```

Per decodificarlo:

```shell
echo <certificato> | base64 --decode
```

5. Il certificato decodificato dovrà essere dato al nuovo utente che potrà ora essere autenticato nel cluster.

## KubeConfig

Quando si utilizza il comando `kubectl`, questo una configurazione di default per inviare i comandi al cluster. Molte volte si hanno però molti cluster da gestire, ed ogni cluster richiede l'accesso con utenti diversi e certificati diversi.
È possibile specificare manualmente, ad ogni comando, il cluster di destinazione e i certificati necessari:

```shell
kubectl get pods
    --servermy-kube-playground:6443
    --client-key admin.key
    --client-certificate admin.crt
    --certificate-authority ca.crt
```

ma questo procedimento sarebbe troppo complicato. La soluzione migliore è utilizzare un file di configurazione e indicare solo quello nel comando

```shell
kubectl get pods --kubeconfig config
```

Di default, `kubectl` cerca il file in `$HOME/.kube/config`, per cui se si salva la configurazione in questa posizione, non è neanche necessario usare l'opzione `--kubeconfig`.

Vi sono tre campi da configurare in *KubeConfig*:

* `clusters`: i server a cui ci si vuole connettere.
* `users`: gli utenti che si vogliono utilizzare per la connessione al cluster.
* `contexts`: associa gli utenti ai cluster.

Il file di configurazione è un manifest *yaml*:

```yaml
apiVersion: v1
kind: Config
current-context: dev@kube-dev
clusters:
  - name: kube-dev
    cluster:
        certificate-authority: ca.crt
        server: https://kubernetes:6443
users:
  - name: dev
    user:
        client-certificate: admin.crt
        client-key: admin.key
contexts:
  - name: dev@kube-dev
    context:
        cluster: kube-dev
        user: dev
        namespace: finance # opzionale, di default va su default namespace
```

`current-context` indica il contesto attuale al quale `kubectl` invierà i comandi di default. Per cambiare il contesto:

```shell
kubectl set-context prod@kube-prod
```

*Nota*: questo comando aggiorna anche il file di configurazione.

Per vedere la configurazione direttamente con `kubectl`:

```shell
kubectl config view
# se la configurazione non è su $HOME/.kube/config bisogna specificarla
kubectl config view --kubeconfig=my-custom-config
```

*Nota*: è possibile operare sulla configurazione completamente da `kubectl` con tutti i comandi visibili in `kubectl config -h`.

### Certificati in KubeConfig

I certificati presenti nella configurazione possono essere indicati in due diversi modi:

* `certificate`: inserendo la path del certificato (es: `certificate-authority: /path/to/crt`).
* `certificate-data`: inserendo il contenuto del certificato in base64 (es: `certificate-authority-data: LS0tLS1CRU...`).

## Specifiche API e Gruppi

Le API di Kubernetes sono raggiungibili tramite `kubectl` ma anche tramite delle classiche API REST. Vi sono diversi path delle API:

* `/version`: ritorna informazioni sulla versione delle API.
* `/metrics`: contiene risorse sulle metriche, usate per controllare lo stato del cluster.
* `/healthz`: contiene API per controllare la salute del cluster.
* `/logs`: contiene risorse per l'integrazione con sistemi di monitoraggio e log esterni.
* `/api`: contiene le Core API, sono le API più vecchie con i servizi principali di Kubernetes.
* `/apis`: contiene tutti gli altri gruppi di API. Sono più recenti. Ogni gruppo è diviso in Risorse e Verbi.

![Gruppi e risorse delle named APIs](/assets/section-7/named_apis.PNG)

Tutte le risorse (che verrano poi utilizzate per la definizione dei ruoli) sono elencabili con:

```shell
kubectl api-resources
```

Se si prova ad accedere alle API che richiedono autenticazione verrà restituito un errore. Infatti deve essere precisato, ad ogni chiamata, la posizione dei certificati che si vogliono utilizzare per l'autenticazione.

```shell
curl https://localhost:6443 -k
    --key admin. key
    --cert admin.crt
    --cacert ca.crt
```

Per evitare di dover inserire questi comandi ogni volta, è possibile utilizzare il `kubectl proxy`, che legge questi certificati dalla `kubeconfig` ed espone un indirizzo locale al quale possono essere inviate tutte le richieste alle API. È il proxy che si occupa di inviare anche i certificati.

```shell
kubectl proxy # espone localhost:8001
curl http://localhost:8001 -k # non è piu necessario indicare i certificati
```

## Permessi e Autorizzazioni

Dopo che un utente è stato autenticato, è necessario verificare quali permessi ha l'utente e su cosa può operare nel cluster. Vi sono diversi modi di autorizzare l'utente:

* *Node*: autorizza le richieste fatte dai `kubelets` nei vari nodi, verso il `kubeapi-server`. Per utilizzare questa autorizzazione, è necessario che l'utente abbia uno username del tipo `system:node:<nome_nodo>` e il gruppo `system:nodes`.
* *ABAC*: vengono definiti i permessi utente per utente, in un file JSON. È necessario definire manualmente tutti i permessi, ed ogni modifica richiede il riavvio dell'Kube API Server. Non viene utilizzato spesso.
* *RBAC*: Vengono definiti dei ruoli con i loro permessi. Gli utenti vengono associati al ruolo. Le modifiche vanno apportate ai permessi del ruolo, e si riflettono su tutti gli utenti che appartengono a quel ruolo.
* *Webhook*: utilizzato per integrare un sistema di autorizzazione esterno (es: [Open Policy Agent](https://www.openpolicyagent.org)). Il controllo dell'autorizzazione viene fatto dal sistema esterno, che utilizza il webhook per dare una conferma o negare l'autorizzazione al cluster.

*Nota*: vi sono altri due tipo di autorizzazioni: *Always Allow* e *Always Deny*, che permettono o negano tutte le autorizzazioni.

Il tipo (o i tipi) di autorizzazione vengono definiti nel `kubeapi-server`, con l'opzione `--authorization-mode`. Di default, l'opzione selezionata è `AlwaysAllow`.
Se vengono inseriti più tipi di autorizzazione, il sistema verifica i permessi in ordine a partire dal primo metodo. Si continua a verificare i permessi procedendo col metodo successivo. Se l'utente viene autorizzato, la procedura si ferma e non vengono chiamati gli altri metodi di autorizzazione.

## RBAC

I ruoli vengono definiti in un file *yaml*, con le parole chiave specificate nel paragrafo [Specifiche API e Gruppi](#specifiche-api-e-gruppi):

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
rules:
- apiGroups: [""] # "" indica il core API group
  resources: ["pods"]
  verbs: ["get", "list", "update", "delete", "create"]
  resourceNames: ["frontend-pod"] # opzionale, specifico con precisione il nome dei pod
- apiGroups: [""]
  resources: ["ConfigMap"]
  verbs: ["create"]
```

`resourceNames` può limitare con precisione il nome delle risorse alle quali si può accedere.
È necessario poi associare il ruolo ad un utente con un *RoleBinding*:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: devuser-developer-binding
subjects:
  # SINGOLO UTENTE
- kind: User
  name: dev-user # case sensitive
  apiGroup: rbac.authorization.k8s.io
  # GRUPPI
- kind: Group
  name: test-users
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

*Nota*: Role e RoleBinding tengono anche conto del `namespace`. Se nessun `namespace` viene definito nei `metadata`, i permessi saranno validi per il default namespace.

Per vedere i Role e RoleBindings:

```shell
kubectl get roles
kubectl get rolebindings
kubectl describe role <nome_ruolo>
kubectl describe rolebindings <nome_role_binding>
```

### Verificare se un Utente può eseguire un Comando

Se si è loggati nel cluster con un certo utente, e si vuole verificare se con questo utente si può eseguire un certo comando, utilizzare:

```shell
kubectl auth can-i <comando>
# esempio:
kubectl auth can-i delete nodes # => no
```

Per controllare i permessi di un altro utente senza loggarsi, usare l'opzione `--as`:

```shell
kubectl auth can-i <comando> --as dev-user
# per selezionare un ben preciso namespace:
kubectl auth can-i <comando> --as dev-user --namespace test
```

### Ruoli Cluster

Non tutte le risorse sono inseribili in un namespace. Alcune risorse sono condivise in tutto il cluster, in particolare i Nodi e i PV. Per gestire i ruoli anche per le risorse *Cluster Scoped* è possibile utilizzare i *Cluster Roles* e i relativi *Cluster Role Bindings*:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-administrator
rules:
- apiGroups: [""] # "" core API
  resources: ["nodes"]
  verbs: ["get", "list", "delete", "create"]
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-admin-role-binding
subjects:
- kind: User
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-administrator
  apiGroup: rbac.authorization.k8s.io
```

*Nota*: i *Cluster Roles* possono essere usati anche per risorse che possono essere *namespaced* (come i Pod). In questo modo, si da l'accesso a quella risorsa su __tutti i namespace__.

*Nota*: in un cluster possono esserci molti ruoli. Per contarli facilmente usare:

```shell
kubectl get clusterroles -A --no-headers | wc -l
```

## Admission Controllers

L'autenticazione RBAC permette di controllare le autorizzazioni a livello di API. È possibile sia necessario invece controllare con molto più dettaglio i permessi che un utente può avere, ad esempio a livello di definizione del manifest (non può modificare l'immagine, oppure le immagini devono provenire solo da un registro privato ecc.). Per controllare queste autorizzazioni si utilizzano gli *Admission Controllers*. Gli Admission Controller possono anche mutare le richieste o effettuare automaticamente operazioni nel cluster. Vi sono vari controller predefiniti all'interno di Kubernetes, come ad esempio:

* `AlwaysPullImages`: obbliga a scaricare sempre le immagini.
* `EventRateLimit`: limita le chiamate alle API di Kubernetes.
* `NamespaceExists`: blocca tutte le richieste verso un namespace che non esiste (attivo di default).
* `NamespaceAutoProvision`: crea automaticamente un namespace se non esiste (non è attivo di default).
* `DefaultStorageClass`: aggiunge automaticamente la storage class a una PVC che non la specifica.

Gli *Admission Controllers* verificano i permessi dopo aver passato Autenticazione e Autorizzazione (in questo caso RBAC).

Per ottenere tutti gli Admission Controllers attivi di default:

```shell
kube-apiserver -h | grep enable-admission-plugins
# se si utilizza kubeadm, kube-apiserver è un Pod, quindi:
kubectl exec kube-apiserver-controlplane -n kube-system -- kube-apiserver -h | grep enable-admission-plugins
```

Per abilitare un plugin, aggiungerlo all'opzione `--enable-admission-plugins` di `kube-apiserver`, per disabilitarne uno attivo di default, inserirlo in `--disable-admission-plugins`.

*Nota*: nelle ultime versioni di Kubernetes, `NamespaceExists` e `NamespaceAutoProvision` sono deprecati, e si utilizza `NamespaceLifecycle`, che assicura anche che i namespace *default*, *kube-public* e *kube-system* non possano essere cancellati. Inoltre, rifiuta sempre le richieste a namespace che non esistono.

## Account di Servizio

I *Service Account* sono utenze utilizzati da bot o servizi che devono accedere in modo autonomo al cluster per leggere alcune informazioni (es: metriche, salute del cluster).
Per creare un nuovo service account:

```shell
kubectl create serviceaccount <nome_account>
# elenco di tutti gli account di servizio
kubectl get serviceaccount
# dettagli sull'account di servizio
kubectl describe serviceaccount <nome_account>
```

Quando si crea un *Service Account*, viene automaticamente creato un token utilizzato come *Bearer Token* nell'header di autenticazione. Il token è contenuto in un secret, che si può visualizzare con:

```shell
kubectl describe secret <nome_token_account> # es: account-token-kbbdm
```

Il nome del token è visibile sul `describe` del account di servizio.

### Account di Servizio per i Servizi Interni al Cluster

Se un servizio che necessita di leggere le API di Kubernetes è deployato all'interno del token, non è necessario generare un *Service Account* ed autenticarsi tramite token. Il secret può essere montato come volume sul Pod in modo che ne abbia visibilità.

Quando un `namespace` viene creato, viene creato un `serviceaccount` di default che viene montato in __tutti__ i Pod nel `namespace` come volume (visibile con `kubectl describe pod <pod>`).

*Nota*: il *Service Account* di default è molto limitato.

Per aggiungere un diverso Service Account al Pod, utilizzare il campo `serviceAccountName`:

```yaml
apiVersion: v1
kind: Pod
metadata:
    name: dashboard-pod
spec:
    containers:
      - name: dashboard-pod
        image: dashboard
    serviceAccountName: dashboard-sa
```

*Nota*: non si puà cambiare il Service Account su un Pod senza prima cancellarlo. Si può invece modificare su un Deployment, che farà partire un rollout per la modifica.

*Nota*: per evitare che il Pod monti automaticamente il Service Account di default, utilizzare il campo `automountServiceAccountToken: false`.

## Docker Registry Privato

Per scarica in un Pod un'immagine Docker privata, è necessario inserire l'indirizzo completo dell'immagine nel registro, ma è anche ncessario effettuare l'accesso a questo registro. Le credenziali vengono salvate in uno specifico tipo di Secret chiamato `docker-registry`:

```shell
kubectl create secret docker-registry regcred \
--docker-server=
--docker-username=
--docker-password=
--docker-email=
```

che viene inserito nel manifest del Pod:

```yaml
# ...
spec:
    containers:
    - name: nginx
      image: # ...
    imagePullSecrets:
    - name: regcred
```

## Contesti di Sicurezza

### Contesti di Sicurezza in Docker

Tutti i processi all'interno di container Docker vengono eseguiti come root. L'utente di root all'interno del container __è diverso__ da quello dell'host. In ogni caso, non sempre si vuole eseguire i processi con root, quindi si può specificare l'utente utilizzato all'interno dei container con:

```shell
docker run --user=1000 ubuntu
```

oppure nel Dockerfile:

```dockerfile
FROM ubuntu:latest
# ...
USER 1000
# ...
```

In aggiunta, si possono aggiungere e rimuovere le *Linux Capabilities* dell'utente, cioè le funzionalità avanzate del kernel di Linux, che possono avere forti ripercussioni sul sistema se usate incorrettamente.
Per modificare le *Capabilities*:

```shell
# Aggiungere un capability
docker run --cap-add MAC_ADMIN ubuntu
# Rimuovere un capability
docker run --cap-drop MAC_ADMIN ubuntu
# Aggiungere tutte le capabilities, massimi privilegi
docker run --privileged ubuntu
```

*Nota*: la lista di tutte le *Capabilities* di Linux sono presenti in `/usr/include/linux/capability.h`.

### Contesti di Sicurezza in Kubernetes

È possibile modificare i *Security Context* sia a livello del Container che dell'intero Pod. È ad esempio possibile modificare l'utente che esegue i processi del container, oppure aggiungere o togliere alcune *Capabilities* di Linux per esempio:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
spec:
  securityContext: # a livello di Pod
    runAsUser: 1000
  containers:
  - name: ubuntu
    image: ubuntu
    # ...
    securityContext: # a livello di singoli container
      runAsUser: 1000
      capabilities:
        add: ["MAC_ADMIN"]
```

La configurazione applicata al Pod viene automaticamente letta anche da __tutti__ i container all'interno del Pod. Se si configura anche un `securityContext` dello specifico container, questo ha la priorità su quello del Pod.

## Policy del Network

Di default, tutti i Pod in tutti i nodi possono comunicare tra di loro nel network interno del cluster. Molte volte però si vuole limitare le connessioni a quelle necessarie (ad esempio: il frontend non dovrebbe connettersi direttamente al database). Ciò può essere gestito attraverso le *Network Policies*, che permettono di limitare le connessioni in ingresso (*Ingress*) ed uscita (*Egress*) dai Pod:

```yaml
# db-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
 name: db-policy
spec:
  podSelector:
    matchLabels:
      role: db # tutti i Pod con role = db
  policyTypes:
  - Ingress # Ingress/Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: api-pod # db riceve solo da api-pod
    ports:
    - protocol: TCP # e solo alla porta 3306
    port: 3306
  - from:
    - podSelector:
    # ...
    ports:
    # ...
```

Quando si definisce un *Ingress*, automaticamente viene permesso l'invio della risposta al Pod che aveva fatto la richiesta, senza definire alcuna regola aggiuntiva. Ovviamente, le richieste in uscita dal Pod vengono ancora bloccate, e necessitano di una regola *Egress* perchè la chiamata vada a buon fine.

*Nota*: le Network Policies vengono gestite dal network di Kubernetes. Ci sono varie soluzioni per gestire il network, ma non tutte supportano le policies (es: *flannel* non le supporta). Le policy possono comunque essere create ma verranno ignorate.

### Network Policy per un Namespace

Di default, la Network Policy è applicata per __tutti__ i namespace. È necessario aggiungere il tag `namespaceSelector` per indicare anche il namespace dei Pod che si stanno selezionando:

```yaml
# ...
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: api-pod
      namespaceSelector:
        matchLabels:
          name: prod
# ...
```

Se si seleziona solo il namespace, e nessun Pod, la regola vale per qualsiasi Pod del namespace selezionato.

### Network Policy per IP

È possibile anche indicare un IP dal quale si possono ricevere chiamate:

```yaml
# ...
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: api-pod
      # AND
      namespaceSelector:
        matchLabels:
          role: prod
    # OR
    - ipBlock:
        cidr: 192.168.5.10/32
# ...
```

Le regole nella lista del tag `from` funzionano come un *OR*, per cui basta che una delle due regole sia valida per lasciar passare la richiesta. `podSelector` e `namespaceSelector` funzionano invece come un *AND*, quindi i due criteri devono essere entrambi rispettati.
È possibile combinare in modo diverso le regole:

```yaml
# ...
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: api-pod
    # OR
    - namespaceSelector:
        matchLabels:
          role: prod
    # OR
    - ipBlock:
        cidr: 192.168.5.10/32
# ...
```

In questo caso tutte le regole sono separate, e solo una deve essere valita perchè la comunicazione sia possibile.

### Testare la Network Policy

È possibile elencare le Network Policies con:

```shell
kubectl get networkpolicies
```

ma controllare la configurazione della policy non assicura il suo corretto funzionamento. È possibile verificare il funzionamento con più precisione il funzionamento entrando nei container interessati e facendo dei test con alcune utility come `telnet` o `nslookup`:

```shell
telnet <ip_pod> <porta>
kubectl exec <nome_pod> --restart=Never -it -- nslookup <ip_pod>:<porta>.<namespace>.pod.cluster.local
```

1. [CKA Course - Security](https://github.com/kodekloudhub/certified-kubernetes-administrator-course/tree/master/docs/07-Security)
2. [PKI Certificates and Requirements](https://kubernetes.io/docs/setup/best-practices/certificates/)
