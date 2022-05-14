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
* Provider esterni
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

```bash
# kube-apiserver.service
--basic-auth-file=user-details.cs
```

È necessario riavviare il servizio perchè le modifiche abbiano effetto. Se si utilizza `kubeadm`, è possibile inserire il comando sul manifest in `/etc/kubernetes/manifests/kube-apiserver.yaml` e il rispettivo Pod verrà ricreato automaticamente.

Per autenticarsi durante le chiamate:

```bash
curl -v -k https://master-node-ip:6443/api/v1/pods -u "user:password"
```

### File Statico con Token

Al posto di avere un file di password, è possibile usare un file di token:

```csv
token,username,userid,groupid
```

che va indicato al `kubeapi-server` con l'opzione `--token-auth-file`.

L'autenticazione viene effettuata con un Bearer token nell'header della chiamata:

```bash
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

```bash
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

```bash
openssl genrsa -out admin.key 2048
# Genera il certificato per l'amministratore e lo inserisce nel gruppo dei masters
openssl req -new -key admin.key -subj "/CN=kube-admin/O=SYSTEM:MASTERS" -out admin.csr
# firmato con la chiave della CA
openssl x509 -req -in admin.csr -CA ca.crt -CAkey ca.key -out admin.crt
```

E importante ricordasi di insererire `/O=SYSTEM:MASTERS` in modo che l'utente venga assegnato al gruppo degli amministratori.

*Nota*: il CN può essere uno qualsiasi, ma è importante assegnare un nome parlante perchè è questo il nome utilizzato per l'autenticazione e comparirà nei logs.

Per autorizzare una chiamata `curl` con un certificato TLS:

```bash
curl https://kube-apiserver:6443/api/v1/pods \
    --key admin.key 
    --cert admin.crt 
    --cacert ca.crt
```

*Nota*: l'opzione `--cacert` è necessaria perchè il certificato è self-signed. Un certificato proveniente da una vera CA dovrebbe già essere contenuto nel sistema. Se si vuole, è possibile inserire anche il `ca.crt` self-signed tra i certificati riconosciuti nel sistema, copiandolo nella cartella `/etc/pki/tls/certs` (o `/etc/ssl/certs`).

### Creare i Certificati dei Componenti di Sistema

I componenti di sistema sono a loro volta dei client di `kubeapi-server`. Vengono creati allo stesso modo del certificato di amministratore, ma è necessario ricordarsi del prefisso `SYSTEM:` nel CN. Si procede quindi alla generazione dei certificati per `kube-scheduler`, `kube-controller-manager`, `kube-proxy`:

```bash
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

```bash
openssl genrsa -out apiserver.key 2048
openssl req -new -key apiserver.key -subj "/CN=kube-apiserver" -out apiserver.csr --config openssl.cnf
openssl x509 -req -in apiserver.csr -signkey apiserver.key -out apiserver.crt
```

È possibile anche generare i certificati client di `kubeapi-server`, usati per le connessioni a `kubelet` e `etcd`. Tutti questi certificati vanno specificati all'avvio del servizio:

```bash
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

```bash
openssl genrsa -out node01-client.key 2048
openssl req -new -key node01-client.key -subj "/CN=SYSTEM:NODE:NODE01/O=SYSTEM:NODES" -out node01-client.csr
openssl x509 -req -in node01-client.csr -CA ca.crt -CAkey ca.key -out node01-client.crt
```

Per avere i permessi corretti, il nodo deve essere dentro il gruppo `SYSTEM:NODES`.

### Vedere i Dettagli dei Certificati

È importante tenere sotto controllo e organizzati tutti i certificati del cluster, magari su una tabella che contenga tutti i dettagli di ogni certificato (vedi [certs-checker.xlsx](/assets/section-7/kubernetes-certs-checker.xlsx)).
Per vedere i dettagli di un certificato nel cluster:

```bash
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

```bash
journalctl -u etcd.service -l
```

Se invece si utilizza `kubeadm`:

```bash
kubectl logs etcd-master
```

Se i problemi riguardano direttamente `kubeapi-server` o `etcd` potrebbe non funzionare il comando `kubectl`, per cui è necessario andare a controllare i log direttamente nei container docker:

```bash
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

```bash
openssl genrsa -out user.key 2048
openssl req -new -key user.key -subk "/CN=user" -out user.csr
```

2. L'amministratore del cluster crea un oggetto *Certificate Signing Request* da inviare al cluster:

```yaml
apiVersion: certificates.k8s.io/v1beta1
kind: CertificateSigningRequest
metadata:
  name: user
spec:
  groups:
  - system:authenticated
  usages:
  - digital signature
  - key encipherment
  - server auth
  request:
    /path/to/csr #in base64
```

*Nota*: la csr deve essere codificata in base64:

```bash
cat user.csr | base64 | tr -d "\n"
```

3. Tutti gli amministratori già certificati possono vedere, revisionare la richiesta e approvarla:

```bash
kubectl get csr
kubectl certificate approve <nome_certificato> # es: user
```

4. Il certificato firmato può essere visto con:

```bash
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

```bash
echo <certificato> | base64 --decode
```

5. Il certificato decodificato dovrà essere dato al nuovo utente che potrà ora essere autenticato nel cluster.

1. [CKA Course - Security](https://github.com/kodekloudhub/certified-kubernetes-administrator-course/tree/master/docs/07-Security)
2. [PKI Certificates and Requirements](https://kubernetes.io/docs/setup/best-practices/certificates/)
