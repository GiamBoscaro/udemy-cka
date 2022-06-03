# APPLICATION LIFECYCLE MANAGEMENT

## Rollouts

Ogni volta che un Deployment viene applicato al cluster (`kubectl apply`), viene creata una revision del deployment per tenere traccia della modifica effettuata. Questo permette di avere sia uno storico di tutte le modifiche del deployment, sia di poter fare un rollback molto facilmente nel caso di fossero problemi.

Per vedere lo stato di un rollout in esecuzione:

```shell
kubectl rollout status deployment/<nome-deployment>
```

Questo comando mostrerà tutti i Pod e le repliche che sono state già aggiornate e quelle in attesa di aggiornamento.
Per vedere invece lo storico di tutte le revisioni effettuate ad un deployment:

```shell
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

```shell
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

```shell
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

## Variabili d'Ambiente

Come su un `docker-compose`, è possibile importare le variabili d'ambiente di un container. Ci sono tre modi diversi per specificare le variabili:

* In modo diretto
* tramite Config Map
* tramite Secrets

Per impostare le variabili d'ambiente in un Pod, utilizzare il campo `env`:

```yaml
# pod-definition.yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp-color
spec:
 containers:
 - name: simple-webapp-color
   image: simple-webapp-color
   ports:
   - containerPort: 8080
   env:
   - name: APP_COLOR
     value: pink
```

Per importare la variabile da una Config Map utilizzare:

```yaml
env:
- name: APP_COLOR
  valueFrom:
    configMapKeyRef:
      name: <nome_config_map>
      key: <chiave_variabile> # nome della variabile nella config map
```

oppure da un Secret

```yaml
env:
- name: APP_COLOR
  valueFrom:
    secretKeyRef: <...>
```

## Config Map

Una Config Map permette di raccogliere in un manifest molte variabili d'ambiente condivise tra vari Pod. Aggiornando la Config Map, tutti i Pod interessati riceveranno i nuovi valori delle variabili.

Per creare una Config Map in modo imperativo:

```shell
kubectl create configmap <nome-config> --from-literal=KEY=value \       
  --from-literal=KEY2=value2
```

per ogni variabile d'ambiente, è necessario specificare l'opzione `--from-literal`. Questo diventa complesso per una mappa con molte variabili, per cui è anche possibile leggere le varibili da un file `.env` e simili:

```shell
kubectl create configmap <nome-config> --from-file=path/to/.env
```

### Deploy di una Config Map da manifest YAML

È possibile anche creare un manifest *yaml* da cui creare la Config Map:

```yaml
# config-definition.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_COLOR: blue
  APP_MODE: prod
  # ...
```

### Iniettare i valori di una Config Map su un Pod

Per iniettare tutti i valori presenti nella Config Map in un Pod:

```yaml
spec:
  envFrom:
  - configMapRef:
      name: app-config # nome della config map
```

*Nota*: le variabili saranno iniettate con lo stesso nome con cui sono presenti dentro la Config Map.

Per iniettare una singola variabile:

```yaml
env:
- name: APP_COLOR # mome della variabile nel Pod
  valueFrom:
    configMapKeyRef:
      name: app-config
      key: APP_COLOR # nome della variabile nella config map
```

Questo procedimento permette anche di cambiare nome alla variabile dentro il Pod. È anche possibile iniettare il file delle variabili d'ambiente direttamente da un volume:

```yaml
volumes:
  - name: app-config-volume
    configMap:
      name: app-config
```

## Secrets

Un Secret è una Config Map utilizzata per salvare variabili sensibili, come password e segreti. Come per le Config Map, è possibile creare un Secret da CLI elencando tutte le variabili al suo interno:

```shell
kubectl create secret generic <nome-secret> --from-literal=KEY=value \       
  --from-literal=KEY2=value2
```

oppure leggendole da un file:

```shell
kubectl create secret generic <nome-secret> --from-file=path/to/.env
```

### Deploy di un Secret da manifest YAML

Se si genera un Secret da un file *yaml* è necessario indicare i valori delle variabili in `base64`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
data:
  DB_Host: bX1zcWw=
  DB_User: cm9vdA==
  DB_Password: cGFzd3Jk
```

per codificare in `base64`  una stringa:

```shell
echo -n 'password' | base64
# per decodificare
echo -n 'password' | base64 --decode
```

### Iniettare i valori di una Secret su un Pod

Per iniettare tutti i valori presenti nel Secret in un Pod:

```yaml
spec:
  envFrom:
  - secretRef:
      name: app-secret # nome del secret
```

*Nota*: le variabili saranno iniettate con lo stesso nome con cui sono presenti dentro il Secret.

Per iniettare una singola variabile:

```yaml
env:
- name: APP_COLOR # mome della variabile nel Pod
  valueFrom:
    secretKeyRef:
      name: app-secret
      key: APP_COLOR # nome della variabile nel secret
```

Questo procedimento permette anche di cambiare nome alla variabile dentro il Pod. È anche possibile iniettare il file delle variabili d'ambiente direttamente da un volume:

```yaml
volumes:
  - name: app-secret-volume
    secret:
      name: app-secret
```

*Nota*: quando viene utilizzato un volume per salvare i secret, all'interno del Pod viene creato un volume con al suo interno un file per ogni chiave presente nel Secret. I valori delle chiavi sono contenuti __in chiaro__ dentro questi file.

### Best Practices per la Sicurezza

I Secret, anche se codificati in `base64`, non sono molto più sicuri di una Config Map. Kubernetes però gestisce i Secret in modo diverso che le Config Map, ad esempio:

* Il valore del Secret viene inviato al nodo solo nel momento in cui viene effettivamente richiesto da un Pod nel nodo.
* `kubelet` salva i valori dei Secret che sono arrivati al nodo in un file system temporaneo (*tmpfs*) e non sul disco rigido.
* Quando un Pod viene distrutto, i segreti che non sono più utilizzati vengono cancellati dal nodo.

Per rendere più sicuro il cluster, è possibile seguire queste linee guida:

* Non caricare i manifest dei Secret nelle repository (a differenza delle ConfigMap che possono essere caricate tranquillamente).
* Abilitare *Encryption at Rest*. Questo permette di salvare i segreti nell'`etcd` criptati (Vedere documentazione [Encrypting Secret Data at Rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)).
* Utilizzare soluzioni alternative ai Secret come *Helm Secrets* o *HashiCorp Vault*.

## Pods Multi Container

Certe volte è utile avere più container nello stesso Pod, che possano comunicare facilmente e che vengano sempre creati e distrutti assieme. I motivi per avere più container nello stesso Pod sono tra i più vari, ma vi sono tre Design Pattern utilizzati principalmente:

* *Side Car*: un container per il servizio e uno o più container *helper* che hanno senso di esistere solo quando il servizio primario è attivo. Gli *helper* potrebbero essere dei servizi di loggin o di sincronizzazione. I container funzionano in parallelo.
* *Adapter*: Un servizio primario e un container adattatore, per standardizzare o convertire i dati in input e output del Pod. Utile anche per dei servizi che fanno aggregazioni. I container funzionano in seriale, tutti i dati passano dall'adapter. Potrebbe ad esempio essere utile per convertire i log prodotti dal container in un formato generico compatibile con il servizio centralizzato di logging.
* *Ambassador*: un container del servizio primario e un container *ambassador*. L'ambasciatore funziona come un proxy e permette al servizio primario di connettersi a diversi endpoint in funzione dell'ambiente in cui si sta operando. Per cui, nel servizio primario l'endpoint è sempre lo stesso, ed è il container ambasciatore che cambia il puntamento in funzione dell'ambiente. I servizi funzionano in seriale. Tutti i dati passano dall'ambasciatore.

## Init Containers

Tutti i container all'interno di un Pod devono rimanere attivi per tutto il tempo oppure il Pod cercherà di restartarli continuamente. Ci sono però alcuni casi in cui un container potrebbe dover funzionare solo fino al completamento di un attività, oppure essere eseguito ad intervalli di tempo.

Per poter fare cioò vengono utilizzati gli Init Containers, cioè dei container che il Pod esegue prima del processo principale.

Possono essere uno o più, e verranno eseguiti in ordine. Tutti gli Init Containers devono completare con successo la loro attività prima che il container primario venga eseguito. Se un Init Container fallisce, il Pod viene restartato finchè tutti i processi di inizializzazione non vengono completati.

Per configurare degli Init Containers in un Pod, aggiunger nelle specifiche i container nel campo `initContainers`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: busybox:1.28
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
  initContainers:
  - name: init-myservice
    image: busybox
    command: ['sh', '-c', 'git clone <some-repository-that-will-be-used-by-application> ; done;']
```

## Applicazioni Self-Healing

Il Replication Controller permette di tenere sotto controllo lo stato dei Pod e di ricrearli qualora crashassero. Lo stato del Pod può essere visto con `kubectl describe`. Con questo comando si possono anche vedere le *Pod Conditions*, un array di booleani che indica con precisione quali fasi dell'esecuzione del Pod sono state superate e quali sono in errore:

* *PodScheduled*: il Pod è stato schedulato in un nodo.
* *Initialized*: il nodo ha iniziato a deployare il Pod.
* *ContainersReady*: i container all'interno del Pod sono pronti.
* *Ready*: il Pod è pronto. È lo stesso flag che si vede con `kubectl get pods`.

### Readiness Probe

Le applicazioni in esecuzione all'interno del Pod sono tra le più varie. Per questo è difficile automatizzare la verifica dello status di *Ready* in tutti i casi possibili. Lo sviluppatore dell'applicazione conosce con precisione il processo di avvio dell'applicazione e quando questa sarà pronta. È possibile impostare un *Readiness Probe* che continua a testare l'applicazione all'interno del Pod finchè non si ha la risposta voluta, che indica che l'applicazione è pronta.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: webapp
    image: webapp
    ports:
      - containerPort: 8080
    readinessProbe:
      httpGet:
        path: /api/ready
        port: 8080
      # dopo quanti secondi iniziare a testare col readinessProbe
      initialDelaySeconds: 10
      # ogni quanto eseguire il readinessProbe
      periodSeconds: 5
      # dopo quanti tentativi il Pod viene considerato in errore
      failureThreshold: 8
```

Finchè il `readinessProbe` non darà una risposta positiva, il Servizio associato al Pod non inoltrerà alcuna richiesta ad esso. Questo è particolarmente importante in un deployment con più repliche. Se il deployment viene scalato e vengono aggiunte repliche in fase di avviamento, il servizio inizierà ad inviare chiamate ai nuovi Pod subito se nessun `readinessProbe` è impostato. Per cui alcuni utenti potrebbero avere problemi nell'utilizzo dell'applicazione.

* *Nota*: se non viene impostato alcun `readinessProbe`, Kubernetes segnerà come Ready il Pod appena tutti i container al suo interno saranno in esecuzione.

Vi sono diversi modi per testare lo stato del Pod:

```yaml
# GET ad un API di test
httpGet:
  path: /api/ready
  port: 8080
# Connessione ad un socket TCP
tcpSocket:
  port: 3306
  # Esecuzione di un comando o script all'interno del container
exec:
  command: 
  - sh
  - /app/is_ready.sh
```

### Liveness Probe

Quando un container all'interno del Pod crasha, lo stato del Pod passa ad errore, il Pod viene riavviato o distrutto. È possibile però che il container rimanga in esecuzione anche se l'applicazione che sta eseguendo non sta effettivamente funzionando (come ad esempio un loop infinito). È necessario quindi impostare un *Liveness Probe* per testare continuamente la salute dell'applicazione durante l'esecuzione.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: webapp
    image: webapp
    ports:
      - containerPort: 8080
    livenessProbe:
      httpGet:
        path: /api/ready
        port: 8080
      # dopo quanti secondi iniziare a testare col readinessProbe
      initialDelaySeconds: 10
      # ogni quanto eseguire il readinessProbe
      periodSeconds: 5
      # dopo quanti tentativi il Pod viene considerato in errore
      failureThreshold: 8
```

La configurazione del `livenessProbe` è uguale a quella del `readinessProbe`, è sono possibili gli stessi tipi di test (chiamata API, connessione ad un socket, esecuzione di comandi).

## References

1. [CKA Course - Application Lifecycle Management](https://github.com/kodekloudhub/certified-kubernetes-administrator-course/tree/master/docs/05-Application-Lifecycle-Management)
2. [Encrypting Secret Data at Rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)
3. [Configure Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
