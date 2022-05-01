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

```bash
curl --header "Content-Type: application/json" --request POST --data '{"apiVersion":"v1", "kind":"Binding", ...}' http://$SERVER/api/v1/namespaces/default/pods/$PODNAME/binding
```

## Label, Selectors e Annotations

È possibile assegnare qualsiasi label agli oggetti di Kubernetes, in modo da porteli categorizzare, filtrare e selezionare tramite i selector successivamente. I labels vanno inseriti nel tag `metadata` dell'oggetto.

Per selezionare degli oggetti in funzione dei `labels`:

```bash
kubectl get pods --selector app=my-app
```

Per contare velocemente tutti gli oggetti con un certo label:

```bash
kubectl get pods -l app=my-app --no-headers | wc -l
```

Un Service o ReplicaSet utilizza i campi `selector` e `matchLabels` per selezionare i Pod che deve gestire. Il Service o ReplicaSet prenderà in carico i Pod che contengono __tutti__ i `labels` presenti in `matchLabels`. I Pod, viceversa, possono avere anche più `labels` diversi, ma per essere presi in carico devono avere almeno i `labels` che il servizio sta cercando.

Le `annotations` sono altri metadati che vengono usati solo a scopo informativo per inserire alcune informazioni nella definizione dell'oggetto. Molte volte vengono create da Kubernetes durante l'esecuzione.

## References

1. [CKA Course - Scheduling](https://github.com/kodekloudhub/certified-kubernetes-administrator-course/tree/master/docs/03-Scheduling)
