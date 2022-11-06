# Ricerca in linguaggio naturale sul dataset Amazon

\[ **Stefano Pigozzi** | Attività #2 | Tema MongoDB | Big Data Analytics | A.A. 2022/2023 | Unimore \]

> ### Indexing e query optimization o Text search
>
> Approfondire un argomento tra questi studiando il capitolo 8 (Indexing e query optimization) o 9 (Text search) del libro “MongoDB in action” disponibile sul sito dell’insegnamento.
>
> Riassumere quindi alcune delle tecniche imparate e mostrarne un'applicazione pratica su alcune nuove interrogazioni sul dataset Amazon utilizzato nelle esercitazioni.
>
> (es. decidere gli indici adeguati ottimizzandone l’esecuzione o mostrare l’utilizzo di varie tecniche di text search, commentando le scelte effettuate).


## Premessa

L'attività è stata svolta su MongoDB 6.0.2, attraverso il progetto [Docker Compose](https://docs.docker.com/compose/) allegato.

Per ricreare lo stesso ambiente di lavoro utilizzato, sarà necessario:

1. Inserire il file `metaexport.json` all'interno della cartella `seed`, non allegato per motivi di dimensioni.

2. Con un daemon Docker in esecuzione, e Docker Compose installato sulla macchina locale, "accendere" il progetto:
   ```console
   # docker compose up -d
   ```

3. Con la [MongoDB Shell](https://www.mongodb.com/try/download/shell) installata sulla macchina locale, è possibile interfacciarsi al database con:
   ```console
   $ mongosh --username=unimore --password=unimore --authenticationDatabase=admin mongodb://127.0.0.1:27017/amazon
   ```


## Introduzione

Sviluppando una applicazione o effettuando analisi su dati, si potrebbe aver bisogno di effettuare query su un database relative ai contenuti in linguaggio naturale di esso.

Ad esempio, su un database come quello Amazon, si potrebbe voler trovare i prodotti contenenti una parola o frase specifica all'interno della descrizione.

Per comodità, potrebbe essere utile avere le funzionalità di base già disponibili su MongoDB, 

Non effettuando confronti diretti tra i contenuti dei campi interrogati e i termini della query, non è possibile utilizzare gli indici comuni della collezione interrogata.

È invece necessario creare un indice apposito, detto *[Text Index](https://www.mongodb.com/docs/manual/core/index-text/)*, in grado di processare il contenuto in linguaggio naturale di uno o più campi di stringhe della collezione.


## Funzionamento generale del Text Index

Ogni collezione può avere associato **un solo Text Index**, che può però coprire qualsiasi numero di suoi campi di testo.


### Creazione dell'indice

Per creare un Text Index, è necessario invocare il metodo `.createIndex()` della collezione con un oggetto dove tutti le chiavi dei campi che si vogliono indicizzare sono mappate alla stringa `"text"`:

```javascript
// Creazione di un Text Index su una collezione inventata
db.EXAMPLE.createIndex({
   description: "text"
})
// -> "description_text"
```

Tutti i documenti già presenti nella collezione vengono indicizzati al momento di creazione dell'indice, quindi l'operazione potrebbe richiedere un po' di tempo.

```javascript
// Creazione di un Text Index multi-campo su una collezione inventata
db.EXAMPLE.createIndex({
   title: "text",
   description: "text",
   context: "text",
   // Anche gli array di stringhe sono indicizzati!
   comments: "text",
})
// -> "title_description_context_comments_text"
```

```javascript
// Creazione di un Text Index wildcard su una collezione inventata
db.EXAMPLE.createIndex({
   // Indicizza qualsiasi stringa contenuta nei documenti
   "$**": "text"
})
// -> "$**_text"
```

#### Personalizzazione nome dell'indice

Per indici multi-campo, è consigliabile specificare nelle opzioni di `.createIndex()` un nome attraverso il campo `name`, onde evitare il comportamento predefinito di MongoDB di concatenare i nomi dei campi che l'indice contiene:

```javascript
// Creazione di un Text Index con nome personalizzato
db.EXAMPLE.createIndex(
   {
      title: "text",
      description: "text",
      context: "text",
      comments: "text",
   }, 
   {
      name: "example_text"
   }
)
// -> "example_text"
```

#### Selezione pesi

Per dare più priorità ad certi campi rispetto ad altri nella ricerca, attraverso l'opzione `weights` di `.createIndex()` è possibile specificare il peso di ciascun campo:

```javascript
// Creazione di un Text Index con nome e pesi personalizzato
db.EXAMPLE.createIndex(
   {
      title: "text",
      description: "text",
      context: "text",
      comments: "text",
   }, 
   {
      name: "better_example_text",
      weights: {
         title: 100,
         description: 10,
         context: 10,
         comments: 7,
      }
   }
)
// -> "better_example_text"
```

#### Preprocessing delle stringhe

Le stringhe vengono preprocessate 

<!-- TODO -->

### Interrogazione sull'indice

Una volta creato l'indice, è possibile usarlo per effettuare interrogazioni di linguaggio naturale sulla collezione utilizzando i metodi della collezione `.find()` e le chiavi speciali `$text` e `$search`:

```javascript
// Cerca documenti che contengono le parole della stringa "La Mia Query" nei campi indicizzati
db.EXAMPLE.find({
   $text: {
      $search: "La Mia Query",
   }
})
```
