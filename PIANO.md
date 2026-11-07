# C60 — piano per dump, aggiornamenti sicuri e firmware nostro

> Piano ricavato da `../c60emu` (studio ed emulazione del C60-A82C) e da `../lt716` (dumper,
> aggiornatore del produttore, MorseOS, incidente 0.5.5). Redatto il 2026-11-07.
> La ricerca web/GitHub è in [`RICERCA.md`](RICERCA.md).

## 0. In breve

Sull'LT716 la sicurezza veniva da una **rotazione gestita dalla ROM**: firmware in `0x0`,
aggiornatore del produttore in `0x20000`. Il firmware azzera il proprio magic, la ROM avvia
l'aggiornatore, l'aggiornatore riscrive `0x0` e resta intatto come rete di sicurezza. Il dumper
è passato da questa strada, ha letto tutta la flash ed è tornato all'aggiornatore.

Il C60 **non ha questa struttura**. L'app sta a `0x14000`, sotto c'è un boot del produttore che
non abbiamo (`0x0–0x14000`). L'immagine OTA va prima in una zona di appoggio sulla flash esterna
(`0x374000`) e poi, al riavvio, il boot la attiva. Anche qui c'è una rotazione (app → appoggio →
boot → app), ma **non sappiamo ancora se il boot abbia un OTA proprio** quando l'app si blocca.

**Tutto il piano principale passa dal Bluetooth, senza aprire il braccialetto** (come sull'LT716):

1. come primo firmware modificato **non** un firmware nuovo, ma **il firmware originale con una
   piccola aggiunta** (comando di lettura flash), costruita con lime e provata nell'emulatore.
   Il ricevitore OTA originale resta intatto, quindi resta anche la strada per tornare indietro;
2. con quel dumper si leggono boot, SDK alto e flash esterna: Q3 si chiude leggendo il codice;
3. un firmware nostro (tipo MorseOS) dopo aver emulato anche il boot reale, con watchdog,
   contatore di riavvii e un'immagine di riserva che ricostruisce la rete di sicurezza dell'LT716.

L'SWire (fase 0) è una rete di sicurezza **facoltativa**, da preparare in parallelo. Non è un
prerequisito: serve solo se un firmware scritto da zero si blocca, come la MorseOS 0.5.5.

## 1. Il dispositivo (fatti consolidati in c60emu)

| Campo | Valore | Fonte |
|---|---|---|
| Nome BLE / MAC | `C60-A82C` / `A4:C1:38:9A:A8:2C` (OUI Telink) | `docs/knowledge/C60-A82C/C60-A82C.md` |
| Modello interno | `TB2C60T7` (BLE), `TC2C60T7` + `Anko43 721177` (firmware) | `firmware-layout.md` |
| Firmware | `V2.04` (il server lo chiama 3.00), 222260 B, SHA-256 `efc00343…` | idem |
| App | KeepHealth (`com.keephealth(pro).android`) | idem |
| SoC | Telink B85, probabile **TLSR8251** (512 KB interni, 32 KB SRAM), ISA TC32 | `docs/notes.md` |
| Display | ST7789-like 126×294 RGB565 su SPI | `docs/goals/hardware-reference.md` |
| Input | un solo tasto touch capacitivo (tap = pagina, lungo = conferma) | idem |
| Protocollo | servizio `FF00`: scrittura `FF02`, notifiche `FF01`, frame `[cmd][len LE][payload][chk]`, checksum `(sum·0x56)+0x5A` | `docs/ble-protocol.md` |
| OTA | `CMD_OTA_ENTER 0x0B` su FF02, poi Telink legacy su `…1911`/`…2b12` | `docs/goals/run-native-firmware-update.md` |

### Mappa della flash

| Range | Contenuto | Ce l'abbiamo? |
|---|---|---|
| `0x00000–0x14000` | boot/SDK basso del produttore (533 chiamate dall'app, es. `0x4ac`, `0x83c`, `0x87c`) | **no** |
| `0x14000–0x4a434` | app = `firmware_3.00.bin` | sì |
| `0x4a434–0x80000` | SDK alto (26 chiamate dall'app, es. `0x4edec`) | **no** |
| `0x80000+` | risorse su flash esterna: font a `0x857a2`, quadranti/sdfs | solo i quadranti |
| `0x374000` | zona di appoggio OTA (firmware **e** quadranti, scrittore comune `0x2aef8`) | modellata |

### Catena OTA nota (tutta nell'app, eseguita nativamente da c60emu)

```
FF02  0B 00 00 0C  -> handler 0x345ec: modalità update, schermata, flag UI (nessuna scrittura flash)
2b12  00 FF / 01 FF / dati / 02 FF  -> ota_data_write 0x1d448
      CRC-16 (init FFFF, poly A001) per pacchetto  0x3e394
      END: ultimo indice + complemento             0x1d850 -> 0x1d906
      ota_flash_store 0x1d27c: i primi 8 byte (marker di avvio) scritti a parte
      buffer RAM 4 KB 0x1d1c4 -> scrittore settori 0x2aef8 -> flash 0x374000 + offset
riavvio -> ??? (boot del produttore: verifica + copia 0x374000 -> 0x14000?)   <- Q3, non visto
```

**Quanto è certo che il ricevitore stia nell'app.** Prove *statiche ed emulate*: nell'app c'è la
tabella GATT del servizio OTA (UUID a `0x49b78`/`0x49b48`, `attribute_t` a `0x49d68` con callback
di scrittura `0x1d449` in-app), e c60emu esegue nativamente parsing, CRC, END e scrittura a
`0x374000`. L'handler ENTER `0x345ec` non riavvia: cambia solo modalità e schermata. È coerente
con una ricezione fatta dall'app in esecuzione. **Non c'è però una prova sul device**: le catture
in `sessions/` registrano solo il traffico FF02 (nessuna riga `2b12`), quindi un OTA firmware reale
non è mai stato osservato. Resta aperto anche se il boot abbia un *secondo* ricevitore (H2).
Primo controllo sul device, senza scrivere nulla: dopo `0B 00 00 0C` il C60 resta connesso con
lo stesso nome e la stessa tabella GATT (ricevitore nell'app), oppure si riavvia e annuncia altro
(boot)?

**UUID del servizio OTA da correggere**: `C60-A82C.md` (lettura GATT del device) e Gadgetbridge
riportano il servizio `…1912`; `ota_upload.py` e `protocol.py` cercano `…1911`. I "1911" nelle
catture sono nomi di file dei frame, non UUID. Quasi certamente `ota_upload.py` così com'è non
troverebbe il servizio: va corretto prima della fase 2.

### Formato immagine: uguale all'LT716 (verificato)

Controllato su `firmware_3.00.bin`: byte 6–7 = `5D 02` (`FLAG_FW_CHECK`, algoritmo 2), magic
`KNLT` a +8, lunghezza a +0x18 = `0x36434` = dimensione del file, **lunghezza ≡ 4 (mod 16)**,
ultimi 4 byte `a3 90 3a 18` = `~crc32` LE del resto (`0x183a90a3`). Quindi
`../lt716/firmware/telink_fw.py` produce già immagini nel formato che il C60 si aspetta, e il
controllo di `./lt716 ota` vale anche qui.

## 2. Cosa insegna lt716

**La catena che ha funzionato** (`../lt716/README.md`):

1. RE dell'APK → protocollo OTA e regole dell'immagine (padding a 16, `5D 02`, CRC32);
2. capire **come si aggiorna il firmware del produttore** (magic azzerato → ROM → aggiornatore);
3. un firmware **dumper** minimo (SDK e-lab in Docker), installato tramite l'aggiornatore: legge
   tutta la flash due volte, l'unica scrittura che si concede è il ritorno all'aggiornatore
   (`05 UPDT`), e rifiuta ogni scrittura fuori dalla zona consentita;
4. dump confrontati, poi scrittura mirata delle risorse con rilettura di ogni pagina;
5. solo dopo, un firmware nostro (MorseOS) con il ritorno all'aggiornatore sempre raggiungibile
   (comando BLE, tasto tenuto 20 s).

**Gli errori**, da non ripetere:

- un OTA fallito (immagine senza padding) ha lasciato partire l'aggiornatore, scambiato per il
  firmware in modalità OTA → **il firmware originale `V15381` è perso**. Lezione: *mai flashare
  senza un dump completo dell'originale*;
- risorse di una versione diversa → schermo bianco. Lezione: *codice e risorse vanno in coppia*;
- **incidente 0.5.5**: `pm_get_32k_tick()` chiamata dal ciclo principale aspetta per sempre,
  nessun watchdog, la ROM riavvia sempre la stessa immagine → band recuperabile solo via SWire.
  Le cinque regole di `INCIDENTE-0.5.5.md` (leggere il codice delle funzioni di libreria,
  watchdog con contatore di riavvii, codice nuovo non all'avvio, un rischio per flash, checklist)
  si applicano qui così come sono.

## 3. Differenze che contano

| | LT716 (TLSR8232) | C60 (TLSR8251, B85) |
|---|---|---|
| Flash | 512 KB interna, tutto lì | 512 KB interna + **flash esterna** (≥ 4 MB: `0x374000`) |
| Dove gira l'app | `0x0`, avviata dalla ROM | `0x14000`, dietro un **boot del produttore** a `0x0` |
| Rete di sicurezza | aggiornatore BLE completo in `0x20000` | **sconosciuta**: il boot ha 80 KB, abbastanza per un BLE minimo, ma non lo sappiamo |
| Chi scrive l'immagine | l'aggiornatore, direttamente in `0x0` | l'app, in appoggio su `0x374000`; poi il boot (Q3) |
| Codice condiviso | nessuno, l'immagine è autonoma | l'app chiama codice fisso in `0x0–0x14000` e `0x4a434–0x80000` |
| Ingresso OTA | `00 FF` diretto (o `12/19`) | `0B 00 00 0C` su FF02, poi `00 FF` |
| Interrupt | vettore dell'immagine in esecuzione | il vettore a `0x10` è del **boot**: come arriva l'IRQ all'app (`0x14010 → 0x141a0`) va capito |
| Emulatore | no | **sì**: c60emu esegue l'app originale, ricezione OTA compresa |
| Firmware originale | perso | disponibile (server KeepHealth) e ricostruibile byte per byte con lime |

Conseguenze:

- **non si tocca mai `0x0–0x14000`**: se il boot si rompe, non c'è una ROM che ripiega su
  un'altra copia (il secondo indirizzo di avvio della B85, `0x20000`, cade dentro l'app);
- un'app nostra deve stare nello spazio dell'app (≤ `0x4a434`) **senza** sovrascrivere l'SDK
  alto, altrimenti il firmware originale non si può più rimettere solo via OTA;
- un firmware scritto da zero deve gestire init, IRQ e BLE senza appoggiarsi al codice fisso,
  oppure dipendere dalle sue ABI. Entrambe le strade richiedono di conoscere il boot.

## 4. La domanda centrale: cosa fa il boot (Q3)

Ipotesi da distinguere, in ordine di quanto sarebbero comode:

- **H2**: il boot ha un suo OTA BLE e parte quando l'app è invalida o un flag lo chiede
  (equivalente dell'aggiornatore LT716). Rete di sicurezza presente.
- **H1**: il boot copia `0x374000 → 0x14000` solo se l'immagine in appoggio ha marker e CRC32
  validi, poi salta all'app. Un'app che si blocca = band bloccata, ma un OTA interrotto non fa
  danni.
- **H3**: il boot salta all'app senza controlli. Come H1, ma più fragile.

Cosa si può sapere **senza rischi**:

1. **Dall'app (statico, lime)**: seguire `0x1d906` dopo l'END. Quale flag scrive, dove (flash
   interna? esterna? registro analogico che sopravvive al riavvio?), come riavvia, cosa succede
   dei primi 8 byte salvati a parte. È il contratto app→boot visto dal lato che abbiamo.
2. **Dall'app**: quali funzioni in `0x0–0x14000` chiama (533 siti) e con che argomenti. Se fra
   queste ci sono primitive BLE/link layer, il boot contiene lo stack (indizio per H2).
3. **Dalla stringa `UPLOAD.`** (`0x35520`) e dal controllo del byte `0xa5` all'entry `0x140a8`:
   capire chi li usa.
4. **Sul device, senza flash**: dopo `0B 00 00 0C` (ENTER), *senza mandare dati*, aspettare il
   timeout come con `lt716 ota --enter-updater` e guardare se compare un servizio o un nome BLE
   diverso. Sull'LT716 è così che si è scoperto l'aggiornatore.

5. **Dall'SDK Telink** (vedi `RICERCA.md` §D): il `boot.link` B85 ha le opzioni `__BOOT_LOADER_EN`
   e `__FW_START_BY_LEGACY_BOOTLOADER_EN`, cioè un bootloader in flash con l'app a un offset non
   nullo. È lo schema più probabile del C60. Va letto il codice SDK del bootloader e
   di `ota_set_flag()` in quella configurazione, che lascia valido il banco vecchio, per
   farsi un'idea di cosa aspettarsi dal boot.

Cosa la chiude **definitivamente**: il dump di `0x0–0x80000` (fasi 1 o 3).

## 5. Piano a fasi

Ogni fase ha un criterio di uscita. Non si passa alla successiva senza.

### Fase 0 — Recupero hardware (facoltativa, in parallelo; non blocca le fasi 2–3)

- Aprire il C60 (vedi `RICERCA.md` per teardown noti), fotografare la scheda, identificare
  SoC (marcatura), flash esterna (marcatura → JEDEC atteso), pad **SWS**, GND, VBAT/3V3.
- Programmatore SWire: abbiamo già la Blue Pill con `USB2Swire` (`../lt716/firmware/vendor/
  FitPro-LT715-TLSR8232/binaries/`); in alternativa `TlsrComSwireWriter` di pvvx con un adattatore
  USB-UART. Il B85 è supportato da entrambi (da verificare sul nostro esemplare).
- Prova di sola lettura via SWire della flash interna (512 KB).
- **Uscita**: lettura SWire riuscita due volte con lo stesso hash. Da quel momento ogni brick
  della flash interna è recuperabile.

> Se il C60 non si apre senza danni, la fase 0 si salta: le fasi 2–5 restano tutte via Bluetooth.
> Il firmware scritto da zero (fase 6) aspetta solo che le fasi 4–5 abbiano dimostrato la strada
> di ritorno con il boot emulato.

### Fase 1 — Dump completo via SWire (se la fase 0 è riuscita)

- `0x00000–0x80000` interni: boot, app, SDK alto, calibrazione, MAC, dati utente.
- La flash esterna non si legge direttamente via SWire: o si comanda il master SPI scrivendo i
  registri via SWire (pin e CS ricavati dal firmware), o la si legge col dumper della fase 3.
- Backup in `files/firmware/` (fuori da git), con hash e data.
- **Uscita**: boot e SDK alto disponibili; Q3 si risolve leggendo il codice.

### Fase 2 — Validare la catena OTA reale con rischio minimo

Prima nell'emulatore (`c60emu`: `tools/ota_native_session.py`, `tools/ota_apply_and_verify.py`,
`tests/test_ota_*.py`), poi sul device:

1. **Ri-flash dell'immagine identica** (`firmware_3.00.bin`, V2.04) con `ota_upload.py` /
   futuro `c60 ota`. Nessun cambiamento di codice: prova solo il trasporto (ENTER, flusso,
   controllo ogni 8 pacchetti, END, riavvio). Misurare durata e comportamento a schermo.
2. **Variante lime `v205`** (1 byte: `V2.04 → V2.05` a `0x3551c`), passata da `telink_fw.py`
   (o con un controllo che padding/CRC restino validi). Verificare `CMD_GET_DEVICE` e la pagina
   info. Poi rimettere la V2.04.
3. I test negativi (CRC sbagliato, END sbagliato, immagine troncata, disconnessione a metà) si
   fanno **solo nell'emulatore** finché la fase 0 non è chiusa; sul device al massimo un OTA
   interrotto a metà, con SWire pronto, per osservare cosa fa davvero il boot.
- **Uscita**: due OTA riusciti (identico e v205) e ritorno alla V2.04; durata e comportamento
  annotati nel diario.

### Fase 3 — "Dumper dentro il firmware originale" (via lime)

Invece di un dumper scritto da zero come sull'LT716, si aggiunge **al firmware originale** un
comando nascosto. Vantaggi decisivi:

- il ricevitore OTA originale resta intatto: **la strada per tornare indietro non cambia**;
- init, IRQ, BLE e boot restano quelli del produttore: nessuna ipotesi su Q3;
- c60emu esegue il firmware originale, quindi la patch si prova **nativamente** prima di
  flasharla (iniezione di frame nel dispatcher RX `0x3447c`, come `tools/native_dial_transfer.py`);
- lime garantisce che l'immagine differisca dalla base solo nei byte voluti.

Precedenti pubblici: `pvvx/TLSR825x_OTA_Flash_Hacker` (dumper installato via OTA Telink, con
ritorno al firmware originale) e `HackSucks/OpenWearOTA` (firmware custom via OTA del produttore,
stessi UUID del KeepHealth). Entrambi presuppongono la rotazione ROM `0x0`/`0x20000`/`0x40000`, che
sul C60 non c'è: si prendono come riferimento, non si usano così come sono.

Contenuto:

- una voce nuova nella tabella comandi a `0x49ec0` (28 record, verificare che ci sia spazio o un
  comando inutile da riutilizzare, es. uno di test/factory), con handler in uno spazio libero
  dell'app (le zone `00`/`FF` vanno trovate con `analyze_firmware_contents.py`);
- `READ addr len`: per `addr < 0x80000` lettura della flash interna (primitive già usate dal
  firmware), per `addr ≥ 0x80000` lettura SPI della flash esterna (la stessa usata per i font a
  `0x864ae`); risposta a blocchi su FF01 con il frame e il checksum C60;
- `INFO`: MID della flash interna, JEDEC dell'esterna, registro di avvio `0x63e`, header di
  `0x0` e `0x14000`;
- **nessun comando di scrittura** in questa versione.

Lato PC: uno script `c60` sul modello di `../lt716/lt716` (BlueZ via D-Bus, chunk da 20 byte,
riprova dei blocchi mancanti) con `info`, `dump int|ext [file]`, `ota <img> [--yes]`.

- **Uscita**: dump interno ed esterno letti due volte, identici; flash esterna completa (font,
  quadranti, appoggio OTA, eventuali altre immagini); **boot e SDK alto in mano** anche senza
  SWire.

### Fase 4 — Chiudere Q3 nell'emulatore

- Caricare il boot reale (`0x0–0x14000`) e l'SDK alto in c60emu, togliere gli stub
  (`0x4ac`, `0x4edec`, …) e far partire l'emulazione **dal reset vero** (`0x0`).
- Eseguire nativamente il passo ACTIVATE oggi modellato in `ota_apply_and_verify.py`.
- Matrice no-brick (`run-native-firmware-update.md` §9) completa: per ogni immagine (buona,
  CRC errato, troncata, marker mancante, app che si blocca) il verdetto dell'emulatore = quello
  del device. Controllare a campione sul device i casi sicuri.
- Stabilire H1/H2/H3 e se esiste un modo per far ripartire il boot in modalità ricezione.
- **Uscita**: il contratto del boot è scritto in lime (`lime/rom/…`), testato, e c'è una
  procedura documentata per tornare al firmware originale da qualunque stato raggiungibile via BLE.

### Fase 5 — Ricostruire la "rotazione" dell'LT716 sul C60

L'LT716 aveva un aggiornatore che non veniva mai toccato. Sul C60 lo si costruisce così (i
dettagli dipendono dall'esito della fase 4):

- **immagine d'oro**: il firmware originale (o la variante dumper della fase 3) tenuto in una
  zona libera della flash esterna, mai sovrascritto;
- **ritorno all'immagine d'oro**: un comando BLE (`UPDT` come `05 UPDT`) e il tasto tenuto 20 s
  copiano l'immagine d'oro in `0x374000` e scrivono il flag di attivazione che il boot si aspetta
  (lo stesso che scrive `0x1d906`), poi riavviano. Il boot riporta il firmware originale;
- se H2 è vera, basta invece far partire l'aggiornatore del boot, come sull'LT716.

Da provare **da sola**, prima di qualunque altra novità: un'immagine che fa solo questo, flashata,
usata per tornare all'originale, verificata.

### Fase 6 — Firmware nostro

Solo con fasi 4–5 chiuse:

- base: SDK Telink B85 (multi-connection o single-connection, open source) in Docker come
  `../lt716/firmware/build.sh`; link a `0x14000`, header Telink, dimensione ≤ spazio app,
  `telink_fw.py` per padding e CRC;
- moduli e manifest come MorseOS (`bundle.ini`, `module.ini`, `tools/bundle.py`), riusando
  `lib/morse`, `core/ui` (adattando display e risoluzione 126×294);
- **dalla prima versione**: watchdog hardware azzerato solo nel ciclo principale, contatore di
  riavvii che sopravvive al reset (registro analogico o settore dedicato), 3 riavvii di fila →
  ritorno all'immagine d'oro, ritorno manuale col tasto e con BLE sempre raggiungibili prima di
  qualunque codice nuovo;
- driver ricavati da c60emu: display (CS `0x800583` bit 0, D/C `0x80059B` bit 3, SPI
  `0x800008`), tasto (`0x800580` bit 1 + `0x800588` bit 5), sensori da identificare.

## 6. Regole prima di ogni flash (adattate da `INCIDENTE-0.5.5.md`)

1. Esiste un dump completo e verificato (hash) dello stato attuale della band.
2. L'immagine è passata dal controllo del formato (`5D 02`, lunghezza ≡ 4 mod 16, `~crc32`,
   dimensione ≤ spazio app, header a `+0x18` coerente).
3. L'immagine è stata eseguita in c60emu fino al ciclo principale e ha risposto ai comandi BLE
   (incluso l'ENTER OTA, cioè la strada per tornare indietro).
4. Elenco scritto di cosa cambia e di cosa può bloccare l'avvio; un solo cambiamento rischioso.
5. Funzioni di libreria/SDK nuove: disassemblate prima, nessuna attesa senza limite.
6. Codice nuovo spento all'avvio, attivato da comando BLE.
7. Mai `0x0–0x14000`, mai `0x4a434–0x80000`, mai l'immagine d'oro.
8. Senza SWire: patch lime del firmware originale (il ricevitore OTA resta) sempre; firmware da
   zero solo con la strada di ritorno della fase 5 provata.

## 7. Cosa si riusa

| Da | Cosa | Per |
|---|---|---|
| `../lt716/firmware/telink_fw.py` | padding, `5D 02`, `~crc32` | ogni immagine (formato verificato identico) |
| `../lt716/lt716` | client BlueZ D-Bus, OTA Telink, protocollo dumper, riprova blocchi | script `c60` |
| `../lt716/firmware/build.sh`, `tools/bundle.py` | build in Docker, manifest moduli | fase 6 |
| `../lt716/firmware/morseos/` | UI, Morse, regole watchdog/emergenza | fase 6 |
| `../c60emu/src/c60emu/protocol.py` | frame C60, checksum, builder OTA con CRC16 | script `c60` |
| `../c60emu/docs/knowledge/C60-A82C/ota_upload.py` | client OTA reale ricavato dall'app | fase 2 |
| `../c60emu/lime/` (`version.lime`, `ble.lime`) | build byte per byte, varianti, funzioni OTA | fasi 2–3 |
| `../c60emu/tools/ota_native_session.py`, `ota_apply_and_verify.py`, `native_dial_transfer.py` | prova nativa della ricezione e dello store | fasi 2–4 |
| `../c60emu/sdk/Telink-BDT-Linux-X64-2.2.1` | BDT (con EVK Telink) per SWire | fasi 0–1 |

## 8. Domande aperte

- Q3: attivazione dell'immagine in appoggio, e se il boot ha un OTA proprio (H1/H2/H3).
- Come arriva l'IRQ all'app (vettore del boot a `0x10` → `0x14010`?).
- Dimensione e modello della flash esterna; zone libere per l'immagine d'oro.
- Pin SPI della flash esterna rispetto al display (c60emu indica `0x800583` bit 0 sia per il CS
  del display sia per la scrittura flash: da chiarire).
- SoC esatto (8251/8253/8258) dalla marcatura.
- Se `CMD_OTA_ENTER` senza dati, al timeout, lascia il device in uno stato diverso (servizio o
  nome BLE nuovo) come sull'LT716.
