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

Sviluppando una applicazione o effettuando analisi su dati, è possibile aver bisogno di effettuare query sul database relative ai contenuti di esso in linguaggio naturale.

Per comodità, MongoDB include funzionalità basilari per la ricerca in linguaggio naturale, in modo che i suoi utilizzatori possano usufruirne senza dover configurare un motore di ricerca esterno come [ElasticSearch](https://www.elastic.co/).


### Il problema degli indici

Una richiesta molto comune che riguarda il testo naturale è trovare i documenti contenenti parole specifiche in uno o più campi: ad esempio, in una collezione di film, si potrebbe voler trovare quello con un determinato titolo.

Non effettuando confronti diretti (`===`) tra i contenuti dei campi interrogati e i termini della query, non è possibile fare uso dei comuni indici usati per la ricerca binaria; è invece necessario un indice apposito, detto *[Text Index](https://www.mongodb.com/docs/manual/core/index-text/)*, in grado di processare correttamente le stringhe dei documenti nella collezione.


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

> Ho scelto di considerare il contenuto e l'effetto di queste operazioni oltre lo scopo di questo testo, in quanto è stato ampiamente trattato in Gestione dell'Informazione alla triennale.

MongoDB effettua queste operazioni a runtime, nello specifico:
- quando si crea un nuovo Text Index, sono processate le stringhe di tutti i documenti già esistenti;
- quando si inserisce un nuovo documento o se ne modifica uno già esistente, sono processate le sue stringhe;
- quando si effettua un'interrogazione basata sul Text Index, sono processati i termini di essa.

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

Prefissando un trattino `-` ad un termine è possibile filtrare i documenti che lo contengono:

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

### 0 - Creazione dell'indice sulla collezione Meta

La collezione `meta` ha un solo campo in cui sono utilizzate stringhe con testo in linguaggio naturale: `description`.

Pertanto, si crea un indice contenente solo quel campo specifico.

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


### 1 - Ricerca semplice

Si desidera cercare all'interno della collezione il videogioco _[Terraria](https://store.steampowered.com/app/105600/Terraria/)_.

Si effettua una query dove il termine "Terraria" viene cercato all'interno del Text Index.

```javascript
db.meta.findOne(
   {
      $text: {$search: "Terraria"}
   }
)
```


La query ha avuto successo ed ha restituito il prodotto che aspettato:

<details>
<summary>Risultato ottenuto</summary>

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

</details>


### 2 - 


### 3 - 


### 4 - 


### 5 - 

