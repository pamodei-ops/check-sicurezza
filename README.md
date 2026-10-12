# check-sicurezza

**Un controllo di base della sicurezza per le app costruite con l'AI.**
Una skill per Claude Code, Codex e gli altri agenti di coding, pensata per lo stack
**React + Vite + Supabase**: quello che usano Lovable e strumenti simili.

Gli chiedi *«controlla la sicurezza della mia app»* e ti restituisce un rapporto in due colonne:

- **Indispensabile**: le cose da sistemare prima di andare in produzione;
- **Miglioria**: tutto il resto, quando c'è tempo.

Per ogni problema ti dice dove sta, perché conta (in parole semplici) e **la frase da dare a
Lovable**, o allo strumento che usi, per correggerlo.

> Nata per il workshop «Il metodo che usiamo in Marketers per costruire app e tool con l'AI»,
> Marketers World 2026. È un controllo di base, non un penetration test: prima di un lancio
> importante, o se l'app gestisce pagamenti o dati sensibili, fai rivedere tutto da uno sviluppatore.

---

## Cosa controlla

| | Controllo | In parole semplici |
|---|---|---|
| 🔑 | Chiavi segrete | Chiavi finite nel codice del sito, che chiunque può leggere aprendo la pagina |
| 🛡️ | Tabelle | Chi può leggere e scrivere cosa: regole di accesso (RLS) e permessi, tabella per tabella |
| ⚙️ | Funzioni del database | Funzioni che chiunque può chiamare senza che controllino chi le sta chiamando |
| ☁️ | Edge function | Funzioni sul server che usano la chiave "di servizio" senza controllare chi chiama |
| 👀 | Controlli finti | Pagine "da admin" protette solo nascondendo un pulsante |
| 🔀 | Accesso | Redirect dopo il login verso indirizzi non controllati |
| 📁 | File caricati | Documenti o foto di persone in cartelle pubbliche |
| 💳 | Pagamenti | Prezzi decisi dal browser, webhook senza verifica della firma |
| 📦 | Dipendenze | Librerie con vulnerabilità gravi note |

## Come lavora

1. **Parte dalla versione giusta** del codice: se il tuo è indietro rispetto a GitHub, te lo dice.
2. **Due passate**: prima trova i sospetti, poi li ricontrolla uno per uno, come un revisore che non si
   fida. Nel rapporto ogni problema è *verificato* o *da confermare*, e i falsi allarmi scartati sono
   elencati a parte.
3. **Guarda quello che gira davvero**: se può, confronta quello che è pubblicato con quello che c'è
   nel codice.
4. **Un giro non basta**: il rapporto è datato e si conserva. Dopo le correzioni gli chiedi di nuovo il
   controllo e ti dice cosa è *risolto*, cosa è *ancora aperto* e cosa è *nuovo*. Finché un giro torna
   pulito.

**Solo lettura.** Non modifica file, non fa commit, non tocca il database, non attacca l'app in
produzione e non scrive chiavi o dati di persone nel rapporto.

---

## Prima di iniziare

La skill lavora sul **codice** della tua app, quindi serve una copia sul tuo computer.

1. **Collega il progetto a GitHub.** Su Lovable: *Connect to GitHub* nelle impostazioni del progetto.
2. **Scarica il codice** in una cartella: `git clone https://github.com/<tuo-account>/<tua-app>.git`
3. **Un agente di coding** installato: [Claude Code](https://code.claude.com/docs/en/quickstart),
   [Codex](https://developers.openai.com/codex) o un altro che legge le skill.
4. *(Facoltativo, consigliato)* **Accesso a Supabase in sola lettura**, tramite l'MCP di Supabase o la
   CLI collegata al progetto. Senza, la skill controlla solo le migrazioni nel repository e te lo dice
   nel rapporto.

---

## Installazione

Il modo più semplice è clonare questo repository **dentro la cartella delle skill** del tuo agente.
Il nome della cartella diventa il nome della skill: lascialo `check-sicurezza`.

### Claude Code

**Per tutti i tuoi progetti** (consigliato):

```bash
# macOS / Linux
git clone https://github.com/pamodei-ops/check-sicurezza.git ~/.claude/skills/check-sicurezza
```

```powershell
# Windows (PowerShell)
git clone https://github.com/pamodei-ops/check-sicurezza.git "$env:USERPROFILE\.claude\skills\check-sicurezza"
```

**Solo per un progetto**: dalla cartella del progetto,

```bash
git clone https://github.com/pamodei-ops/check-sicurezza.git .claude/skills/check-sicurezza
```

Claude Code vede le skill nuove senza riavvio. Se la cartella `skills` non esisteva prima di aprire
la sessione, chiudi e riapri Claude Code. Documentazione: [code.claude.com/docs/en/skills](https://code.claude.com/docs/en/skills).

### Codex

```bash
# macOS / Linux
git clone https://github.com/pamodei-ops/check-sicurezza.git ~/.codex/skills/check-sicurezza
```

```powershell
# Windows (PowerShell)
git clone https://github.com/pamodei-ops/check-sicurezza.git "$env:USERPROFILE\.codex\skills\check-sicurezza"
```

Documentazione: [developers.openai.com/codex/skills](https://developers.openai.com/codex/skills).

### Altri agenti (Gemini CLI, Cursor e simili)

La skill segue lo standard aperto **Agent Skills** (`SKILL.md` con nome e descrizione in testa).
Clona il repository nella cartella delle skill del tuo strumento, con il nome `check-sicurezza`:
la trovi nella sua documentazione.

### Senza git

1. Crea la cartella `check-sicurezza` dentro la cartella delle skill del tuo agente (vedi sopra).
2. Scarica [`SKILL.md`](https://raw.githubusercontent.com/pamodei-ops/check-sicurezza/main/SKILL.md)
   e mettilo dentro.

### Se il tuo strumento non legge le skill

Apri [`SKILL.md`](SKILL.md), copia tutto il testo e incollalo come primo messaggio all'agente,
aggiungendo in fondo: *«Segui queste istruzioni sul progetto in questa cartella.»*

---

## Come si usa

1. Apri l'agente **nella cartella del progetto** (es. `cd la-mia-app` e poi `claude` o `codex`).
2. Scrivi:

   > controlla la sicurezza della mia app

   oppure, su Claude Code, il comando `/check-sicurezza`.
3. Leggi il rapporto: è nel file `report-sicurezza-AAAA-MM-GG.md` e riassunto in chat.
   **Non committarlo**: dice dove sono i punti deboli.
4. Correggi gli **indispensabili**, uno alla volta, dando a Lovable le frasi del rapporto.
5. Poi chiedi:

   > ricontrolla dopo le correzioni

   La skill rilegge il rapporto precedente e ti dice cosa è risolto e cosa no. Si va avanti
   **finché un giro torna pulito**.

### Il rapporto, in breve

```markdown
# Controllo di sicurezza di base · La mia app · 2026-10-16 · giro 1

## Indispensabile (prima di andare in produzione)
1. **La tabella dei profili si legge senza essere loggati** · verificato
   - Dove: tabella `profiles`
   - Perché conta: chiunque abbia il link dell'app può scaricare nomi ed email
   - Cosa dire all'AI: «Attiva le regole di accesso sulla tabella profiles: ognuno legge solo il suo profilo»

## Miglioria (quando c'è tempo)
...

## Falsi allarmi scartati
...
```

---

## Aggiornare

```bash
git -C ~/.claude/skills/check-sicurezza pull     # Claude Code
git -C ~/.codex/skills/check-sicurezza pull      # Codex
```

## Limiti

- È un **controllo di base**: trova gli errori più comuni delle app costruite con l'AI, non tutti.
- Pensato per **React + Vite + Supabase**. Su altri stack alcuni passi non si applicano: la skill lo
  scrive nella sezione *«Cosa non ho controllato»*.
- Le prove "dal vivo" le fa **solo se glielo chiedi esplicitamente** e solo su un ambiente di prova,
  mai sull'app in produzione.

## Licenza

[MIT](LICENSE): usala, modificala, condividila.
Di [Pierdomenico Amodei](https://github.com/pamodei-ops), Marketers.
