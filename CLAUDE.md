# Psicologi Monteverde — contesto per Claude Code

Sito dello studio di psicologia di Michele Argalia e Cecilia Belardelli
(Monteverde, Roma). Ricostruito da zero per lasciare Wix: zero costi di
hosting/abbonamento, gestibile in autonomia da loro tramite un pannello.

## Architettura, in breve

**Sito statico puro. Nessun build step, nessun framework, nessun bundler.**
HTML/CSS/JS scritti a mano, serviti così come sono.

- **Hosting**: Netlify (piano gratuito — vedi sezione Crediti sotto)
- **Codice**: questo repository GitHub (`micheleargalia-stack/psicologi-monteverde`)
- **Pannello di amministrazione**: Decap CMS (`/admin`), autenticazione via
  Netlify Identity + Git Gateway
- **Contenuti**: file JSON in `content/`, letti **a runtime** dal browser
  direttamente da `raw.githubusercontent.com` (non dal build) — vedi sotto
  perché questo è centrale nell'architettura
- **Email**: EmailJS (piano gratuito, 200 email/mese) per il modulo
  "Contattaci" — le credenziali (service ID, template ID, public key) sono
  già nel codice di `index.html`

## Il meccanismo centrale: contenuto letto a runtime

Le pagine HTML sono "gusci" quasi vuoti di testo statico di fallback. Ogni
pagina, al caricamento, fa fetch dei propri dati da
`https://raw.githubusercontent.com/micheleargalia-stack/psicologi-monteverde/main/content/...`
e sovrascrive il DOM via JavaScript. Il testo statico che si vede nell'HTML
è **solo un fallback** nel caso il fetch fallisca (offline, cache, ecc.).

Questo significa:
- **Un cambiamento nei file `content/*.json` non richiede una nuova
  pubblicazione Netlify per essere visibile** — il sito lo legge live da
  GitHub. Questo è il motivo per cui `netlify.toml` ha un `ignore` command
  che salta la build quando cambia solo `content/` (vedi sezione Crediti).
- Ogni fetch usa `?t=${Date.now()}` come cache-busting, perché
  `raw.githubusercontent.com` ha una cache CDN che altrimenti serve
  contenuto vecchio per alcuni minuti dopo un commit.
- `GITHUB_USER`, `GITHUB_REPO`, `GITHUB_BRANCH` sono hardcoded come
  costanti JS **ripetute in ogni singolo file HTML** (non c'è un file di
  configurazione condiviso, perché non c'è build step per iniettarlo).
  Se cambia lo username GitHub o il nome del repo, va aggiornato ovunque.

## Struttura del pannello (Decap CMS)

Config in `admin/config.yml`. Collezioni:
- `pagine` — file singoli: Homepage (`content/site/home.json`), profilo
  Michele, profilo Cecilia
- `servizi` — 4 pagine di servizio (file singoli)
- `dizionario` — voci del "Dizionario Psy" (cartella, entry ripetibili)
- `articoli` — blog (cartella, entry ripetibili, `published: false` di
  default — vedi Ruoli sotto)
- `documenti` — moduli scaricabili tipo consenso informato
- `pagine-libere` — pagine create liberamente dal pannello, con
  `parentLabel`/`parentLink` opzionali per un breadcrumb (non vera
  gerarchia annidata — Decap non la supporta nativamente)

**`admin/index.html` non è la pagina admin standard di Decap** — è stata
riscritta a mano per:
1. Filtrare quali collezioni un utente vede, in base al suo ruolo Netlify
   Identity (`admin` vede tutto, `editor` vede tutto tranne niente per ora
   in pratica, `autore` vede solo `articoli`) — la mappa è in
   `ROLE_ACCESS` dentro il file
2. Avere un vero modulo di login (email/password) invece del popup
   iframe di Netlify Identity, che si è dimostrato inaffidabile (falliva
   in silenzio, senza errori visibili)
3. Gestire il caso di sessione "stale" (ruolo assegnato dopo il login: il
   JWT in cache non lo sa finché non si rifà login)

**Nota sui ruoli**: non è una vera sicurezza server-side — Decap CMS non
supporta ruoli nativamente. È un filtro solo-UI. Un "autore" con accesso
diretto potrebbe tecnicamente aggirarlo. Per questo studio, con
collaboratori fidati, è considerato sufficiente.

**`admin/mappa.html`**: vista di sola lettura che elenca tutte le pagine
esistenti (pubblicate/bozza/nascoste, in menu o no) con link diretti alla
modifica di ognuna in Decap (`/admin/#/collections/{col}/entries/{slug}`).

## Crediti Netlify — attenzione

Piano gratuito: 300 crediti/mese, 15 crediti per ogni deploy di
produzione (~20 deploy/mese). **Ogni commit su `main` triggera un deploy**
a meno che `netlify.toml` non lo blocchi. `netlify.toml` attuale ignora i
deploy quando cambia solo `content/` (i contenuti non ne hanno bisogno,
vedi sopra) — se lo si modifica, verificare che questa regola resti
intatta, o i salvataggi dal pannello ricominceranno a consumare crediti
inutilmente.

Se si fa un force-push da locale, **verificare prima lo stato online**
(`git clone` e diff) per non sovrascrivere modifiche fatte nel frattempo
dal pannello — è già successo una volta in questo progetto.

## Sistema di design

- Colori: variabili CSS in `style.css` (`--sage`, `--sage-deep`, `--cotto`,
  `--ink`, `--stone`, `--paper`) — anche modificabili dal pannello
  (`theme.primary`/`theme.accent`/`theme.headingFont` in `home.json`,
  applicati via JS a runtime, non nel CSS statico)
- Font: Fraunces (titoli) + Inter (corpo) + IBM Plex Mono (etichette/eyebrow)
- Motivo grafico: due cerchi sovrapposti ("vesica piscis") — usato nel
  favicon e in un logo proposto. **Nota**: è un simbolo antico e molto
  diffuso, non originale in sé; solo la combinazione con testo/colori
  specifici del brand è potenzialmente registrabile.

## Cose lasciate volutamente aperte / in sospeso

- **Privacy Policy** (`content/pagine-libere/privacy-policy.json`):
  bozza scritta, `published: false`. Non attivarla senza revisione di un
  legale/DPO.
- **Tabella ISEE** in "Spazio a Tutti"
  (`content/site/servizio-psicoterapia-sociale.json`): placeholder, mancano
  le cifre reali.
- **3 articoli storici recuperati dal vecchio sito**: `published: false`
  di proposito, uno contiene dati non più attuali (segnalato nel testo
  stesso), un altro tratta un tema politico del 2016 — da rivedere prima
  di pubblicare.
- **Foto profilo**: campo pronto (`photo` in `michele.json`/`cecilia.json`),
  non ancora caricate.
- **Email di conferma automatica** (Auto-Reply EmailJS): non configurata,
  va fatta dalla dashboard EmailJS, non da codice.
- **`robots.txt`** blocca tutto (`Disallow: /`) — il sito non è ancora
  quello pubblico, il dominio vero (psicologimonteverde.com) è ancora su
  Wix. `_redirects` è già pronto con la mappa vecchi-URL → nuovi, da
  attivare al trasloco del dominio.
- **`sitemap.xml`** usa già l'indirizzo finale `psicologimonteverde.com` —
  da sottomettere a Search Console solo dopo il trasloco.

## Cosa NON fare

- Non aggiungere un build step (bundler, framework) senza discuterne:
  romperebbe il meccanismo di lettura runtime da GitHub raw, che è
  centrale nell'architettura.
- Non assumere che un ruolo Decap CMS sia una vera barriera di sicurezza.
- Non fare `git push --force` senza prima controllare lo stato online.
- Non aggiungere dipendenze esterne a pagamento senza confermare con
  Michele — il vincolo di partenza del progetto è costo zero.
