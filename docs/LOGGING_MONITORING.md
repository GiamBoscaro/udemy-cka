# LOGGING AND MONITORING

## Monitorare il Cluster

Al momento Kubernetes non ha un sistema integrato e completo di monitoraggio dello stato del cluster e delle risorse, ma esistono molte alternative open source, tra cui:

* Heapster (deprecato)
* Metrics Server
* Prometheus
* Elastic Stack
* DataDog
* Dynatrace

Il servizio più utilizzato al momento *Metrics Server*.

### Installazione di Metrics Server

Metrics Server raccoglie e aggrega le metriche provenienti dai Pod nel cluster. I dati sono salvati solo in RAM, quindi non è possibile avere uno storico completo dello stato del sistema.
Un modulo di `kubelet` chiamato `cAdvisor` raccoglie le metriche dei Pod in un nodo e le mette a disposizione di Metric Server.

*Nota*: Può esserci un solo server attivo in ogni cluster.

Per deployare Metrics Server, utilizzare i manifest presenti nella sua repository:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

*Nota*: nel caso di `minikube` è possibile deployare facilmente Metrics Server col comando `minikube addons enable metrics-server`.

### Utilizzo di Metrics Server

Metrics Server necessita di un pò di tempo per raccogliere alcune metriche dai nodi. In condizioni normali, è possibile vedere lo stato delle risorse dei nodi e pod tramite:

```bash
kubectl top node
kubectl top pod
```

## Logs delle Applicazioni

I logs possono essere visualizzati in modo molto simile a Docker. Per visualizzare i log che un container produce dentro un Pod, utilzzare:

```bash
kubectl logs -f <pod-name> <container-name>
```

Il nome del container è opzionale se dentro il Pod è presente un solo container.

## References

1. [CKA Course - Logging and Monitoring](https://github.com/kodekloudhub/certified-kubernetes-administrator-course/tree/master/docs/04-Logging-and-Monitoring)