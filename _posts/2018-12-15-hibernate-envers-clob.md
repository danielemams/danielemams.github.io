---
layout: post
title:  "Hibernate Envers & LOB"
---
Ciao a tutti,

scrivo quest’ articolo dopo una mia esperienza tra Hibernate Envers e tabelle con almeno una colonna di tipo LOB/CLOB/BLOB.

Nota. Per chi non conoscesse <a href="https://docs.jboss.org/envers/docs/" target="_blank">Hibernate Envers</a> o il tipo di dato <a href="https://docs.oracle.com/javadb/10.8.3.0/ref/rrefclob.html" target="_blank">CLOB</a>.

La cosa che si voleva evitare era che quando veniva fetchata la tabella, evitare di fetchare sempre e comunque anche il field di tipo CLOB, perche pesante.

In generale, in quest' articolo si parlerà di 2 argomenti distinti:

1. Fare in modo di mappare Entity JPA con field CLOB, evitando di fetchare sempre il field CLOB, perche pesante
2. Avere N tabelle correlate tra di loro tramite ForeignKeys, sotto Hibernate Envers, ed avere un meccanismo per avere una relazione tra le N revisioni delle N tabelle

Parto dicendo che la mia situazione a livello di database era di una tabella con una colonna di tipo CLOB.

<img src="/img/mams.png"/>

Di seguito lo script sql:

{% highlight sql %}
--------------------------------------------------------
--  mytable
--------------------------------------------------------
create table mytable
(
  mytable_k number not null,
  data CLOB null
);

alter table mytable
add constraint mytable_pk
primary key(mytable_k) using index;
--------------------------------------------------------
{% endhighlight %}

A livello di JPA, la relativa Entity era la seguente:

{% highlight java %}
@Entity
@Table(name = "MYTABLE")
@Audited
public class MyTable implements Serializable {
    private static final long serialVersionUID = 1L;

    @Id
    @Column(name = "MYTABLE_K")
    private Long mytableK;

    @Lob
    @Column(name = "DATA")
    private String data;

    public Long getMytableK() {
        return mytableK;
    }

    public void setMytableK(Long mytableK) {
        this.mytableK = mytableK;
    }

    public String getData() {
        return data;
    }

    public void setData(String data) {
        this.data = data;
    }
}
{% endhighlight %}

Il problema di un' Entity del genere, è che quando si fetcha l' Entity, viene fetchato sempre anche il CLOB. Cosa che vogliamo evitare.

Proviamo con una prima soluzione, che spiegero' successivamente perche' non funzionante. Il field CLOB lo mappiamo in questo modo:

{% highlight java %}
@Basic(fetch = FetchType.LAZY)
@Lob
@Column(name = "DATA")
private String data;
{% endhighlight %}

In questo modo si puo specificare a JPA di caricare il field in modo LAZY.

Purtroppo, pero', per la specifica JPA, LAZY:

<em>The <code>LAZY</code> strategy is a hint to the persistence provider runtime that data should be fetched lazily when it is first accessed. The implementation is permitted to eagerly fetch data for which the <code>LAZY</code> strategy hint has been specified.</em>

Ed infatti Hibernate non rispetta tale LAZY loading specificato sul field tramite @Basic().

Al che decido di strutturare il modello dati in maniera differente.

Praticamente creo una tabella secondaria (chiamata Extended), dove ci mettero' il CLOB. In questo modo avro' 2 Entity, e se fetcho la principale, non verra' fetchata la Extended.

<img src="/img/extended.png"/>

Di seguito lo script sql:

{% highlight sql %}
--------------------------------------------------------
--  mytable
--------------------------------------------------------
create table mytable
(
  mytable_k number not null
);

alter table mytable
add constraint mytable_pk
primary key(mytable_k) using index;

--------------------------------------------------------
--  mytable_extended
--------------------------------------------------------
create table mytable_extended
(
  mytable_fk number not null,
  data clob null
);

alter table mytable_extended
add constraint mytable_extended_pk
primary key(mytable_fk) using index;

alter table mytable_extended
add constraint mytableextended_mytable_fk
foreign key(mytable_fk)
references mytable(mytable_k);
--------------------------------------------------------
{% endhighlight %}

E queste le 2 relative Entity:

MyTable:

{% highlight java %}
@Entity
@Table(name = "MYTABLE")
@Audited
public class MyTable implements Serializable {
    private static final long serialVersionUID = 116273916315698924L;

    @Id
    @Column(name = "MYTABLE_K")
    private Long mytableK;
    
    public Long getMytableK() {
        return mytableK;
    }

    public void setMytableK(Long mytableK) {
        this.mytableK = mytableK;
    }
}
{% endhighlight %}

MyTableExtended:

{% highlight java %}
@Entity
@Table(name = "MYTABLE_EXTENDED")
@Audited
public class MyTableExtended implements Serializable {
    private static final long serialVersionUID = 1L;

    @Id
    @Column(name = "MYTABLE_FK", nullable = false, updatable = false)
    private Long myTableFk;

    @OneToOne(optional = false, cascade = CascadeType.REMOVE, orphanRemoval = true)
    @JoinColumn(name = "MYTABLE_FK", nullable = false)
    private MyTable myTable;

    @Lob
    @Column(name = "DATA")
    private String data;

    public Long getMyTableFk() {
        return myTableFk;
    }

    public void setMyTableFk(Long myTableFk) {
        this.myTableFk = myTableFk;
    }

    public MyTable getMyTable() {
        return myTable;
    }

    public void setMyTable(MyTable myTable) {
        this.myTable = myTable;
    }

    public String getData() {
        return data;
    }

    public void setData(String data) {
        this.data = data;
    }
}
{% endhighlight %}

Da notare che la relazione @OneToOne è unidirezionale, dall' Extended alla tabella padre. Questo perche, anche se si esplicita il caricamente del field OneToOne in modalità LAZY, anche qui Hibernate non rispetta tale modalità, e carica EAGER il field OneToOne. Ponendo tale relazione solo dall' Extended alla tabella padre, quando fetchiamo la sola tabella padre, evitiamo di fetchare inutilmente la tabella Extended.

In questo modo abbiamo fino ad ora 2 Entity auditate da Hibernate Envers. (notare su entrambe le entity l' annotation @Audited)

Quando facciamo insert/update di record nelle singole tabelle, Hibernate Envers storicizzerà record di auditing. Ma attenzione: se modifico solo 1 delle 2 tabelle, envers storicizza l' audit della sola tabella in questione, e non di tutte e 2. Altro punto di attenzione, le mie API permettono di fare update di solo MyTable, di MyTable e MyTableExtended insieme, ma non di solo MyTableExtended.

Ogni record di audit è identificato da un REV number: quando aggiorno entrambe le tabelle, ogni audit di tale versione in entrambe le tabelle di history, avrà lo stesso REV number.

A questo punto ho esposto un API che, preso in ingresso un id di myTable, ritorna tutte le revisioni di quell' id, e per ogni revisione, ci aggancia la myTableExtended relativa a quella revisione.

Prima di tutto ho introdotto un' interfaccia:

{% highlight java %}
public interface IEntityWithExtended<T> {
    T getEntityExtended();
    void setEntityExtended(T entityExtended);
}
{% endhighlight %}

A questo punto ho fatto implementare tale interfaccia da MyTable, inserendo in questa Entity un field @Transient:

{% highlight java %}
public class MyTable implements Serializable, IEntityWithExtended<MyTableExtended> {
   ...
   @Transient
   private MyTableExtended myTableExtended;

   @Override
   public MyTableExtended getEntityExtended() {
      return myTableExtended;
   }

   @Override
   public void setEntityExtended(MyTableExtended entityExtended) {
      myTableExtended = entityExtended;
   }
}
{% endhighlight %}

Fatto questo, ora l' obbiettivo è quello di chiamare un metodo che ritorna tutte le revisioni di MyTable, e tutte le revisioni di MyTableExtended. Dopodichè mergiare i 2 insiemi di revisioni, basandosi sul REV Number, l' identificativo di revisione discusso precedentemente. Dato che, come ho gia detto, la mia API permette di fare update di solo MyTable, di MyTable + MyTableExtended, ma non di sola MyTableExtended, sicuramente esisterà una revisione di MyTable per ogni update. Per quanto riguarda, invece, le reivisioni di MyTableExtended, non è detto che ne esiste una per ogni update di MyTable. In sostanza, utilizzero' un metodo che mi ritorna il REV Number di MyTableExtended giusto, relativo alla revisione in questione di MyTable. Per farla breve, data una revisione di MyTable (revisione 'x'), la sua relativa revisione di MyTableExtended è quella che avrà REV Number <= x.

Qui di seguito i metodi che mi sono scritto:

allRevisionMap():

{% highlight java %}
public final Map<Integer, E> allRevisionMap(final long id) {
    final AuditReader reader = AuditReaderFactory.get(getEntityManager());
    final List<Object[]> triplet = reader.createQuery().forRevisionsOfEntity(getEntityType(), false, true)
        .add(AuditEntity.id().eq(id))
        .getResultList();
    return this.getRevisionMap(triplet);
}
{% endhighlight %}

getRevisionMap():

{% highlight java %}
private Map<Integer, E> getRevisionMap(final List<Object[]> input) {
    return Optional.ofNullable(input)
                .map(Collection::stream)
                .orElseGet(Stream::empty)
                .collect(Collectors.toMap(el -> ((UserRevEntity) el[1]).getId(), el -> (E) el[0]));
}
{% endhighlight %}
Dove UserRevEntity è l' entity che mappa la tabella di Hibernate Envers. che contiene il REV Number di tutte le revisioni

findExtendedRevNumber():

{% highlight java %}
private Optional<Integer> findExtendedRevNumber(final Integer number, final Set<Integer> domain) {
    return domain.stream().filter(n -> n <= number).max(Comparator.naturalOrder());
}
{% endhighlight %}

mergeRevisionMap():

{% highlight java %}
public <EXT, T extends IEntityWithExtended<EXT>> List<T> mergeRevisionMap(
final Map<Integer, T> entityMap,
final Map<Integer, EXT> entityExtendedMap) {
    return entityMap.entrySet().stream()
                .peek(entry -> this.findExtendedRevNumber(entry.getKey(), entityExtendedMap.keySet()).ifPresent(n ->
                        entry.getValue().setEntityExtended(entityExtendedMap.get(n))))
                .map(Map.Entry::getValue)
                .collect(Collectors.toList());
}
{% endhighlight %}

A questo punto, dal Service basta fare:

{% highlight java %}
final Map<Integer, MyTable> myTableRevisionMap = allRevisionMap(id);
final Map<Integer, MyTableExtended> myTableExtendedRevisionMap = allRevisionMap(id);
final List<MyTable> myTables = mergeRevisionMap(myTableRevisionMap, myTableExtendedRevisionMap);
{% endhighlight %}

Spero possa essere d' aiuto anche a voi!

Ciao! :)