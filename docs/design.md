# Helix Parser — Note di design

Come lavora con i file, le regole che vengono utilizzate dal parser.

---

## 1. Il documento normalizzato

Ogni file letto con successo diventa un oggetto `Document` con questi campi:

| Campo           | Tipo           | Da dove viene                                   |
|-----------------|----------------|-------------------------------------------------|
| `title`         | `str`          | Dal lettore del formato                         |
| `date`          | `date \| None` | Dal contenuto del file                          |
| `content`       | `str`          | Testo del file, normalizzato                    |
| `word_count`    | `int`          | Calcolato dal contenuto normalizzato            |
| `excerpt`       | `str`          | Calcolato dal contenuto normalizzato            |
| `source_format` | `str`          | Estensione del file, in minuscolo (es. `.md`)   |
| `source_path`   | `str`          | Percorso del file relativo alla cartella radice |

Nel file JSON la data viene salvata come testo nel formato `YYYY-MM-DD`,
oppure `null` se assente.

---

## 2. Flusso delle operazioni

Per ogni file trovato nella cartella:

1. **Controlla l'estensione.** Se non è gestita, il file viene scartato
   subito, senza aprirlo.
2. **Leggi il file** mantenendo il testo originale e la divisione in righe.
   Qui si scartano i file non UTF-8 e quelli binari.
3. **Estrai i metadati dal testo originale**, riga per riga:
   titolo e data.
4. **Ricava il contenuto testuale**, con una logica diversa per ogni formato
   (es. per il CSV: trasformare le celle in testo).
5. **Normalizza il contenuto**, con la stessa regola per tutti i formati.
   Se dopo la normalizzazione il contenuto è vuoto, il file viene scartato.
6. **Costruisci l'oggetto `Document`**: numero di parole ed estratto vengono
   calcolati qui, a partire dal contenuto normalizzato.

---

## 3. Titolo

| Formato | Regola principale                        | Se non trovato |
|---------|------------------------------------------|----------------|
| `.md`   | La prima riga che inizia con `# `        | Nome del file  |
| `.txt`  | Nessuna                                  | Nome del file  |
| `.csv`  | Nessuna                                  | Nome del file  |

- Il titolo è il testo che segue `# `, senza spazi iniziali e finali.
- le righe `## ...` non sono titoli di primo livello e vengono ignorate.
- **Nome del file:** senza estensione, con `_` e `-` sostituiti da spazi
  (es. `verbale_riunione-marzo.txt` → `verbale riunione marzo`).

Il nome del file è il comportamento predefinito per tutti i formati.
Un formato lo sostituisce solo se ha un modo migliore per trovare il titolo.

---

## 4. Data

La data si cerca nel testo originale, in questo ordine:

1. Una riga che inizia con `Data:` oppure `Date:` (maiuscole e minuscole
   non contano), seguita da una data.
2. Se non c'è, una riga che contiene **solo** una data, tra le prime 5 righe.

Formati accettati:

- `YYYY-MM-DD` (es. `2026-02-15`)
- `YY-MM-DD`   (es. `26-02-15`)
- `DD/MM/YYYY` (es. `15/02/2026`)
- `DD/MM/YY`   (es. `15/02/26`)
- `MM/DD/YYYY` (es. `02/15/2026`)
- `MM/DD/YY`   (es. `02/15/26`)

Regole:

- Una data scritta dentro una frase non viene considerata
  (es. "come da verbale del 12/03/2025").
- Se la data non c'è o non è valida (es. `31/02/2026`), il campo `date`
  vale `None`. Il documento resta valido.

---

## 5. Contenuto normalizzato

### Regola comune a tutti i formati

- Rimuovere gli spazi all'inizio e alla fine.
- Sostituire ogni sequenza di spazi, tab e a capo con un solo spazio.

### Regole per formato (passo 4 del flusso)

- **`.txt`:** il testo viene usato così com'è.
- **`.md`:** si rimuovono i simboli a inizio riga (`#`, `-`, `*`, `>`)
  e il grassetto `**`. Il resto resta invariato.
- **`.csv`:** ogni riga diventa i suoi valori separati da `, `.
  Anche la riga di intestazione viene inclusa.

### Campi calcolati

- **`word_count`:** numero di parole separate da spazi nel contenuto
  normalizzato.
- **`excerpt`:** le prime 30 parole del contenuto normalizzato.
  Si aggiunge `...` solo se il contenuto è più lungo.

---

## 6. Errori e file scartati

Un file problematico non blocca mai l'elaborazione: viene escluso dall'indice,
registrato nel riepilogo e si passa al file successivo.

Ogni scarto viene registrato con:
- **percorso** del file;
- **categoria**: un motivo fisso e leggibile, usato per raggruppare il riepilogo;
- **dettaglio**: il messaggio dell'errore originale, quando esiste.

| Caso                                        | Categoria                    | 
|---------------------------------------------|------------------------------|
| Estensione non supportata                   | `estensione non supportata`  |
| Il file non si può aprire (permessi, ecc.)  | `file illeggibile`           |
| Il file non è UTF-8 valido                  | `codifica non UTF-8`         |
| Il file contiene byte nulli                 | `contenuto binario`          |
| Contenuto vuoto dopo la normalizzazione     | `contenuto vuoto`            |

Casi che **non** sono scarti ma trattati in modo diverso:

| Caso                                        | Esito                                                     |
|---------------------------------------------|-----------------------------------------------------------|
| File o cartella nascosti (nome con `.`)     | Ignorati intenzionalmente, non compaiono nel riepilogo    |
| CSV con sola intestazione                   | Elaborato (contenuto = nomi delle colonne)                |
| Riga CSV con numero di colonne diverso      | Riga saltata e segnalata come avviso; file elaborato      |

Vengono intercettati solo gli errori legati ai file (lettura, codifica, contenuto).

Note:
- La lettura usa la codifica `utf-8-sig`, che accetta anche il BOM
  aggiunto da alcuni editor Windows.
- I file con accenti, emoji e simboli sono UTF-8 validi e vengono elaborati.

---

## 7. Scansione della cartella

- Le sottocartelle vengono esplorate.
- File e cartelle il cui nome inizia con `.` (es. `.DS_Store`) vengono
  ignorati e non compaiono nel riepilogo.
- Il controllo dell'estensione non distingue maiuscole e minuscole
  (`.TXT` è trattato come `.txt`).

---

## 8. Riepilogo finale

Al termine dell'elaborazione il programma mostra:

- il numero di documenti elaborati;
- il numero di documenti scartati, ciascuno con percorso e motivo;
- gli avvisi (es. righe CSV saltate), con il file a cui si riferiscono.
