---
layout: post
title:  "How to install Oracle12c on Linux Debian8"
---
Ciao a tutti,

scrivo quest' articolo dopo essere riuscito a installare Oracle12c su Linux Debian8.

Purtroppo Oracle supporta in via ufficiale solo Linux RedHat.

Per questo motivo, dopo essere riuscito ad installarlo su Debian8, mi sono detto di scrivere una guida, che potrebbe essere d' aiuto anche a voi.

Partiamo.

Innanzitutto scaricare i sorgenti di Oracle12 (sono 2 file zip da scaricare dal sito Oracle. Serve la login):

1. <a href="http://download.oracle.com/otn/linux/oracle12c/121020/linuxamd64_12102_database_1of2.zip" target="_blank">src1</a>
2. <a href="http://download.oracle.com/otn/linux/oracle12c/121020/linuxamd64_12102_database_2of2.zip" target="_blank">src2</a>

Dopodichè scaricare lo script <a href="https://github.com/danielemams/oracle12c-debian8/blob/master/install.sh" target="_blank">install.sh</a> dal mio repo github.

Dopo aver fatto cio:

1. scompattare i due src.zip uno sopra l' altro

{% highlight shell %}
mams@debian:~$ mkdir oracle12c
mams@debian:~$ unzip linuxamd64_12102_database_1of2.zip -d oracle12c
mams@debian:~$ unzip linuxamd64_12102_database_2of2.zip -d oracle12c
{% endhighlight %}

2. copia install.sh nella directory scompattata

{% highlight shell %}
mams@debian:~$ cp install.sh oracle12c
{% endhighlight %}

3. esegui install.sh

{% highlight shell %}
mams@debian:~$ ./install.sh
{% endhighlight %}

4. eseguire il seguente comando

{% highlight shell %}
mams@debian:~$ sudo chmod a+x $ORACLE_HOME/bin/sqlplus && sudo ln -s $ORACLE_HOME/bin/sqlplus /usr/local/bin/ 2> /dev/null
{% endhighlight %}

A questo punto scaricare sempre dal mio repo github <a rel="noreferrer noopener" href="https://github.com/danielemams/oracle12c-debian8/blob/master/oracle_start.sh" target="_blank">oracle_start.sh</a>.

Dopodichè, per Debian8, si dovrebbe modificare /usr/local/bin/oraenv alla riga 225 da:

if [ ${ORACLE_BASE:-"x"} == "x" ]; then

a:

if [ ${ORACLE_BASE:-"x"} = "x" ]; then

In realta, dopo aver parlato con <a href="https://github.com/teknoraver" target="_blank">Matteo Croce</a>, siamo giunti a tale conclusione:

Lo script "/usr/local/bin/oraenv" usa come shebang #!/bin/sh.

Su RHEL (supportata da Oracle) /bin/sh è un symlink a /bin/bash, mentre in debian /bin/sh è un symlink a dash (la shell di Debian). Su bash funziona l' == (estensione di bash), su dash no, perche supporta solo script POSIX. 

Quindi le 3 soluzioni sono:

1. o si reimposta a livello globale il symlink di sh a bash (soluzione preferibile tra le 3), tramite:

{% highlight shell %}
mams@debian:~$ sudo dpkg-reconfigure dash
{% endhighlight %}

2. o quando vogliamo avviare oracle_start.sh (ma cosi rischiamo che altri script lanciati, se esistono, non vadano sotto bash):

{% highlight shell %}
mams@debian:~$ bash /usr/local/bin/oraenv
{% endhighlight %}

3. o, come gia detto sopra, si modifica lo script /usr/local/bin/oraenv alla riga 225 (ma anche qui risolveremo il singlo script, e non eventuali altri lanciati a catena, se esistono) da:

if [ ${ORACLE_BASE:-"x"} == "x" ]; then

a: 

if [ ${ORACLE_BASE:-"x"} = "x" ]; then

Ora si puo' lanciare lo script oracle_start.sh.

Una volta dentro a sqlplus, al prompt SQL> :

- per avviare scrivere: STARTUP;
- per stoppare scrivere: SHUTDOWN;
- per uscire scrivere: EXIT;

Per creare nuove istanze di database, una volta dentro a sqlplus:

{% highlight sql %}
SQL> create user NOME_DB identified by PASSWORD;
{% endhighlight %}

Se dovesse rispondere con:

{% highlight sql %}
ORA-65096: invalid common user or role name in oracle
{% endhighlight %}

Allora eseguire:

{% highlight sql %}
SQL> alter session set "_ORACLE_SCRIPT"=true;
{% endhighlight %}

Poi dare le grant, ad esempio:

{% highlight sql %}
SQL> grant all privileges to NOME_DB;
{% endhighlight %}

Fatto questo, avete un istanza di oracle12c su debian8 funzionante, e la possibilita di aggiungere N database.

Spero sia di vostro aiuto.

Ciao! :)