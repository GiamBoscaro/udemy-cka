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

1. [CKA Course - Security](https://github.com/kodekloudhub/certified-kubernetes-administrator-course/tree/master/docs/07-Security)
