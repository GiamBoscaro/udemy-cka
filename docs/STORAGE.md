# STORAGE

## Volumi

I Pod sono effimeri, vengono continuamente creati e distrutti. Questo significa che i dati generati al loro interno non sono permanenti ma vengono persi ogni volta che il Pod viene eliminato.
Se si vogliono mantenere i dati interni al Pod, è necessario utilizzare un *Volume*.

### Volume nel Pod

Per creare un volume che esiste solo finchè il Pod è in esecuzione, utilizzare `emptyDir`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - image: k8s.gcr.io/test-webserver
    name: webserver
    volumeMounts:
    - mountPath: /cache
      name: cache-volume
  volumes:
  - name: cache-volume
    emptyDir: {}
```

*Nota*: in caso di crash del Pod, il volume __non__ viene cancellato. Il volume viene rimosso solo quando il Pod viene rimosso.

### Volume nell'Host

Per salvare i dati nell'host che sta ospitando il Pod, utilizzare `hostPath`:

```yaml
apiVersion: v1
kind: Pod
metadata:
    name: my-pod
spec:
    containers:
    - image: alpine
      name: alpine
      command: ["/bin/sh", "-c"]
      # il risultato andrebbe perso senza volume, quando il Pod si spegne
      args: ["shuf -i 0-100 -n 1 >> /opt/number.out;"]
      # associa la cartella opt all'interno del Pod con il volume
      volumeMounts:
      - mountPath: /opt
        name: data-volume
    # creo un volume nell'host, nella cartella data
    volumes:
    - name: data-volume
      hostPath:
        path: /data
        type: Directory
```

Questa soluzione va bene se si ha un singolo nodo, ma se il Pod è replicato su più nodi, si avranno incongruenze nei dati chi Pod vedono. Il contenuto del volume __non__ viene condiviso tra nodi diversi.
Una soluzione per ovviare a questo problema è montare un volume esterno al nodo, condiviso tra tutti (come un Cloud Storage).

## Volumi Persistenti

Un normale volume è associato al Pod. Qualsiasi modifica deve quindi essere effettuata su tutti i Pod che utilizzano il volume. Uno sviluppatore che crea un nuovo Pod, dovrà ricordarsi anche di montare tutti i volumi necessari.
Un amministratore del cluster può però creare uno grande *Persistent Volume* condiviso in tutto il cluster, da cui ogni Pod potrà andarsi a riservare lo spazio necessario per operare:

```yaml
# pv-definition.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-vol1
spec:
  accessModes: [ "ReadWriteOnce" ]
  persistentVolumeReclaimPolicy: Recycle
  capacity:
    storage: 1Gi
  # tipo di volume
  hostPath:
    path: /tmp/data
```

Le possibili `accessModes` sono:

* `ReadOnlyMany`: vari Pod e Nodi possono montare il volume e leggerlo.
* `ReadWriteOnce`: solo un Nodo può montare il volume. I Pod all'interno di questo nodo hanno tutti accesso al volume.
* `ReadWriteMany`: vari Pod e Nodi possono montare il volume sia in lettura che in scrittura.
* `ReadWriteOncePod`: un unico Pod in tutto il cluster può montare il volume.

*Nota*: `hostPath` deve essere utilizzato solo per test, ma non è adatto ad un ambiente di produzione. È consigliato utilizzare un cloud storage o soluzioni simili.

*Nota*: __solo gli amministratori__ creano i Persistent Volumes. Gli sviluppatore potranno crearsi le loro Claims per utilizzare lo spazio all'interno del volume.

`persistentVolumeReclaimPolicy` indica cosa fare quando la claim associata al volume viene cancellata. Vi sono tre possibilità:

* `Retain`: la claim viene cancellata ma non il persistent volume. Dovrà essere cancellato manualmente.
* `Delete`: il volume viene cancellato assieme alla claim.
* `Recycle`: il volume rimane disponibile, ma viene resettato prima di essere messo a disposizione del cluster.

Per elencare tutti i persistent volume presenti:

```shell
kubectl get persistentvolume
```

## Claims dei Volumi Persistenti

Una volta creato il Persistent Volume, i Pod devono richiedere l'utilizzo dello spazio con una Persistent Volume Claim. Nella claim vanno elencate le caratteristiche del volume che si necessita, oltre a quanto spazio si vuole riservare:

```yaml
# pvc-definition.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  accessModes: [ "ReadWriteOnce" ]
  resources:
   # quanto spazio è richiesto
   requests:
     storage: 500Mi
```

Kubernetes cercherà di assegnare la claim al volume più vicino alle caratteristiche richieste. Se non viene trovato alcun volume, la claim rimarrano in *Pending* finche un nuovo Persistent Volume non verrà creato.

*Nota*: un volume può avere solo una claim assegnata a se stesso. Se non ci sono volumi disponibili con le risorse richieste, Kubernetes potrebbe assegnare la claim ad un volume con più risorse del necessario. Questo volume __non__ potrà essere assegnato ad altre claims e quindi lo spazio in più rimarrà inutilizzato.

Per elencare le claim presenti:

```shell
kubectl get persistentvolumeclaim
```

### Utilizzare una Claim nel Pod

Per utilizzare una claim in un Pod, utilizzare il tag `persistentVolumeClaim`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
    - name: myfrontend
      image: nginx
      volumeMounts:
      - mountPath: "/var/www/html"
        name: mypd
  volumes:
    - name: mypd
      persistentVolumeClaim:
        claimName: myclaim
```

## Classi di Storage

Quando viene creato un Persistent Volume, è necessario assegnarli uno Storage. Questo Storage deve già esistere prima di poter creare il volume. È possibile però utilizzare le Storage Class che crea dinamicamente Storage quando vengono richiesti dai volumi. La storage class viene configurata in base al provider. Ad esempio, se si utilizza Google Cloud Storage:

```yaml
# sc-definition.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
   name: google-storage
provisioner: kubernetes.io/gce-pd
parameters:
  # I parametri cambiano per ogni provider! Per Google:
  type: pd-standard # pd-standard, pd-ssd
  replication-type: none # none, regional-pd
```

A questo punto, __non servono più i Persistent Volumes__, in quanto la Storage Class lo creerà automaticamente quando una Claim lo richiede. Nella Claim va indicata la Storage Class:

```yaml
# pvc-definition.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  accessModes: [ "ReadWriteOnce" ]
  storageClassName: google-storage
  resources:
   # quanto spazio è richiesto
   requests:
     storage: 500Mi
```

Modificando i parametri della Storage Class è possibile creare vere e proprie classi di storage (come ad esempio una class veloce con SSD e una più lenta con HDD).

### Binding Mode

Quando viene creata una Storage Class, si può specificare la `volumeBindingMode`, che indica quando la Storage Class andrà a creare il volume effettivamente:

* `WaitForFirstConsumer`: quando viene eseguito il primo Pod che richiede un volume compatibile.
* `Immediate`: appena la classe viene creata.

1. [CKA Course - Storage](https://github.com/kodekloudhub/certified-kubernetes-administrator-course/tree/master/docs/08-Storage)
