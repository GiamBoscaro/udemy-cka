# OTHER TOPICS

## JSONPath

Kubectl comunica con API Server per ricevere le informazioni richieste. API Server fornisce le informazioni in formato JSON, ma non tutto viene visualizzato nel terminale, neanche con l'opzione `-o wide`.
Per vedere tutti i dati forniti da API Server, mostrare i risultati in formato *yaml* o *json*:

```shell
kubectl get nodes -o json
```

Osservando la struttura dei dati, è possibile ricavare informazioni ben precise utilizzando una query jsonpath:

```shell
kubectl get pods -o=jsonpath=<query>
```

Alcuni esempi di query:

```shell
# l'immagine utilizzata dal primo container del primo pod in lista
kubectl get pods -o=jsonpath='{.items[0].spec.containers[0].image}'
# tutti i nomi di tutti i nodi nel cluster
kubectl get nodes -o=jsonpath='{.items[*].metadata.name}'
# due query combinate: tutte le architetture dei nodi e tutte le cpu dei nodi
kubectl get pods -o=jsonpath='{.items[*].status.nodeInfo.architecture}{'\n'}{.items[*].status.status.capacity.cpu}'
# output => master node01
#           4      4
```

*Nota*: le query vanno sempre inserite all'interno di `'{}'`, sempre con __singoli apici__.
*Nota*: per rendere più leggibili i risultati, è possibile inserire dei caratteri di formattazione come `'{\n}'` o `'{\t}'`.

### Cicli Iterativi

Il risultato dell'ultima query non è molto leggibile, ma è possibile utilzzare un ciclo iterativo per iterare nei dati e stamparli in modo più ordinato. Un ciclo è formato dai comandi `{range .items[*]} {end}`, ad esempio:

```shell
# Nome nodo          Numero CPU del nodo
'{range .items[*]}
{.metadata.name}{'\t'}{.status.capacity.cpu}{'\n'}
{end}'
```

Quindi la query da inserire in `kubectl`:

```shell
kubectl get pods -o=jsonpath='{range .items[*]}{.metadata.name}{'\t'}{.status.capacity.cpu}{'\n'}{end}'
```

### Colonne Personalizzate

```shell
kubectl get pods -o=custom-columns=<nome-colonna>:<jsonpath>
# stampo la colonna dei nomi dei nodi e delle cpu, con i relativi headers
kubectl get pods -o=custom-columns=NODE:.metadata.name,CPU:.status.capacity.cpu
# NODE   CPU
# master 4
# node01 4
```

*Nota*: non deve essere inserita la path `items[*]` in quando è sottinteso che si sta iterando nella lista di risultati ottenuti dal comando.

### Ordinamento

Si possono anche ordinare i risultati in funzione di una JSON PATH:

```shell
kubectl get nodes --sort-by=.metadata.name
```

### Selettori Avanzati

È possibile utilzzare dei comandi più avanzati per selezionare il valore delle chiavi del JSON.
Preso questo JSON di esempio:

```json
{
"contexts": [
        {
            "name": "aws-user@kubernetes-on-aws",
            "context": {
                "cluster": "kubernetes-on-aws",
                "user": "aws-user"
            }
        },
        // ....
    ]
}
```

Se si vuole selezionare il nome del `context` (`contexts[?].name`) in funzione dell'utente associato ad esso (che si trova in `context.user`)

```shell
kubectl config view --kubeconfig=my-kube-config \
-o jsonpath="{.contexts[?(@.context.user=='aws-user')].name}"
```

## Helm

Helm è un packet manager, install wizard e molto altro per Kubernetes. Permette di deployare nel cluster applicazioni complesse con un solo comando (es: Wordpress), aggiornarle o cancellarle facilmente. L'installazione può essere configurata tramite un solo file `values.yaml` che contiene tutte le variabili per personalizzare l'installazione.

### Installazione

Per installare Helm, se il sistema supporta `snap`, la versione ufficiale sempre aggiornata è installabile con;

```shell
sudo snap install helm --classic
```

Tutti gli altri metodi di installazione sono gestiti dalla community e sono in genere sempre aggiornati. Ad esempio, con `apt`:

```shell
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

### Come Funziona

I manifest forniti da Helm sono dei template personalizzabili, alcuni valori delle proprietà sono delle variabili che vengono lette dal file `values.yaml`, ad esempiop

```yaml
apiVersion: v1
kind: Secret
metadata:
    name: wordpress-admin-password
data:
    key: {{ .Values.passwordEncoded }}
```

```yaml
# values.yaml
image: wordpress:4.0-apache
storage: 200Gi
passwordEncoded: CsjhWVUxSdz12Qzg
```

*Nota*: i template dei manifesti ed il file `values.yaml` vengono assieme detti *Chart*. Inoltre, il Chart contiene anche un file `Chart.yaml` con alcune informazioni sul Chart stesso, come versione, nome, autori ecc.

### Download dei Chart

[Artifact Hub](https://artifacthub.io) è una repository online per gli Helm Chart. È possibile cercare in questo sito alcuni Chart per deployare facilmente le applicazioni. È possibile navigare nella repository anche da riga di comando:

```shell
helm search hub <nome-chart> # es: wordpress
```

Vi sono altre repository online, che possono essere aggiunte ad Helm tramite:

```shell
helm repo add <nome> <url>
# esempio
helm repo add bitnami https://charts.bitnami.com/bitnami
# per elencare tutte le repo installate
helm repo list
# per cercare nelle repo al posto dell'hub
helm search repo <nome-chart>
```

Per effettuare l'installazione di un certo Chart:

```shell
helm install <nome-release> <nome-chart>
# esempio:
helm install release-1 wordpress # il nome release è scelto dall'utente
```

*Nota*: è possibile installare più release nello stesso cluster. Ogni release sarà indipendente dalle altre.

È possibile anche installare un Chart da file locali:

```shell
# scaricare un chart senza installarlo
helm pull --untar <nome-repo>/<nome-chart>
# vedere i file scaricati
ls <nome-chart>
# installare il chart
helm install <nome-release> ./<nome-chart>
```

Altri comandi utili:

```shell
# elenco dei pacchetti installati
helm list
# disinstallare una release
helm uninstall <nome-release>
```

1. [CKA Course - Other Topics](https://github.com/kodekloudhub/certified-kubernetes-administrator-course/tree/master/docs/13-Other-Topics)
2. [Helm](https://helm.sh)
