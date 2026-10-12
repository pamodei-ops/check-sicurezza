---
name: check-sicurezza
description: Controllo di base della sicurezza di un'app web costruita con l'AI, pensato per lo stack React + Vite + Supabase (tipico di Lovable). Usala quando chiedi "controlla la sicurezza della mia app", "posso andare in produzione?", "ci sono chiavi esposte?", "ricontrolla dopo le correzioni" o prima di un lancio. Solo lettura: non modifica nulla e non attacca l'app in produzione. Ricontrolla ogni problema prima di segnalarlo e restituisce un rapporto datato in due colonne, indispensabile e miglioria, da rifare dopo le correzioni finché un giro torna pulito.
---

# Controllo di sicurezza di base (React + Vite + Supabase)

Fai un controllo **di base** della sicurezza dell'app che si trova nella cartella di lavoro. Chi ti
chiede il controllo spesso non è uno sviluppatore: spiega ogni problema in parole semplici e per
ognuno scrivi la frase da dare all'AI che ha costruito l'app (Lovable o altro) per correggerlo.

Non è un penetration test e non sostituisce uno sviluppatore prima di un lancio importante. Dillo
nel rapporto.

## Regole che non si discutono

1. **Solo lettura.** Non modificare file, non fare commit, non lanciare migrazioni, non cambiare
   impostazioni. Se trovi un problema, lo descrivi: non lo correggi.
2. **Niente attacchi all'app in produzione.** Le prove "dal vivo" (passo 9) solo se chi ti ha chiesto
   il controllo dice di sì in modo esplicito, e solo su un ambiente di prova.
3. **Mai segreti in chiaro.** Se trovi una chiave, nel rapporto scrivi dove sta e i primi 6 caratteri
   seguiti da `…`. Mai la chiave intera, nemmeno nel terminale.
4. **Mai dati di persone.** Se una query restituisce dati personali, riporta solo quante righe sono
   e quali colonne, non il contenuto.
5. **Non inventare.** Se un controllo non si può fare (manca l'accesso al database, manca una
   cartella), scrivilo nella sezione "Cosa non ho controllato".

## Il metodo: come lavori

Quattro abitudini, prese da come lavora uno sviluppatore che deve firmare il "via libera":

1. **Parti dalla versione giusta.** Prima di controllare, assicurati di avere l'ultima versione del
   codice (passo 0). Un controllo fatto su codice vecchio dà risposte sbagliate con molta sicurezza.
2. **Due passate: prima trovi, poi ricontrolli.** Nella prima passata raccogli i sospetti. Nella
   seconda li riprendi uno per uno come farebbe un revisore che non si fida di chi li ha trovati,
   nemmeno di te: rileggi il codice o la regola vera, e cerchi se il problema è già protetto altrove.
   Solo dopo lo metti nel rapporto, segnato come **verificato** o **da confermare**. Anche gli
   strumenti automatici danno falsi allarmi: un sospetto non è un problema finché non l'hai visto.
3. **Controlla quello che gira davvero.** Quello che sta nel repository non è per forza quello che è
   pubblicato: una modifica unita non è una modifica pubblicata. Se puoi, confronta (passo 4); se
   non puoi, scrivilo.
4. **Un giro non basta.** Il rapporto si conserva con la data. Dopo le correzioni il controllo si
   rifà, rileggendo il rapporto precedente: ogni problema diventa **risolto**, **ancora aperto** o
   **nuovo**. Si va avanti finché un giro torna pulito. "L'AI dice che è sistemato" non vale come
   risolto: si riverifica.

## Passo 0 · Capire il progetto

- **La versione giusta.** Se la cartella è un repository git: `git status` e `git fetch`, poi
  confronta con il ramo principale remoto (`git log HEAD..origin/main --oneline`). Se sei indietro,
  dillo e chiedi di aggiornare prima di continuare: non fare `pull` da solo se ci sono modifiche
  locali. Annota il commit da cui parti (`git rev-parse --short HEAD`): va nel rapporto.
- **Il giro precedente.** Se nella cartella c'è già un `report-sicurezza-*.md`, leggi il più recente:
  questo controllo è il giro successivo (vedi "Il rapporto").
- Leggi `package.json`, `vite.config.*`, la cartella `src/` e, se c'è, la cartella `supabase/`
  (`migrations/`, `functions/`, `config.toml`).
- Guarda quali file `.env*` esistono e quali sono tracciati da git: `git ls-files | grep -i env`.
- Se hai accesso a Supabase in sola lettura (MCP o CLI già collegata), dichiara su quale progetto
  stai lavorando prima di ogni query. Se non ce l'hai, lavora sulle migrazioni del repository e
  dillo nel rapporto.

## Passo 1 · Chiavi segrete finite nel posto sbagliato

Tutto quello che finisce nel codice del sito lo può leggere chiunque apra la pagina.

- Ogni variabile che inizia con `VITE_` è **pubblica**: finisce nel codice del browser. Elencale.
  Se il nome contiene `SECRET`, `SERVICE`, `PRIVATE`, `SK_` o simili → indispensabile.
- Cerca in `src/`, nei file di configurazione e, se c'è, in `dist/`:
  - la chiave `service_role` di Supabase (un JWT che inizia con `eyJ` e nel cui contenuto c'è
    `"role":"service_role"`);
  - chiavi di pagamento segrete (`sk_live_`, `sk_test_`, `rk_live_`), chiavi private
    (`-----BEGIN`), token di API esterne scritti nel codice.
- Cerca le stesse cose nelle migrazioni SQL e nella cronologia recente (`git log -p -S "service_role" -- . | head`).
- File `.env` con valori veri tracciati da git → indispensabile.

La chiave `anon` (o `publishable`) di Supabase **è pubblica per scelta**: non è un problema che sia
nel codice. Il problema è se il database si fida di lei (passi 2-4).

## Passo 2 · Le tabelle: chi può leggere e scrivere cosa

Su Supabase la difesa vera sono le regole di accesso (RLS) e i permessi (GRANT). Sono due strati
diversi e servono tutti e due.

Se hai accesso al database, in sola lettura:

```sql
-- tabelle senza RLS
select c.relname as tabella
from pg_class c join pg_namespace n on n.oid = c.relnamespace
where n.nspname = 'public' and c.relkind = 'r' and not c.relrowsecurity;

-- regole aperte a tutti
select tablename, policyname, cmd, roles, qual, with_check
from pg_policies
where schemaname = 'public' and (qual = 'true' or with_check = 'true');

-- cosa possono fare i visitatori non loggati (anon) sulle tabelle
select table_name, privilege_type
from information_schema.role_table_grants
where table_schema = 'public' and grantee in ('anon', 'PUBLIC');
```

Senza accesso, cerca nelle migrazioni `CREATE TABLE` senza il relativo
`ENABLE ROW LEVEL SECURITY`, e le policy con `USING (true)` o `WITH CHECK (true)`.

Se puoi leggere gli avvisi automatici di Supabase (Security Advisor, o `get_advisors` dall'MCP),
usali come elenco di sospetti, non come verdetto: spesso decine di avvisi si riducono a pochi
problemi veri. Per ognuno guarda il codice: una funzione segnalata come "eseguibile da tutti" che
dentro controlla chi la chiama non è un problema; una vista segnalata può essere voluta.

Come leggere i risultati:
- tabella con dati di persone (email, telefono, profili, ordini, messaggi) senza RLS → indispensabile;
- regola `true` per scrivere (INSERT, UPDATE, DELETE, ALL) → indispensabile;
- regola `true` per leggere dati di persone → indispensabile; su dati pubblici (es. il programma
  di un evento) → di solito va bene, segnalalo come miglioria solo se c'è un dubbio;
- nella colonna `roles`, una regola per tutti compare come `{public}`: non scartarla filtrando
  solo `anon` o `authenticated`.

## Passo 3 · Le funzioni del database

```sql
select p.proname as funzione,
       p.prosecdef as security_definer,
       has_function_privilege('anon', p.oid, 'EXECUTE') as eseguibile_da_anon,
       p.proconfig as impostazioni
from pg_proc p join pg_namespace n on n.oid = p.pronamespace
where n.nspname = 'public';
```

- `SECURITY DEFINER` eseguibile da `anon` → leggi il corpo: se non controlla chi la chiama e
  legge o scrive dati di persone → indispensabile.
- `SECURITY DEFINER` senza `search_path` fissato nelle impostazioni → miglioria.
- Togliere il permesso solo ad `anon` non basta: `anon` lo eredita da `PUBLIC`. La correzione
  giusta è `REVOKE EXECUTE ... FROM PUBLIC`, poi `GRANT` solo ai ruoli che servono.

## Passo 4 · Le edge function

In `supabase/functions/*` e in `supabase/config.toml`:

- `verify_jwt = true` **non è un permesso**: la chiave pubblica dell'app vale come "loggato".
  Ogni funzione che usa la chiave `service_role` deve controllare **dentro** chi la sta chiamando
  e con che ruolo. Funzione con `service_role` e nessun controllo sul chiamante → indispensabile.
- `verify_jwt = false` è lecito solo se la funzione si protegge da sola (firma di un webhook, un
  segreto condiviso): verifica che lo faccia.
- Segreti scritti nel codice invece che letti con `Deno.env.get(...)` → indispensabile.
- **Quella pubblicata è la stessa?** Se hai accesso a Supabase, confronta il codice delle funzioni
  pubblicate (dalla dashboard, dalla CLI o dall'MCP) con quello del repository. Se una correzione
  c'è nel repository ma non nella funzione pubblicata, il problema è ancora aperto: dillo.
  Vale anche per il sito: le modifiche unite su GitHub arrivano online solo quando lo strumento le
  pubblica.

## Passo 5 · I controlli fatti solo nella grafica

Nascondere un pulsante non è un permesso. Cerca le pagine o le azioni "da admin" protette solo nel
frontend (route guard, `if (isAdmin)` in React) che poi leggono o scrivono il database direttamente
con `supabase.from(...)`. Se il database non ripete lo stesso controllo (passi 2-4) →
indispensabile.

## Passo 6 · Accesso e redirect

- Parametri come `next`, `redirect`, `returnTo` usati per mandare l'utente a un indirizzo dopo il
  login senza controllare che sia un indirizzo dell'app → indispensabile (redirect aperto).
- Su Supabase Auth, se puoi vederle: conferma dell'email, protezione dalle password già rubate,
  indirizzi di redirect ammessi. Quello che è spento → miglioria, spiegando cosa comporta.

## Passo 7 · File caricati

Bucket di Supabase Storage pubblici che contengono file di persone (documenti, foto, ricevute) →
indispensabile. Bucket pubblici con immagini del sito → va bene.

## Passo 8 · Pagamenti (se ci sono)

- Il prezzo deve deciderlo il server: un prezzo che arriva dal browser e viene passato così com'è
  al sistema di pagamento → indispensabile.
- I webhook di pagamento devono verificare la firma (es. Stripe: `constructEvent` con l'header
  `stripe-signature`). Senza → indispensabile.

## Passo 9 · Le dipendenze e la prova dal vivo

- `npm audit --omit=dev`: riporta solo le vulnerabilità `high` e `critical`, con il pacchetto e la
  versione che le risolve. Non lanciare `npm audit fix`.
- **Prova dal vivo, solo con il sì esplicito e solo su un ambiente di prova:** con la chiave
  pubblica dell'app, prova a leggere le tabelle che al passo 2 sembravano aperte (una richiesta
  `GET`, senza scrivere niente). Se rispondono con dati che un visitatore non dovrebbe vedere, il
  problema è confermato. Riporta il numero di righe, mai il contenuto.

## Il rapporto

Il rapporto è la **scheda** di questo controllo: si conserva, e il giro successivo riparte da lì.
Scrivilo in `report-sicurezza-AAAA-MM-GG.md` nella cartella di lavoro (non committarlo: può dire
dove sono i punti deboli) e riassumilo in chat. Forma:

```markdown
# Controllo di sicurezza di base · [nome app] · [data] · giro [N]

- Codice controllato: commit [abc1234], [aggiornato con il ramo principale / indietro di N commit]
- Stack: [React + Vite + Supabase, versioni principali]
- Database: [progetto Supabase controllato, in sola lettura / solo migrazioni del repository]
- Versione pubblicata: [confrontata con il repository / non confrontata, perché ...]

## Rispetto al giro precedente (solo dal giro 2)
- Risolti: [...] · Ancora aperti: [...] · Nuovi: [...]

## Indispensabile (prima di andare in produzione)
1. **[Il problema in una riga]** · [verificato / da confermare]
   - Dove: [file:riga, oppure tabella / funzione]
   - Perché conta: [cosa può succedere, in parole semplici]
   - Come l'ho verificato: [cosa ho riletto o provato nella seconda passata]
   - Cosa dire all'AI: «[la richiesta da incollare a Lovable o allo strumento che usi]»

## Miglioria (quando c'è tempo)
1. ...

## Falsi allarmi scartati
- [sospetto] → [perché non è un problema: es. la funzione controlla già chi la chiama]

## Cosa non ho controllato
- [i passi saltati e perché]

## Prossimo passo
Correggi gli indispensabili, poi chiedimi di nuovo il controllo: confronto con questo rapporto,
finché un giro torna pulito.

Questo è un controllo di base, non un penetration test. Prima di un lancio importante, o se l'app
gestisce pagamenti o dati sensibili, fai rivedere il rapporto e l'app da uno sviluppatore.
```

Regole del rapporto:
- **Indispensabile** = fa perdere soldi, espone dati di persone, permette a qualcuno di fare cose
  che non dovrebbe, o blocca la pubblicazione. **Miglioria** = tutto il resto. Nel dubbio, miglioria.
- Se non trovi niente in una colonna, scrivilo: anche "nessun problema indispensabile trovato" è
  un risultato.
- **Da confermare** vuol dire che non sei riuscito a vederlo con i tuoi occhi nella seconda passata
  (ad esempio perché manca l'accesso al database): non presentarlo come certo.
- Un problema diventa **risolto** solo quando lo hai riverificato tu in questo giro, non perché il
  commit o l'AI dicono di averlo sistemato.
- Mai chiavi intere, mai dati di persone, mai istruzioni su come sfruttare un problema: solo dove
  sta e come si chiude.
