# TROUBLESHOOTING

## Problemi con l'Applicazione Deployata

Se il cluster è in funzione ma l'applicazione non sta funzionando, vi sono varie punti da controllare per scoprire dov'è il problema:

* Controllare la connessione con il server e lo stato della rete (es: con `curl`).
* Controllare i Service che espongono il Pod:

```bash
kubectl describe service <nome-servizio>
# verificare che i selettori siano corretti
# verificare che il servizio stia puntando ad almeno un Endpoint
```

* Controllare lo stato del Pod che esegue l'applicazione:

```bash
kubectl get pod # verificare che lo stato sia in Running, e quanti restart ci sono stati
kubectl describe pod <nome-pod> # verificare la cronologia degli stati
kubectl logs <nome-pod> -f # verificare i log attuali dell'applicazione
kubectl logs <nome-pod> -f --previous # verificare i log dei Pod precedenti
```

## Problemi con il Control Plane

Per debuggare i problemi al Control Plane, cominciare col verificare il suo corretto funzionamento e lo stato dei nodi e dei Pod nel cluster:

```bash
kubectl get nodes
kubectl get pods
```

In particolare i Pod che ospitano i servizi del Master Node:

```bash
kubectl get pods -n kube-system
kubectl logs kube-apiserver-master -n kube-system
```

Se i servizi sono invece installati direttamente nell'host:

```bash
service <nome-servizio> status
sudo journalctl -u <nome-servizio>
# kube-apiserver, kube-controller-manager, kube-scheduler, kubelet, kube-proxy
```

## Problemi con i Worker Nodes

Per verificare i problemi dei nodi, controllare innanzitutto lo stato di essi:

```bash
kubectl get nodes
kubectl describe node <nome-nodo>
```

`describe node` permette di vedere chiaramente molte flag che aiutano a comprendere il problema (poca memoria rimasta, spazio insufficiente ecc.).
È possibile poi entrare nel nodo e verificare lo stato del sistema con vari comandi

```bash
top # verificare le risorse, memoria e cpu
df -h # verificare lo storage e i dischi rigidi
service kubelet status # verificare lo stato del kubelet (service kubelet start per avviarlo)
sudo journalctl -u kubelet -f # verificare i log del kubelet
openssl x509 -in /var/lib/kubelet/<nome-nodo>.crt -text # verificare la correttezza dei certificati
```

## Problemi di Rete

I principali servizi che gestiscono il network di Kubernetes sono: CoreDNS, Kube Proxy e i plugin CNI.

### Problemi con CoreDNS

CoreDNS consuma più risorse più Pod ci sono nel cluster, e anche più storage in funzione della cache che utilizza.

* CoreDNS in Pending:

  * Verificare che un plugin di rete sia installato.

* CoreDNS in Error o CrashLoopBackOff

  * Verificare che la versione di Docker non sia troppo vecchia, e nel caso aggiornarla.
  * Utilizzare l'opzione `resolvConf` per indicare un diverso file `resolv.conf`.
  * Disabilitare la cache e ripristinare `/etc/resolv.conf`.
  
* CoreDNS in Running:

  * Verificare che `kube-dns` abbia degli endpoint validi:

```bash
kubectl -n kube-system get ep kube-dns
```

### Problemi con Kube Proxy

* Verificare che il Pod `kube-proxy` sia in esecuzione.
* Verificare i logs di `kube-proxy`.
* Verficiare la ConfigMap e il file di configurazione.
* Verificare che `kube-proxy` sia in esecuzione all'interno del Pod, nel container.

1. [CKA Course - Troubleshooting](https://github.com/kodekloudhub/certified-kubernetes-administrator-course/tree/master/docs/12-Troubleshooting)
2. [Debugging Services Issues](https://kubernetes.io/docs/tasks/debug-application-cluster/debug-service/)
3. [DNS Troubleshooting](https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/)
