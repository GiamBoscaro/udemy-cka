# APPLICATION LIFECYCLE MANAGEMENT

## Rollouts

Ogni volta che un Deployment viene applicato al cluster (`kubectl apply`), viene creata una revision del deployment per tenere traccia della modifica effettuata. Questo permette di avere sia uno storico di tutte le modifiche del deployment, sia di poter fare un rollback molto facilmente nel caso di fossero problemi.

Per vedere lo stato di un rollout in esecuzione:

```bash
kubectl rollout status deployment/<nome-deployment>
```

Questo comando mostrerà tutti i Pod e le repliche che sono state già aggiornate e quelle in attesa di aggiornamento.
Per vedere invece lo storico di tutte le revisioni effettuate ad un deployment:

```bash
kubectl rollout history deployment/<nome-deployment>
```

### Strategia di Rollout

Esistono due strategie per il rollout di un aggiornamento:

* *Recreate*: tutti i Pod vengono cancellati e ricreati. Questo causa downtime.
* *Rolling Update*: i Pod vengono distrutti e ricreati uno alla volta. Non c'è downtime ma ci sarà un lasso di tempo in cui sono presenti nel cluster Pod con diverse caratteristiche. Questa è la strategia di default.

Nel file *yaml*, la strategia di rollout del Pod viene specificata nel campo `strategy`:

```yaml
# deployment-definition.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
 name: myapp-deployment
 labels:
  app: nginx
spec:
 strategy:
   type: Recreate
 template:
   metadata:
     name: myapp-pod
     labels:
       app: myapp
       type: front-end
   spec:
    containers:
    - name: nginx-container
      image: nginx:1.7.1
 replicas: 3
 selector:
  matchLabels:
    type: front-end       
```

### Rollback

Quando un aggiornamento viene applicato, un nuovo Replica Set viene creato. Man mano che il rollout prosegue, i nuovi Pod vengono creati nel nuovo Replica Set e contemporaneamente vengono distrutti in quello vecchio. Alla fine del processo si avranno due Replica Set, uno con zero repliche e uno con tutte le repliche richieste.

Se viene richiesto un rollback, il nuovo Replica Set viene scalato a zero replice e quello vecchio torna in funzione con le n replichè desiderate.

Per effettuare un rollout:

```bash
kubectl rollout undo deployment/<nome-deployment>
```

## Comandi e Argomenti

### Comandi e Argomenti in Docker

Quando si definisce un container con Docker, è possibile specificare un comando che viene eseguito all'esecuzione del container:

```dockerfile
FROM node:latest
# ...
CMD ["npm", "start"]
```

Se si vuole passare qualche argomento al comando, è possibile utilizzare l'`ENTRYPOINT`:

```dockerfile
FROM node:latest
# ...
ENTRYPOINT ["npm"]
# argomento di default se non viene passato alcun argomento
CMD ["start"]
```

Alla creazione del container è possibile indicare un argomento che verrà appeso all'`ENTRYPOINT`. Se non è presente alcun argomento, verrà utilizzato il `CMD` di default.

```bash
# esegue il default: npm start
docker run --name my-container
# esegue l'entrypoint npm con argomento debug: npm debug
docker run --name my-container debug
# sovrascrive l'entrypoint con il comando node ed esegue il nuovo
# entrypoint con l'argomento app.js: node app.js
docker run --name my-container --entrypoint node my-container app.js
```

### Comandi e Argomenti in Kubernetes

Si può avere un approccio equivalente in Kubernetes. In questo caso, l'`ENTRYPOINT` viene chiamato `command`, e gli argomenti `CMD` vengono chiamati `args`. Un manifesto di un Pod d'esempio può essere:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-sleeper-pod
spec:
 containers:
 - name: ubuntu-sleeper
   image: ubuntu-sleeper
   # sovrascrive l'entrypoint di default
   command: ["sleep2.0"]
   # aggiunge l'argomento al comando 
   # (sovrascrivendo quello di default)
   args: ["10"]
```

## References

1. [CKA Course - Application Lifecycle Management](https://github.com/kodekloudhub/certified-kubernetes-administrator-course/tree/master/docs/05-Application-Lifecycle-Management)
