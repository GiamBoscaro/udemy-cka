# OTHER TOPICS

## JSON PATH

Kubectl comunica con API Server per ricevere le informazioni richieste. API Server fornisce le informazioni in formato JSON, ma non tutto viene visualizzato nel terminale, neanche con l'opzione `-o wide`.
Per vedere tutti i dati forniti da API Server, mostrare i risultati in formato *yaml* o *json*:

```bash
kubectl get nodes -o json
```

Osservando la struttura dei dati, è possibile ricavare informazioni ben precise utilizzando una query jsonpath:

```bash
kubectl get pods -o=jsonpath=<query>
```

Alcuni esempi di query:

```bash
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

```bash
# Nome nodo          Numero CPU del nodo
'{range .items[*]}
{.metadata.name}{'\t'}{.status.capacity.cpu}{'\n'}
{end}'
```

Quindi la query da inserire in `kubectl`:

```bash
kubectl get pods -o=jsonpath='{range .items[*]}{.metadata.name}{'\t'}{.status.capacity.cpu}{'\n'}{end}'
```

### Colonne Personalizzate

```bash
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

```bash
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

```bash
kubectl config view --kubeconfig=my-kube-config \
-o jsonpath="{.contexts[?(@.context.user=='aws-user')].name}"
```

1. [CKA Course - Other Topics](https://github.com/kodekloudhub/certified-kubernetes-administrator-course/tree/master/docs/13-Other-Topics)
