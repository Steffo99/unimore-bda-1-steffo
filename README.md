\[ **Stefano Pigozzi** | Traccia #2 | Tema MongoDB | Big Data Analytics | A.A. 2022/2023 | Unimore \]

# Ricerca in linguaggio naturale sul dataset Amazon

> ### Indexing e query optimization o Text search
>
> Approfondire un argomento tra questi studiando il capitolo 8 (Indexing e query optimization) o 9 (Text search) del libro “MongoDB in action” disponibile sul sito dell’insegnamento.
>
> Riassumere quindi alcune delle tecniche imparate e mostrarne un'applicazione pratica su alcune nuove interrogazioni sul dataset Amazon utilizzato nelle esercitazioni.
>
> (es. decidere gli indici adeguati ottimizzandone l’esecuzione o mostrare l’utilizzo di varie tecniche di text search, commentando le scelte effettuate).


## Premessa

L'attività è stata svolta su MongoDB 6.0.2, attraverso la macchina virtuale predisposta su [Azure Labs](https://labs.azure.com/virtualmachines), `LabBDA2`.


## Introduzione

Sviluppando una applicazione o effettuando analisi su dati, è possibile aver bisogno di effettuare query sul database relative ai contenuti di esso in linguaggio naturale.

Per comodità, MongoDB include funzionalità basilari per la ricerca in linguaggio naturale, in modo che i suoi utilizzatori possano usufruirne senza dover configurare un motore di ricerca esterno come [ElasticSearch](https://www.elastic.co/).

Esistono [funzionalità di ricerca più avanzate](https://www.mongodb.com/docs/atlas/atlas-search/atlas-search-overview/) basate su [Apache Lucene](https://lucene.apache.org/), ma sono limitate al piano managed (a pagamento) del database, [MongoDB Atlas](https://www.mongodb.com/pricing): esse non saranno trattate in questa relazione.


### Il problema degli indici comuni

Quando si effettua una query in linguaggio naturale su una collezione di documenti, l'obiettivo è quello di trovare uno o più documenti dal contenuto più simile possibile al contenuto della richiesta effettuata: ad esempio, in un database di film, si potrebbe voler trovare un film con uno specifico titolo, o in cui compare un termine specifico nella descrizione.

Non effettuando confronti diretti (`"a" === "b"`) tra i contenuti dei campi interrogati e i termini della query, non è possibile effettuare una ricerca binaria sugli indici della collezione; è invece necessario utilizzare un indice apposito creato preventivamente detto *[Text Index](https://www.mongodb.com/docs/manual/core/index-text/)*, in grado di indicizzare correttamente le singole parole dei documenti nella collezione.

Essi usano strutture dati come le _posting list_, che catalogano i documenti all'interno dei quali appare un dato termine.


## Funzionamento generale del Text Index

### Creazione dell'indice

Per creare un Text Index, è necessario invocare il metodo `.createIndex()` della collezione con un oggetto dove tutti le chiavi dei campi che si vogliono indicizzare sono mappate alla stringa `"text"`.

Ogni collezione può avere associato **un solo Text Index**, che può però coprire qualsiasi numero di suoi campi di testo.

```javascript
// Creazione di un Text Index su una collezione inventata
db.EXAMPLE.createIndex({
   description: "text"
})
// -> "description_text"
```

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

Per operare efficacemente con il linguaggio naturale, è necessario effettuare alcune operazioni di preprocessing sulle stringhe in questione, trasformandole in insiemi di token.

Queste operazioni, su MongoDB, sono:

1. **tokenizzazione**: le parole vengono separate, ripulite da punteggiatura e diacritici, e trasformate in _token_
2. **eliminazione delle stopwords**: i token più comuni e insignificanti ai fini della ricerca, come le congiunzioni, vengono eliminati
3. **stemming**: i token vengono ridotti al loro prefisso, in modo che non ci sia disambiguazione tra singolari, plurali, maschili o femminili

Esse sono effettuate a runtime, nello specifico:
- quando si crea un nuovo Text Index, sono preprocessate le stringhe di tutti i documenti già esistenti;
- quando si inserisce un nuovo documento o se ne modifica uno già esistente, sono preprocessate le sue stringhe;
- quando si effettua un'interrogazione basata sul Text Index, sono preprocessati i termini di essa.


##### Lingua

Perchè il preprocessing funzioni correttamente, è necessario conoscere la lingua utilizzata all'interno delle stringhe da processare.

Essa può essere impostata a livello di collezione attraverso l'opzione `default_language` di `.createIndex()`:

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
      name: "english_text",
      default_language: "english",
   }
)
// -> "english_text"
```

È possibile sovrascrivere la lingua a livello di documento (anche innestato!) inserendo in esso la chiave `language`:

```javascript
({
   // Questo documento sarà processato in italiano...
   language: "italian",
   title: "Il Miglior Manuale di MongoDB del Mondo",
   description: "Un manuale di MongoDB in grado di insegnare a chiunque come fare query!",
   translations: [
      {
         // Questo documento innestato sarà processato in inglese!
         language: "english",
         title: "The Best MongoDB Manual in the World",
         description: "A MongoDB manual able to teach anyone how to construct queries!",
      }
   ]
})
```

È possibile disattivare l'eliminazione delle stopwords e lo stemming specificando la stringa `"none"` come lingua.

```javascript
({
   // Questo documento sarà solo tokenizzato
   language: "none",
   code: "A21 B34 ZZZ56",
})
```


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


#### Query speciali

Circondando di virgolette doppie `"` un termine della ricerca, è possibile richiedere la presenza esatta di uno dato termine o frase all'interno del documento:

```javascript
// Cerca documenti che contengono le parole della stringa "La Mia Query" nei campi indicizzati, e che hanno SICURAMENTE la parola "Mia" da qualche parte
db.EXAMPLE.find({
   $text: {
      $search: `La "Mia" Query`,
   }
})
```

```javascript
// Cerca documenti che contengono esattamente la stringa "La Mia Query" nei campi indicizzati
db.EXAMPLE.find({
   $text: {
      $search: `"La Mia Query"`,
   }
})
```

Prefissando un trattino `-` ad un termine è possibile nascondere i documenti che lo contengono:

```javascript
// Cerca documenti che contengono le parole della stringa "La Mia Query" nei campi indicizzati, e che hanno SICURAMENTE la parola "Mia" da qualche parte
db.EXAMPLE.find({
   $text: {
      $search: `La Mia Query -Sua -Tua`,
   }
})
```


#### Query miste

È possibile effettuare query `$text` assieme a query "normali", effettuando l'intersezione dei risultati:

```javascript
// Cerca documenti che siano libri e che contengano le parole della stringa "Tutto su MongoDB"
db.EXAMPLE.find({
   $text: {
      $search: "Tutto su MongoDB",
   },
   type: "book",
})
```


#### Punteggio

Al fine di ordinare i documenti restituiti dalla query `$text`, a ciascuno di essi viene assegnato un punteggio, che dipende quanto ogni token di esso è rilevante alla richiesta effettuata, e, se specificati, dai pesi dell'indice interrogato.

È possibile includere il punteggio nel contenuto dei documenti restituiti specificando l'oggetto `{$meta: "textScore"}` come valore di una delle chiavi di proiezione del metodo `.find()`.

```javascript
// Cerca documenti che contengano le parole della stringa "Tutto su MongoDB"
db.EXAMPLE.find(
   {
      $text: {
         $search: "Tutto su MongoDB",
      },
   },
   {
      score: {$meta: "textScore"}
   }
)
```


## Interrogazioni di esempio sul dataset Amazon

### 0 - Creazione del Text Index sulla collezione `meta`

La collezione `meta` ha un solo campo in cui sono utilizzate stringhe con testo in linguaggio naturale: `description`.

Pertanto, si è creato un indice contenente solo quel campo specifico.

```javascript
db.meta.createIndex(
   {
      description: "text",
   },
   {
      name: "meta_text",
      default_language: "english",
   }
)
```


### 0 - Creazione del Text Index sulla collezione `reviews`

La collezione `reviews` ha due campi di testo in linguaggio naturale: `summary`, il titolo della recensione, e `reviewText`, il contenuto completo della recensione.

Quindi, si è eliminato l'indice pre-esistente `summary_text`, e si è creato un indice contenente quei due campi, dando peso maggiore al campo `summary`, più rilevante.

```javascript
db.reviews.dropIndex("summary_text")

db.reviews.createIndex(
   {
      summary: "text",
      reviewText: "text",
   },
   {
      name: "reviews_text",
      default_language: "english",
      weights: {
         summary: 2,
         reviewText: 1,
      }
   }
)
```


### 1 - Ricerca semplice

Si desidera cercare all'interno della collezione tutti i dati relativi al videogioco _[Terraria](https://store.steampowered.com/app/105600/Terraria/)_.

Si effettua una query di ricerca per il termine "Terraria":

```javascript
db.meta.findOne(
   {
      $text: {$search: "Terraria"}
   }
)
```

La query ha successo e restituisce un documento con le caratteristiche richieste, ma il dataset pare essere sporco: immagine e prezzo del prodotto si riferiscono a una copia del videogioco _[Minecraft](https://www.minecraft.net/en-us)_. 

```javascript
({
  _id: ObjectId("58adad4fb2d633c456578630"),
  asin: '9941113300',
  description: "Dig, fight, explore, build! Nothing is impossible in this action-packed adventure game. The world is your canvas and the ground itself is your paint. Grab your tools and go! You can do many things in Terraria: make weapons and fight off a variety of enemies in numerous biomes, dig deep underground to find accessories, money, and other useful things, gather wood, stone, ores, and other resources to create everything you need to make the world your own and defend it. Build a house, a fort, even a castle and people will move in to live there and perhaps even sell you different wares to assist you on your journey. But beware, there are even more challenges awaiting you. Are you up to the task? ? ??? Terraria: Collector's Edition Includes: ??? In game item ??? Trading cards ??? Poster ??? Product Features: ??? Sandbox play ??? Randomly generated worlds ??? Free content updates ??? Co-op and PvP multiplayer modes playable via internet ? System Requirements OS: Windows Xp, Vista, 7. Processor: 1.6 Ghz. Memory: 512MB. Hard Disk Space: 200MB. Video Card: 128mb Video Memory, capable of Shader Model 1.1. DirectX?: 9.0c or Greater.",
  price: 19.78,
  imUrl: 'http://ecx.images-amazon.com/images/I/51OZAkphvcL._SY300_.jpg',
  related: {
    also_viewed: [
      'B00JQHU9RC', 'B00BU3ZLJQ', 'B00750FR6A',
      'B00DBCGC9C', 'B00DUV027M', 'B003YEXO3Y',
      'B00KVQYJR8', 'B003YEXO66', 'B00KVQYM2U',
      'B00E7SB8WK', 'B00L5M2I34', 'B00F8KSX0G',
      'B00KN1GBW2', 'B007PVHMCG', 'B00D5BGCUI',
      'B00GEECDA6', 'B00J58RFD8', '054568515X',
      'B00JIOMB60', 'B00DS5FBP8', '0545669936',
      'B00HLV78GA', 'B00FEN164W', 'B00C6R5GHM',
      '1438826486', 'B00L3GSIL8', 'B00E45ELGQ',
      '1438832931', '1500164615', 'B00DOQCWF8',
      'B00HLV78TW', 'B003YF51RA', 'B00KX95JLS',
      'B00EJOCAYW', 'B00DOQD0U4', 'B00DOQCZM8',
      '1405268425', '1405268417', 'B00DOQD0YK',
      '1405267674', 'B00K0N9ARG', 'B00CHKEF7K',
      'B00HSQS3IU', '1118537149', 'B00HYJNJNA',
      'B00JRCHT8I', 'B00DOQCV9U', 'B00B0FV4FE',
      'B00ENVS2KM', 'B00EJOCB4G', 'B00DOQCTWY'
    ],
    buy_after_viewing: [ 'B00JQHU9RC', 'B00BU3ZLJQ', '054568515X', '0545669936' ]
  },
  salesRank: { 'Video Games': 49227 },
  categories: [ [ 'Video Games', 'PC', 'Games' ] ]
})
```


### 2 - Ricerca mista

Si desidera trovare tutte le recensioni negative (2 stelle o inferiore) relative a qualche tipo di gioco (contenenti la parola "game" in qualche campo).

Si effettua la seguente query:

```javascript
db.reviews.find(
   {
      $text: {$search: "game"},
      overall: {$lte: 2},
   }
)
```

La query ha successo, e restituisce 192139 documenti.


### 3 - Aggregazione con ricerca speciale

Si desidera trovare il sales rank medio di prodotti riguardanti "Dungeons & Dragons" (e non con le parole Dungeons e Dragons nella descrizione).

Si effettua la seguente aggregazione:

```javascript
db.meta.aggregate([
   // Trova i documenti che contengono "Dungeons & Dragons"
   {$match: {
      $text: {$search: `"Dungeons & Dragons"`}
   }},
   // Trasforma il sottodocumento salesRank in un array
   {$project: {
      salesRank: {$objectToArray: "$salesRank"},
   }},
   // Effettua l'unwind dell'array
   {$unwind: {
      path: "$salesRank",
   }},
   // Raggruppa per categoria calcolando la media
   {$group: {
      _id: "$salesRank.k",
      avg: {$avg: "$salesRank.v"},
   }}
])
```

L'aggregazione ha successo, e restituisce il seguente risultato:

```javascript
[
   { _id: 'Video Games', avg: 30079.060606060608 },
   { _id: 'Books', avg: 257210 },
   { _id: 'Software', avg: 26086 }
]
```

### 4 - Aggregazione con ricerca mista

Si desidera trovare il sales rank medio, minimo e massimo dei videogiochi contenenti il termine "Dante".

Si effettua la seguente aggregazione:

```javascript
db.meta.aggregate([
   // Trova i videogiochi che contengono il termine "Dante"
   {$match: {
      $text: {$search: "Dante"},
      // Le categorie sono tuple di termini dal più generale al più specifico; per risolvere questa query è sufficiente che un termine qualsiasi sia "Video Games"
      categories: {$elemMatch: {$elemMatch: {$eq: "Video Games"}}},
   }},
   // Trasforma il sottodocumento salesRank in un array
   {$project: {
      salesRank: {$objectToArray: "$salesRank"},
   }},
   // Effettua l'unwind dell'array salesRank
   {$unwind: {
      path: "$salesRank",
   }},
   // Raggruppa per categoria calcolando posizione minima, media e massima
   {$group: {
      _id: "$salesRank.k",
      best: {$min: "$salesRank.v"},
      avg: {$avg: "$salesRank.v"},
      worst: {$max: "$salesRank.v"}
   }}
])
```

L'aggregazione ha successo, e restituisce il seguente risultato:

```javascript
[
   {
      _id: 'Video Games',
      best: 2112,
      avg: 17816.904761904763,
      worst: 74608
   }
]
```


### 5 - Aggregazione con ricerca e lookup

Si desidera trovare una bizzarra irregolarità statistica: trovare i dati dei 10 prodotti con le recensioni più alte, contando solo le recensioni in cui compare la parola "cringe".

Si effettua la seguente query:

```javascript
db.reviews.aggregate([
   // Trova le recensioni che contengono il termine "cringe"
   {$match:
      {$text: {$search: "cringe"}},
   },
   // Raggruppa le recensioni per codice prodotto, conteggiandole e calcolandone la valutazione media
   {$group: {
      _id: "$asin",
      avgRating: {$avg: "$overall"},
      reviewCount: {$sum: 1},
   }},
   // Scarta i prodotti che non hanno almeno 3 recensioni "cringe": sarebbero sicuramente degli outliers
   {$match: {
      reviewCount: {$gte: 3},
   }},
   // Effettua il lookup dei dati completi dei prodotti attraverso l'asin
   {$lookup: {
      from: "meta",
      localField: "_id",
      foreignField: "asin",
      as: "product",
   }},
   // Dato che nella collezione "meta" ci potrebbero essere più prodotti con lo stesso asin, effettua l'unwind dell'array dei prodotti trovati
   {$unwind: {
      path: "$product",
   }},
   // Mantieni solo i prodotti per cui è stato trovato un corrispondente nella tabella meta; ci sono tantissime non-corrispondenze
   {$match: {
      product: {$exists: true},
   }},
   // Effettua una proiezione per mantenere solo i campi a cui si è interessati
   {$project: {
      _id: 1,
      avgRating: 1,
      reviewCount: 1,
      title: "$product.title",
   }},
   // Ordina i prodotti per valutazione media decrescente, poi per numero di recensioni decrescente, e infine per ordine lessicale di ASIN, in modo da avere risultati deterministici
   {$sort: {
      avgRating: -1,
      reviewCount: -1,
      _id: 1,
   }},
   // Mostra solo i 10 prodotti con valutazioni migliori
   {$limit: 10},
])
```

L'aggregazione ha successo, e restituisce la seguente "classifica":

```javascript
[
   {
      _id: '0060392991',
      avgRating: 5,
      reviewCount: 8,
      title: 'Mankind: Have a Nice Day - A Tale of Blood and Sweatsocks'
   },
   {
      _id: '0060890096',
      avgRating: 5,
      reviewCount: 4,
      title: 'Catch-22 CD'
   },
   {
      _id: '0061461369',
      avgRating: 5,
      reviewCount: 4,
      title: 'Jennifer Johnson Is Sick of Being Single: A Novel'
   },
   {
      _id: '0062108026',
      avgRating: 5,
      reviewCount: 4,
      title: 'Slow Getting Up: A Story of NFL Survival from the Bottom of the  Pile'
   },
   {
      _id: '0099496941',
      avgRating: 5,
      reviewCount: 4,
      title: 'All Quiet on the Western Front'
   },
   {
      _id: '0007423632',
      avgRating: 5,
      reviewCount: 3,
      title: 'Prince of Thorns (The Broken Empire)'
   },
   {
      _id: '0061713244',
      avgRating: 5,
      reviewCount: 3,
      title: 'The Longest Trip Home: A Memoir'
   },
   {
      _id: '0062115359',
      avgRating: 5,
      reviewCount: 3,
      title: 'Spin: A Novel'
   },
   {
      _id: '0007386648',
      avgRating: 4.909090909090909,
      reviewCount: 11,
      title: 'Unbroken'
   },
   {
      _id: '0060838582',
      avgRating: 4.75,
      reviewCount: 8,
      title: 'Fast Food Nation: The Dark Side of the All-American Meal'
   }
]
```
