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

### Rollback

Quando un aggiornamento viene applicato, un nuovo Replica Set viene creato. Man mano che il rollout prosegue, i nuovi Pod vengono creati nel nuovo Replica Set e contemporaneamente vengono distrutti in quello vecchio. Alla fine del processo si avranno due Replica Set, uno con zero repliche e uno con tutte le repliche richieste.

Se viene richiesto un rollback, il nuovo Replica Set viene scalato a zero replice e quello vecchio torna in funzione con le n replichè desiderate.

Per effettuare un rollout:

```bash
kubectl rollout undo deployment/<nome-deployment>
```

### Comandi

## References

1. [CKA Course - Application Lifecycle Management](https://github.com/kodekloudhub/certified-kubernetes-administrator-course/tree/master/docs/05-Application-Lifecycle-Management)
