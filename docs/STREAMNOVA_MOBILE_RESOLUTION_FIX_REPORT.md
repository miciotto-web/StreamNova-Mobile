# StreamNova Mobile — Fix selezione risoluzione video (HTTP + Debrid)

**Data:** 08/11/2026 • **Branch:** `feature/streamnova-mobile-phase2` • **APK installato su S22:** `mobile-release.apk` v1.1.1 (versionCode 3, update 19:53)

---

## 1. Problemi riportati

| # | Sintomo | Modalità |
|---|---------|----------|
| 1 | La risoluzione scelta è sempre la più bassa (video 1080 → seleziona 480p) | HTTP (web scraper) |
| 2 | Selezionata una fonte 1080p il player mostra 720p | Debrid (TorBox) |

---

## 2. Analisi delle cause (prima dei fix)

### Problema 1 — HTTP parte sempre dal profilo più basso

Il player mobile (`mobile/.../screens/PlayerScreen.kt` → `buildMobilePlayer`) non impostava alcun
`BandwidthMeter`. ExoPlayer usa quindi il `DefaultBandwidthMeter` di default con **stima iniziale
~1 Mbps**: la selezione adattiva (`AdaptiveTrackSelection`) dei flussi HLS dei provider HTTP
partiva di conseguenza dalla rendition più bassa (480p) e risaliva molto lentamente
(isteresi `qualityIncreaseMinBufferMs = 10 s` su ogni gradino).

La app **TV** ha già questo meccanismo corretto: `NuvioExoPlayerPerformanceHelper.buildBandwidthMeter()`
crea un meter con **stima iniziale 50 Mbps** ("avvio al profilo a bitrate massimo") impostato con
`ExoPlayer.Builder.setBandwidthMeter()`. Il mobile non replicava il port.

### Problema 2 — Debrid: card 1080p → player 720p

Tre difetti combinati nella catena di sblocco TorBox:

1. **Il ViewModel mobile non passava stagione/episodio**: `MobilePlayerViewModel.resolvePlayable`
   chiamava `TorBoxRepository.resolveInfoHash(infoHash, fileIdx)` senza `season`/`episode`
   (la TV li passa esplicitamente in tutti i punti). Per un season pack lo sblocco restava
   quindi senza il vincolo SxxExx.
2. **`unlockInternal` ignorava `fileIdx`** e per i film usava il nome del **primo file** del
   torrent (`entry.files.firstOrNull()?.name`): in un torrent multi-file (720p+1080p) il file
   scelto era arbitrario rispetto alla card selezionata.
3. **La cache dei link era indicizzata solo per hash** (`linkCacheKey = hash`): lo stesso torrent
   esposto da due card con risoluzioni diverse (720p e 1080p, stesso infoHash, `fileIdx`
   differenti) condivideva il link per 15 minuti — selezionando la card 1080p si otteneva il
   link 720p sbloccato in precedenza. (Sintomo esatto riportato.)

Inoltre `TorBoxFileSelector`, davanti a **più file per lo stesso episodio** (pack multi-qualità),
restituiva `singleOrNull()` = `null` e l'intero sblocco falliva.

### Problema 3 (emerso durante la verifica) — crash del selettore sorgenti

Aprendo "The Pitt" in modalità Debrid l'app si chiudeva:
`IllegalArgumentException: Key "Torrentio 🧲 1080p-null-77b4e714…" was already used`.
Due **installazioni** dello stesso addon (Torrentio globale + Torrentio IT) restituiscono lo
stesso torrent: la chiave del `LazyColumn` del selettore
(`serverName-streamUrl-infoHash`) collideva. Fix necessario per poter usare il flusso Debrid.

---

## 3. Fix applicati

### 3.1 HTTP — BandwidthMeter Nuvio sul player mobile
`mobile/src/main/java/com/aistudio/streamnova/mobile/ui/screens/PlayerScreen.kt` (`buildMobilePlayer`)

```kotlin
val bandwidthMeter = NuvioExoPlayerPerformanceHelper.buildBandwidthMeter(context) // stima iniziale 50 Mbps
ExoPlayer.Builder(context, renderersFactory)
  ...
  .setBandwidthMeter(bandwidthMeter)
```

Parità con la TV: la selezione automatica HLS parte dal profilo più alto e la discesa adattiva
resta attiva se la banda reale non regge. Aggiunta diagnostica log-only
(`MobilePlayer: BandwidthMeter mobile: stima iniziale=…`, `Selezione video: WxH, banda stimata=…`).

### 3.2 Debrid — selezione file con regole di precedenza corrette

**`core/data/.../torbox/TorBoxResolver.kt`**
- Nuovo parametro `fileIdx` con queste regole di precedenza:
  - **con stagione/episodio noti → il match SxxExx è autorevole** (stessa regola TV): il fileIdx
    NON viene usato, perché l'indice del metadata torrent dell'addon **non coincide** con
    l'ordinamento dell'array file di TorBox (verificato sul campo: Torrentio dichiara
    `fileIdx=6` per il file 2x01 del pack S02 Pir8, ma l'indice 6 di TorBox è il file 2x07);
  - **senza episodio (film) → `fileIdx` è il segnale migliore** della card scelta (es. variante
    1080p di un torrent multi-file) e viene onorato se punta a un video reale
    (`files[fileIdx].isPlayableVideo()`), altrimenti fallback sul selettore storico.
- Estratto il predicato `isPlayableVideo()` (mime `video/` oppure estensione video) usato sia
  per la validazione del fileIdx sia per il log dei file video.

**`core/data/.../torbox/TorBoxFileSelector.kt`**
- Più file per lo **stesso episodio** (pack multi-qualità 720p+1080p): vince **il file più
  grande** (qualità più alta; scarta di fatto sample/trailer) invece di fallire lo sblocco.

**`core/data/.../torbox/TorBoxRepository.kt`**
- `linkCacheKey(infoHash, season, episode, fileIdx)`: la chiave della cache link include ora il
  **fileIdx** (`hash:f6`, `hash:s2e1:f6`) — le card 720p/1080p dello stesso torrent hanno voci
  distinte; senza fileIdx la chiave resta quella storica.
- `unlockInternal` passa `fileIdx` a `TorBoxResolver.resolve`.
- `invalidateResolvedLink` invalida tutte le varianti fileIdx dello stesso torrent/episodio.

### 3.3 Debrid — il ViewModel mobile passa stagione/episodio

`mobile/.../ui/MobilePlayerViewModel.kt`

```kotlin
private suspend fun resolvePlayable(source: StreamSource, season: Int?, episode: Int?): StreamSource? {
  ...
  TorBoxRepository.resolveInfoHash(
    infoHash = infoHash, fileIdx = source.fileIdx, season = season, episode = episode)
}
```
- `prepare()` passa l'episodio risolto (`resolvedEpisode`);
- `selectSource()` passa l'episodio corrente dello stato (stesso criterio della TV);
- la cache link torna ad essere per-episodio anche sul mobile.

### 3.4 Crash selettore sorgenti — chiavi univoche

`mobile/.../screens/PlayerScreen.kt`
- Nuovo helper `uniqueSourceKeys(sources)`: chiave riga = `serverName-streamUrl-infoHash-fileIdx-addonInstallId`,
  con disambiguazione progressiva (`…#2`) per i duplicati esatti residui. Il `LazyColumn` del
  selettore usa `itemsIndexed` con queste chiavi.

---

## 4. Test di regressione

| File | Contenuto |
|------|-----------|
| `core/data/src/test/.../torbox/TorBoxRepositoryFileIdxTest.kt` (**nuovo, 8 test**) | fileIdx onorato per film anche con primo file 720p; card 720p/1080p non condividono il link; fileIdx invalido/non-video → fallback; SxxExx vince sul fileIdx (caso field: pack 12 episodi, fileIdx=6 per 2x01); a parità di episodio vince il più grande (1080p); chiavi cache distinte per fileIdx |
| `mobile/src/test/.../MobileSourceKeysTest.kt` (**nuovo, 4 test**) | stesso torrent da due installazioni → chiavi distinte; duplicato esatto disambiguato; file diversi distinti; sorgenti HTTP duplicate disambiguate |

**Esiti**

```
:core:data:testDebugUnitTest  → 1561+ test, 0 failure (incl. TorBox: 18 ok)
:mobile:testDebugUnitTest     → tutti ok (incl. MobileSourceKeysTest 4/4)
:app:testDebugUnitTest        → 849 test, 0 failure (la modifica core non impatta la TV)
```

---

## 5. Verifica pratica su Samsung S22 (SM-S908B, Wi-Fi ADB)

APK **release firmato** con il keystore originale dell'installazione
(`C:\Users\miciuss\StreamNovaSigning\my-upload-key.jks`, alias `upload`, cert
`CN=Fabio Lo Reto` — SHA-256 `c9795461…3f25e0`): aggiornamento in-place, **dati e impostazioni
preservati** (script: `.artifacts/build_mobile_release.ps1`, password da credenziali DPAPI,
mai stampate).

### HTTP (modalità Web Streaming) — PRIMA vs DOPO
| | Build precedente | Build con fix |
|---|---|---|
| Prima selezione automatica | 480p (stima default ~1 Mbps) | **1280x720 = variante più alta disponibile** (master di quella sorgente: 480p/720p) |
| Logcat | — | `BandwidthMeter mobile: stima iniziale=50 Mbps` → `Selezione video: 1280x720, banda stimata=263 Mbps` |
| Riga Qualità nel player | "Auto • 480p" | **"Auto • 720p"** (stabile) |

### Debrid (modalità TorBox) — card 1080p su season pack
```
1. DATI STREAM DALL'ADDON (resolve): infoHash=77b4e714…, season=2, episode=1, fileIdx=6
3. FILE SELEZIONATO: file.id=10, nome='The.Pitt.2x01.Prima.Ora.07.00-08.00.ITA.ENG.1080p.DLMux.AAC.x265-Pir8.mkv'
4. URL FINALE: https://nexus-326.ceur.tb-cdn.st/dld/…
```
- La card scelta (IlCorsaroViola 1080p) sblocca **il file 1080p dell'episodio richiesto (2x01)**:
  con stagione/episodio il match SxxExx prevale sul `fileIdx=6` che con la logica precedente
  selezionava il file 2x07.
- Riga Qualità nel player: **"Auto • 1080p"** ✔ (prima: file sbagliato/risoluzione più bassa).
- Selettore sorgenti: 49 sorgenti renderizzate senza crash (fix chiavi duplicate).

Nota di stato del device: durante le prove la modalità motore del telefono risultava
"Web Streaming Gratuito (HTTP Scraper)"; è stata **ripristinata a "Debrid TorBox"** a fine
verifica. Volume media momentaneamente a muto per i test, ripristinato a 10/15.

---

## 6. File modificati

| File | Modifica |
|------|----------|
| `mobile/.../ui/screens/PlayerScreen.kt` | BandwidthMeter Nuvio + diagnostica; `uniqueSourceKeys` + `itemsIndexed` nel selettore |
| `mobile/.../ui/MobilePlayerViewModel.kt` | `resolvePlayable(source, season, episode)`; chiamanti aggiornati |
| `core/data/.../torbox/TorBoxResolver.kt` | Parametro `fileIdx` con precedenza corretta; `isPlayableVideo()` |
| `core/data/.../torbox/TorBoxFileSelector.kt` | Match multiplo stesso episodio → file più grande |
| `core/data/.../torbox/TorBoxRepository.kt` | `linkCacheKey` con fileIdx; `unlockInternal` passa fileIdx; `invalidateResolvedLink` per-varianti |
| Test | `TorBoxRepositoryFileIdxTest.kt` (8), `MobileSourceKeysTest.kt` (4) |

## 7. Strumenti lasciati in `.artifacts/`
- `build_mobile_release.ps1` — build release mobile firmata con credenziali DPAPI (keystore `my-upload-key.jks`, alias `upload`).
- Screenshot di verifica: `s22_quality_1080p*`, `s22_debrid_final*`, `s22_final_quality*`, dump UI `ui_fin.xml`, `ui_final_quality.xml`.
