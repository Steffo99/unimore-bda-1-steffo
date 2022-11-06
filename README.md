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

1. Inserire il file `metaexport.json` all'interno della cartella `seed`, non allegato per via di diritti di ridistribuzione sconosciuti

2. Con un daemon Docker in esecuzione, e Docker Compose installato sulla macchina locale, "accendere" il progetto:
   ```console
   # docker compose up 
   ```
