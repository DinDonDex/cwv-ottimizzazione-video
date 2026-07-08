# Linee guida: video in background ATF compliant con i Core Web Vitals

Obiettivo: caricare un video di sfondo nella hero section (above the fold) della homepage senza penalizzare LCP, CLS e INP.

---

## 1. Principio architetturale fondamentale

**Il video non deve mai essere l'elemento che determina l'LCP.** La strategia vincente è:

1. Il browser carica e mostra **subito un'immagine poster ottimizzata** → questa diventa l'elemento LCP (veloce, prevedibile).
2. Il video viene **iniettato/caricato in un secondo momento** (dopo il load o in idle) e fa un fade-in sopra il poster.
3. L'utente percepisce un caricamento istantaneo; il video è un "progressive enhancement".

> Nota: da Chrome 112+ il primo frame di un video autoplay è un candidato LCP. Se il video parte tardi o il primo frame arriva lento, l'LCP peggiora. Il poster preloadato risolve il problema alla radice perché viene dipinto prima.

---

## 2. LCP (target < 2,5s)

### Poster image
- Formato **AVIF o WebP**, due versioni dedicate:
  - **Desktop**: 1600–1920px di larghezza, aspect ratio 16:9
  - **Mobile**: ~800px di larghezza, aspect ratio 9:16 (stesso crop verticale del video mobile)
- Peso target: **< 100 KB** (idealmente 50–80 KB per un hero full-screen).
- Il poster deve essere **il primo frame del video** (o molto simile) per evitare uno "stacco" visivo al fade-in.
- Preload nel `<head>` con priorità alta:

```html
<link rel="preload" as="image"
      href="/hero-poster.avif"
      imagesrcset="/hero-poster-mobile.avif 800w, /hero-poster.avif 1920w"
      imagesizes="100vw"
      fetchpriority="high">
```

### Video
- `preload="none"` (o al massimo `"metadata"`): il browser non deve scaricare il video in competizione con le risorse critiche.
- **Mai** `fetchpriority="high"` sul video, mai preload del file video.
- **Mai** iframe YouTube/Vimeo come background: caricano centinaia di KB di JS che distruggono LCP, TBT e INP. Solo `<video>` nativo self-hosted o su CDN.

---

## 3. Risoluzioni e ottimizzazione dei file video

Per un video di background (con overlay di testo e spesso filtro scuro) la risoluzione può essere più bassa dell'istinto: nessuno guarda i dettagli e la compressione maschera molto.

| Contesto | Risoluzione | Aspect ratio | Peso target |
|---|---|---|---|
| Desktop | 1280×720 (max 1920×1080) | 16:9 | 1,5–2,5 MB |
| Mobile portrait | 720×1280 (o 608×1080) | 9:16 | 0,8–1,5 MB |
| Tablet | riusa il desktop 720p | 16:9 | — |

Regole:

- **Desktop**: parti dal 720p; con overlay scuro un 720p upscalato è indistinguibile dal 1080p nel 90% dei casi e pesa la metà. Sali a 1080p solo se il degrado è visibile su un monitor 27". **Mai oltre il 1080p**, mai inseguire i display retina/HiDPI: per video in movimento con overlay non porta beneficio percepibile.
- **Mobile**: **versione dedicata con crop verticale 9:16**, non lo stesso file ridotto. Un 16:9 croppato da `object-fit: cover` su schermo 9:16 mostra solo una striscia centrale, spesso tagliando il soggetto. Il crop va ricomposto in export sul soggetto.
- **Conta più il bitrate della risoluzione**: un 1080p a CRF 30 può pesare meno di un 720p a CRF 23. Per un background puoi spingerti a **CRF 28–32 (H.264)** e **38–42 (VP9)** senza artefatti visibili. Verifica sempre a occhio sul dispositivo reale.
- Durata loop: **10–15 secondi max**.
- **Rimuovere la traccia audio** (`-an`): è muto comunque, risparmia peso.
- Doppio formato: **WebM (VP9/AV1)** + fallback **MP4 (H.264)**.

Comandi ffmpeg:

```bash
# Desktop — MP4 H.264
ffmpeg -i input.mov -an -vf "scale=1280:-2" \
  -c:v libx264 -crf 28 -preset slow -movflags +faststart hero.mp4

# Desktop — WebM VP9
ffmpeg -i input.mov -an -vf "scale=1280:-2" \
  -c:v libvpx-vp9 -crf 38 -b:v 0 hero.webm

# Mobile — crop verticale 9:16 + MP4
ffmpeg -i input.mov -an -vf "crop=ih*9/16:ih,scale=720:1280" \
  -c:v libx264 -crf 30 -preset slow -movflags +faststart hero-mobile.mp4

# Mobile — WebM VP9
ffmpeg -i input.mov -an -vf "crop=ih*9/16:ih,scale=720:1280" \
  -c:v libvpx-vp9 -crf 40 -b:v 0 hero-mobile.webm
```

- `-movflags +faststart` è obbligatorio sull'MP4: sposta il moov atom all'inizio del file, permettendo lo streaming immediato.
- Servire da **CDN** con `Cache-Control: public, max-age=31536000, immutable` e supporto alle richieste `Range`.

---

## 4. CLS (target < 0,1): dove riservare lo spazio

**Lo spazio lo riserva il contenitore via CSS, e quel CSS deve essere critico (inline nel `<head>`).**

Il ragionamento:

- Poster e video sono in `position: absolute` dentro `.hero`: sono fuori dal flusso, quindi **non possono causare shift per definizione**, qualunque cosa succeda al loro caricamento.
- Lo spazio lo definisce `.hero { height: 100svh }` (o un `aspect-ratio`) — un valore che non è esprimibile con attributi DOM, quindi è per forza CSS.
- Il punto critico è il **quando**: se la regola sta in un foglio esterno che carica tardi, il browser fa un primo paint con la hero collassata e poi la espande → CLS enorme. Tutta la geometria ATF va **inline nel `<head>`** o nel primo CSS render-blocking, mai in un file differito.

```html
<head>
  <style>
    /* CSS critico inline: la hero ha già la sua altezza al primo paint */
    .hero { position: relative; height: 100svh; overflow: hidden; }
    .hero__media {
      position: absolute; inset: 0;
      width: 100%; height: 100%;
      object-fit: cover;
    }
    .hero__video { opacity: 0; transition: opacity .6s ease; }
    .hero__video.is-playing { opacity: 1; }
  </style>
</head>
```

- Poster e video occupano **lo stesso box in absolute** → il passaggio poster→video non può causare shift.
- Il testo/CTA della hero va posizionato sopra con z-index, mai in flusso dipendente dal media.
- **Gli attributi `width`/`height` sul poster vanno messi comunque**: nel pattern assoluto non riservano spazio (il CSS li sovrascrive), ma comunicano al browser l'aspect-ratio intrinseco a costo zero e coprono il caso in cui il markup venga riusato in un contesto in-flow.
- Font della hero: `font-display: swap` + preload del font per evitare shift da FOUT.

> Regola generale: `width`/`height` nel DOM servono per media **nel flusso normale** del documento. Nel pattern hero con overlay assoluto lo spazio è responsabilità del contenitore via CSS critico; gli attributi restano come buona pratica difensiva.

---

## 5. INP / TBT

- **Zero librerie player** (no video.js, plyr, ecc.) per un background: solo API native.
- Lo script di iniezione del video deve essere minimo (poche righe) e caricato con `defer` o inline.
- Usare `requestIdleCallback` o l'evento `load` per il caricamento del video, così il main thread resta libero durante l'interattività iniziale.

---

## 6. Pattern di implementazione consigliato

```html
<section class="hero">
  <img class="hero__media hero__poster"
       src="/hero-poster.avif"
       srcset="/hero-poster-mobile.avif 800w, /hero-poster.avif 1920w"
       sizes="100vw"
       width="1920" height="1080"
       alt="" fetchpriority="high" decoding="async">

  <video class="hero__media hero__video"
         muted playsinline loop
         preload="none"
         poster="/hero-poster.avif"
         aria-hidden="true" tabindex="-1">
    <!-- sorgenti iniettate via JS -->
  </video>

  <div class="hero__content">
    <h1>Titolo</h1>
  </div>
</section>
```

```js
window.addEventListener('load', () => {
  const start = window.requestIdleCallback || ((cb) => setTimeout(cb, 200));
  start(() => loadHeroVideo());
});

function loadHeroVideo() {
  // Rispetta le preferenze utente e le connessioni lente
  if (window.matchMedia('(prefers-reduced-motion: reduce)').matches) return;

  const conn = navigator.connection;
  if (conn && (conn.saveData || /(^|-)2g/.test(conn.effectiveType || ''))) return;

  const video = document.querySelector('.hero__video');
  if (!video) return;

  // Selezione della versione: via JS, più affidabile dell'attributo
  // media su <source> (supporto browser disomogeneo)
  const isPortraitMobile = window
    .matchMedia('(max-width: 768px) and (orientation: portrait)').matches;

  const sources = isPortraitMobile
    ? [{ src: '/hero-mobile.webm', type: 'video/webm' },
       { src: '/hero-mobile.mp4',  type: 'video/mp4'  }]
    : [{ src: '/hero.webm', type: 'video/webm' },
       { src: '/hero.mp4',  type: 'video/mp4'  }];

  sources.forEach(({ src, type }) => {
    const s = document.createElement('source');
    s.src = src;
    s.type = type;
    video.appendChild(s);
  });

  video.load();
  video.play()
    .then(() => video.classList.add('is-playing'))
    .catch(() => { /* autoplay bloccato: resta il poster, nessun problema */ });
}
```

Punti chiave del pattern:

- **Attributi obbligatori per l'autoplay**: `muted` + `playsinline` (iOS). Senza `muted` l'autoplay viene bloccato.
- La `.catch()` sul `play()` garantisce degradazione elegante: se l'autoplay fallisce resta il poster.
- Il fade-in avviene solo a riproduzione avviata, mai su `loadedmetadata`.
- Non serve gestire resize/rotazione dopo il caricamento: chi ruota il telefono a metà sessione vedrà il video croppato da `object-fit: cover`, compromesso accettabile.

### Variante ancora più prudente: niente video su mobile
Su mobile spesso la scelta migliore per CWV, batteria e dati è **servire solo il poster**:

```js
if (window.matchMedia('(max-width: 768px)').matches) return;
```

---

## 7. Accessibilità e rispetto dell'utente (impatta indirettamente i CWV)

- `prefers-reduced-motion: reduce` → non caricare il video (già nel codice sopra).
- `Save-Data` e connessioni 2G/slow-2G → non caricare il video.
- `aria-hidden="true"` sul video decorativo, `alt=""` sul poster decorativo.
- Se il video contiene movimento significativo, valutare un pulsante pausa (WCAG 2.2.2).

---

## 8. Checklist pre-rilascio

- [ ] Poster AVIF/WebP < 100 KB, due versioni (desktop 16:9, mobile 9:16), preload con `fetchpriority="high"`
- [ ] Video desktop 1280×720 (max 1080p) 16:9, < 2,5 MB
- [ ] Video mobile 720×1280 con crop verticale 9:16 dedicato, < 1,5 MB (oppure: solo poster su mobile)
- [ ] Video senza audio, loop ≤ 15s, WebM + MP4
- [ ] CRF verificato a occhio su dispositivo reale (H.264: 28–32, VP9: 38–42)
- [ ] MP4 con `faststart`, servito da CDN con cache lunga e supporto Range
- [ ] `preload="none"`, sorgenti iniettate dopo `load` / idle, selezione mobile/desktop via JS
- [ ] `muted playsinline loop` presenti, `.catch()` sul play
- [ ] Geometria della hero in **CSS critico inline nel `<head>`**, media in `position:absolute` (CLS = 0)
- [ ] `width`/`height` sul poster come pratica difensiva
- [ ] Nessuna libreria player, nessun iframe di terze parti
- [ ] `prefers-reduced-motion` e `saveData` rispettati

---

## 9. Verifica e monitoraggio

- **Lab**: Lighthouse (mobile, throttling attivo) e WebPageTest → verificare che l'elemento LCP sia il poster e che il video non compaia nel waterfall prima del load.
- **Field**: CrUX / PageSpeed Insights (dati reali a 28 giorni) e RUM con la libreria `web-vitals` per monitorare LCP, CLS e INP della homepage nel tempo.
- Test da ripetere su: 4G throttled, Safari iOS (autoplay), Chrome Android.
