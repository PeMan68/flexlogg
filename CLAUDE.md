# Flexlogg – projektanteckningar för Claude

Personlig tidloggare (PWA) för iPhone. En lärare loggar arbetstid i kategorier och får förslag på
in-/utstämplingar att föra över till jobbets flexapp. Allt UI och all text är på svenska.
Publiceras via GitHub Pages från branch `master`, rot. Ingen byggprocess, inga beroenden.

## Filer
- `index.html` – hela appen (HTML + CSS + JS i en fil, vanilla JS, ingen ramverk).
- `sw.js` – service worker (offline, stale-while-revalidate). **Höj `CACHE`-versionen
  (`flexlogg-vN`) vid varje ändring**, annars får telefonerna inte nya versionen.
- `manifest.webmanifest`, `icon-192.png`, `icon-512.png`, `apple-touch-icon.png`.

## Lagring
- Endast `localStorage`, nyckel `flexlogg-v1`: `{ S, E, T, tab }`. Ingen server, inget konto.
- `flexlogg-lastbackup` = tidsstämpel för senaste säkerhetskopia (påminnelse efter 14 dagar).
- Säkerhetskopia = JSON-fil `{ app, v, saved, S, E, T }` via share sheet/nedladdning; återställ via fil eller inklistrad text.
- Ändringar i datamodellen måste vara bakåtkompatibla: migrera i `normalizeSettings()`.

## Datamodell
- `S` (inställningar):
  - `categories: [{ id, name, type: 'work'|'sub'|'other', parent?, color, archived? }]`
    - `work` = huvudkategori, räknas som arbetstid.
    - `sub` = delkategori under en `work` (`parent`), delar upp arbetstid. Bara två nivåer.
    - `other` = ej arbetstid, loggas men räknas inte.
    - `note?: true` = kommentarsruta visas vid stopp/byte för kategorin; `notePrompt?` = stödtext
      (etikett i rutan, t.ex. "Elevens namn"). Förslagen (senast använda) hämtas per kategori.
      Posten får fältet `note` i `E`.
  - `schedules: [{ from: 'YYYY-MM-DD' | '', hours: [7 × { on, start, end }] }]` – **flexram**
    (tiden då stämplingar i jobbappen får ligga), index 0 = måndag. Sorterad på `from`,
    första har `from: ''` (= från början). Varje dag räknas med schemat som gällde då
    (`hoursFor(day)`), så historiken ändras inte när flexramen ändras.
  - `lunch` – minuter som jobbappen drar (60). Läggs på utstämplingen i förslaget, men bara på dagar
    med mer arbetstid än `lunchOver` (min, förval 240 = 4 h). Kortare dagar får ingen rast.
  - `offDays` – används inte längre (finns kvar i gamla data, ignoreras). Helgtid fördelas bakåt, se Förslag.
  - `v` – settings-version (2), används för engångsmigreringar.
- `E` (poster): `[{ id, cat, start, end|null, startedBy? }]`, ms-tidsstämplar. `end: null` = pågår.
  `startedBy` på en huvudkategoripost = id på delkategoriposten som startade den.
- `T` (överfört till jobbappen): `{ 'YYYY-MM-DD' | 'sep-<måndag>': { at, in, out, dur } }`.
  Om aktuellt förslag skiljer sig från sparat visas "Ändrad efter överföring".

## Beteende (beslutat med användaren)
- Start/stopp per knapp. Starta delkategori → startar huvudkategorin om den inte går.
  Stoppa delkategori → stoppar huvudkategorin bara om den startades av just den delkategorin.
  Stoppa huvudkategori → stoppar dess delkategorier. Byte av delkategori under samma
  huvudkategori stoppar den gamla.
- Stämpeltid: "Nu", −5/−15/−30 min eller valfri tid väljs *innan* man trycker; återgår till Nu.
- Arbetstid per dag = union av alla work/sub-intervall (ingen dubbelräkning), minus tid där en
  `other`-post (lunch, ärende …) ligger över. Avdraget gäller alla ej-arbetstid-kategorier och görs
  i både `daySummary()` och `computeStats()` (`subtractIv`). Posterna själva ändras aldrig.
- Förslag per dag (`proposal()`): total arbetstid inkl. kvälls-/helgtid placeras inom flexramen.
  In = första stämpling inom ramen, annars ramens start. Ut = In + arbetstid + lunch. Går den över
  ramens slut flyttas In tidigare.
- Veckofördelning (`weekData()`): tid på dagar utan flexram (helg) + överskott fördelas bakåt från
  fredag. Dagens max = flexramens längd − lunch. Det som inte ryms flyttas till dagen före, men bara
  till dagar med loggad tid (lediga dagar lämnas orörda). Går det över måndag blir det
  "Ryms inte i veckan" (röd varning, egen post med nyckel `sep-<måndag>` i `T`) som användaren själv
  lägger på annan vecka i jobbappen.
- Användaren loggar sin faktiska lunch i Flexlogg (stoppar Arbete) och stämplar inte lunch i jobbappen.
- Statistik: vecka/månad/halvår/år. Delkategorier i % av sin huvudkategori + rad "Övrigt".
  "Utanför flexram" visas som virtuell huvudkategori med fördelning per delkategori.
  Ej arbetstid bara i timmar. Stapeldiagram: per vecka (år/halvår), per dag (månad/vecka),
  läge "Inom/utanför" eller "Per kategori".

## Flikar
Stämpla · Logg (redigera/lägg till/ta bort poster per dag) · Flex (veckoförslag, kopiera,
markera överfört) · Statistik · Inställningar (kategorier, flexram-scheman, lunch, data).

## Arbetssätt
- **Committa och pusha till `master` direkt när en ändring är klar** (med höjd `CACHE`-version).
  Höj även `APP_VERSION` i `index.html` så den matchar (visas längst ned under Inställningar).
  Användaren kan bara testa via GitHub Pages, så ändringen måste vara pushad för att kunna provas.
- Användaren vill resonera färdigt innan något byggs – fråga/föreslå först vid nya features.
- Testa i mobilbredd (390 px). Syntaxkontroll: extrahera `<script>` och kör `node --check`.
