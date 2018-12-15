---
layout: post
title:  "JPA & Locking"
---
Ciao a tutti,

in quest' articolo voglio parlere di come utilizzare in JPA le sue API per gestire locking.

JPA ha 2 modalità di gestire i lock:

1. Optimistic Lock,
2. Pessimistic Lock.

## Optimistc Lock
Gli "optimistic lock" sono lock nel persistence context di JPA.
Non toccano minimanente i lock del database sotto stante.
Tali lock agiscono su Entity JPA che hanno un field marchiato con l' annotation @Version.

### Field Version
Per attivare un optimistic lock, è necessario che l' Entity in questione abbia un field versionato tramite l' annotation @Version
Di seguito un esempio di entity:

{% highlight java %}
@Entity
@Table(name = "MYTABLE")
@Audited
public class MyTable implements Serializable {
    private static final long serialVersionUID = 1L;

    @Id
    @Column(name = "MYTABLE_K")
    private Long mytableK;

    @Version
    @Column(name = "VERSION")
    private Long version;

    public Long getMytableK() {
        return mytableK;
    }

    public void setMytableK(Long mytableK) {
        this.mytableK = mytableK;
    }

    public Long getVersion() {
        return version;
    }

    public void setVersion(Long version) {
        this.version = version;
    }
}
{% endhighlight %}

Questo il relativo script sql:

{% highlight sql %}
--------------------------------------------------------
--  mytable
--------------------------------------------------------
create table mytable
(
  mytable_k number not null,
  version not null
);

alter table mytable
add constraint mytable_pk
primary key(mytable_k) using index;
--------------------------------------------------------
{% endhighlight %}

Ogni entity puo' avere al piu 1 solo field marchiato con @Version.
Tale field puo' essere di uno dei seguenti tipi: int, Integer, long, Long, short, Short, java.sql.Timestamp.

NB. Tale field non lo dobbiamo MAI inserire/aggiornare a mano, ma sarà JPA a gestirlo!

### Tipologie di Optimistic Lock
JPA ci fornisce tali tipologie di Optimistic Lock:

- OPTIMISTIC: tipologia base di Optimistic Lock,
- OPTIMISTIC_FORCE_INCREMENT: come OPTIMISTIC, ma incrementa il @Version, anche se l' entity non è cambiata,
- READ: alias di OPTIMISTIC, deprecato,
- WRITE: alias di OPTIMISTIC_FORCE_INCREMENT, deprecato.

### Come usare l' Optimistic Lock
#### Find
{% highlight java %}
entityManager.find(MyTable.class, myTableId, LockModeType.OPTIMISTIC);
{% endhighlight %}

#### Query
Di seguito un esempio con CriteriaQuery:

{% highlight java %}
public MyTable search(final Long myTableK) {
    final CriteriaBuilder cb = getCriteriaBuilder();
    final CriteriaQuery<MyTable> query = cb.createQuery(MyTable.class);
    final Root<MyTable> from = query.from(MyTable.class);
    query.select(from);
    query.where(cb.equal(from.get(MyTable_.myTableK), myTableK));
    final TypedQuery<MyTable> typedQuery = entityManager.createQuery(query);
    typedQuery.setLockMode(LockModeType.OPTIMISTIC);
    return typedQuery.getSingleResult();
}
{% endhighlight %}

#### Locking esplicito
{% highlight java %}
final MyTable myTable = entityManager.find(MyTable.class, myTableId);
entityManager.lock(myTable, LockModeType.OPTIMISTIC);
{% endhighlight %}

#### Refresh
{% highlight java %}
final MyTable myTable = entityManager.find(MyTable.class, myTableId);
entityManager.refresh(myTable, LockModeType.OPTIMISTIC);
{% endhighlight %}

#### Named Query
{% highlight java %}
@NamedQuery(name="myTableById",
  query="SELECT t FROM MyTable t WHERE t.myTableK = :myTableK",
  lockMode = LockModeType.OPTIMISTIC)
{% endhighlight %}

### Come funziona l' Optimistic Lock
In linea generale, il funzionamento dell' optimistic lock è molto semplice:

Nella transazione T1:
- viene fetchata l' Entity, e quindi JPA legge il valore del field marchiato con @Version
- viene aggiornato almeno un field dell' Entity
- viene committata la transazione

Quando la transazione viene committata (o quando si fa flush()), in un' unica istruzione, JPA verifica se il field @Version sul database metcha con il field @Version letto al punto (1).
Se metcha, nessun' altro ha aggiornato tale Entity prima di noi.
Altrimenti viene sollevata OptimisticLockException, stando a significare che qualcun' altro in concorrenza ha aggiornato l' Entity prima di noi.

NB. Dopo un po di prove effettuate, ho notato che se i field dell' Enitty non vengono realmente cambiati, non viene verificato il @Version, nè viene aggiornato.
Se si vuole comunque verificare ed aggiornare il field @Version, utilizzare la tipologia di Optimistic Lock: OPTIMISTIC_FORCE_INCREMENT,


## Pessimistic Lock
A differenza dei precedenti, i "pessimistic lock" sono lock che agiscono direttamente sul database sottostante.
Questo vuol dire che se lockiamo con Pessimistic Lock una riga sul database, chiunque accede al database a tale riga, utilizzando qualsiasi altro client per connettersi al database, vedrà la riga lockata.
Inoltre non hanno bisogno che l' Entity abbia il field marchiato con @Version.

### Tipologie di Pessimistic Lock
Esitono 2 tipologie di Pessimistic Lock:
- Shared Lock: lock di lettura,
- Exclusive Lock: lock di scrittura.
Gli Shared Lock, se esistono, permettono altri Shared Lock in concorrenza sulla stessa riga sul database (n thread in parallelo possono leggere la stessa riga).
Mentre non permettono l' acquisizione di un Exclusive Lock in concorrenza (se n thread stanno leggendo, nessuno puo scrivere).

Gli Exclusive Lock, invece, se esistono, bloccano l' acquisizione di altri Shared ed Exclusive Lock in concorrenza (se un thread sta scrivenno, nessun' altro puo leggere nè scrivere).

JPA ci fornisce tali tipologie di Pessimistic Lock:

- PESSIMISTIC_READ: Shared Lock
- PESSIMISTIC_WRITE: Exclusive Lock
- PESSIMISTIC_FORCE_INCREMENT: come PESSIMISTIC_WRITE, ma forzano JPA ad incrementare il field @Version nell' Entity (quindi questa tipologia necessita del field @Version)

Uno Shared Lock, per poter scrivere, deve essere convertito in Exclusive Lock.

### Eccezioni
- PessimisticLockException: viene sollevata se l' acquisizione del Pessimistic Lock, o il convertire uno Shared in Exclusive, ha fallito
- LockTimeoutException: viene sollevata se l' acquisizione del Pessimistic Lock è andata in timeout.

### Come usare il Pessimistic Lock
#### Find
{% highlight java %}
entityManager.find(MyTable.class, myTableId, LockModeType.PESSIMISTIC_READ);
{% endhighlight %}

#### Query
Di seguito un esempio con CriteriaQuery:

{% highlight java %}
public MyTable search(final Long myTableK) {
    final CriteriaBuilder cb = getCriteriaBuilder();
    final CriteriaQuery<MyTable> query = cb.createQuery(MyTable.class);
    final Root<MyTable> from = query.from(MyTable.class);
    query.select(from);
    query.where(cb.equal(from.get(MyTable_.myTableK), myTableK));
    final TypedQuery<MyTable> typedQuery = entityManager.createQuery(query);
    typedQuery.setLockMode(LockModeType.PESSIMISTIC_READ);
    return typedQuery.getSingleResult();
}
{% endhighlight %}

#### Locking esplicito
{% highlight java %}
final MyTable myTable = entityManager.find(MyTable.class, myTableId);
entityManager.lock(myTable, LockModeType.PESSIMISTIC_READ);
{% endhighlight %}

#### Refresh
{% highlight java %}
final MyTable myTable = entityManager.find(MyTable.class, myTableId);
entityManager.refresh(myTable, LockModeType.PESSIMISTIC_READ);
{% endhighlight %}

#### Named Query
{% highlight java %}
@NamedQuery(name="myTableById",
  query="SELECT t FROM MyTable t WHERE t.myTableK = :myTableK",
  lockMode = LockModeType.PESSIMISTIC_READ)
{% endhighlight %}

### Pessimistic Lock Timeout
È possibile attendere l' acquisizione del Pessimistc Lock entro un timeout:
{% highlight java %}
final Map<String, Object> properties = new HashMap<>(); 
properties.put("javax.persistence.lock.timeout", 1000L); 
 
entityManager.find(MyTable.class, myTableId, LockModeType.PESSIMISTIC_READ, properties);
{% endhighlight %}

Se entro il timeout non viene acquisito il Pessimistic Lock, viene sollevata LockTimeoutException.

## Note
Dopo varie prove prove ho raccolto le seguenti note:
1. Optimistic Lock o Pessimistic Lock non lockano le classiche find() o query che non specificano di voler acquisire il lock.
2. Entrambi i tipi di lock vengono rilasciati quando la transazione viene committata o rollbackata.
3. Se usate come database Oracle, c' è un problema sui Pessimistic Lock: Oracle traduce sia la PESSIMISTIC_READ che la PESSIMISTIC_WRITE con una SELECT .. FOR UPDATE. Questo vuol dire che la PESSIMISTIC_READ, in Oracle, viene gestita esattamente come una PESSIMISTC_WRITE! Infatti la specifica JPA dice che il vendor puo decidere di fare questa cosa.
4. Se si ha la necessita di lockare un entity sul database, e tale lock dev' essere "condiviso" in N chiamate HTTP diverse:
 - bisogna salvare in una tabella il fatto che esiste tale lock in atto
 - tutte le altre chiamate vanno a verificare se esiste tale "lock logico" sul database
 - chi scrive tale lock logico:
  - Apre una transazione T1
  - Accede a tale lock logico tramite PESSIMISTIC_WRITE, con un timeout impostato ad esempio a 0.
  - Se l' acquisizione va in timeout, vuol dire che qualcuno sta scrivendo in concorrenza con me, e gestisco un' eventuale Eccezione.
  - Se gia esiste la riga, gestire un eventuale Eccezione
  - Altrimenti persistere tale lock logico
  - Committare T1 (in questo modo si rilascia il PESSIMISTIC_WRITE)
5. nel caso in cui le chiamate che lockano scattano in asincrono, bisogna cercare di acquisire il lock logico in un metodo sincrono, in modo da poter dare subito un feedback al chiamante. Dopodichè dal metodo sincrono chiamiamo il metodo asincrono, ed eliminare il lock logico solo alla fine del metodo asincrono.