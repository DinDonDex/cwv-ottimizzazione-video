# Linee guida WordPress: video in background ATF compliant con i Core Web Vitals

> ## ⚠️ AMBITO DI APPLICAZIONE
>
> **Questa soluzione vale ESCLUSIVAMENTE per video in BACKGROUND collocati ABOVE THE FOLD (hero della homepage).**
>
> Presupposti su cui si regge l'intero pattern: video **decorativo**, **muto**, **in loop**, **senza controlli**, con **testo/CTA in overlay**.
>
> **NON usare questa soluzione per:**
> - Video di **contenuto** (con audio, controlli, o che l'utente deve guardare attivamente) → serve un player con facade/click-to-play
> - Video **below the fold** → lì la strategia corretta è l'opposto: lazy-load con `IntersectionObserver`, nessun preload del poster, nessun `fetchpriority="high"`
> - Video in **pagine interne** dove il video non è l'elemento hero ATF
>
> Applicare queste tecniche fuori dal loro ambito (es. preload del poster su un video a metà pagina) **peggiora** i Core Web Vitals invece di migliorarli.

Obiettivo: implementare in WordPress un video di sfondo nella hero della homepage senza penalizzare LCP, CLS e INP. Copre due scenari: **con ACF** e **senza ACF**.

**Vincolo di progetto: tutti i media (video e poster, desktop e mobile) devono essere sostituibili dal backend senza toccare il codice.** Il codice del tema definisce solo *come* i media vengono caricati; *quali* media caricare è sempre contenuto editabile.

La strategia architetturale è la stessa della guida generale: **poster preloadato = elemento LCP, video nativo iniettato via JS dopo il load**. Qui vediamo come calarla nell'ecosistema WordPress, dove i problemi principali sono tre: i page builder, i plugin di ottimizzazione e il lazy-loading nativo di WP.

---

## 1. Cosa NON fare in WordPress

- **Niente "video background" dei page builder** (Elementor, Divi, WPBakery): quasi tutti caricano il video subito nell'HTML con `preload="auto"`, spesso senza poster ottimizzato, e alcuni usano YouTube come sorgente. Risultato: LCP distrutto. Se il tema è costruito con un builder, la hero video va comunque implementata a mano (template custom o blocco dedicato).
- **Niente embed YouTube/Vimeo** come background, nemmeno tramite plugin.
- **Niente plugin "video background"** generici dal repository: nessuno di quelli diffusi implementa il pattern poster-first + iniezione differita.
- **Niente URL hardcoded nel tema**: violerebbe il vincolo di editabilità. Ogni riferimento a un media passa da un campo (ACF o Customizer) che restituisce ID attachment o URL.
- **Il video NON va servito da WordPress/PHP**: deve essere un file statico. I file caricati nella Media Library finiscono in `/wp-content/uploads/` come statici, quindi vanno bene — verifica però `upload_max_filesize`/`post_max_size` (i video possono superare i limiti di hosting condivisi) e che il CDN davanti al sito supporti le richieste `Range`.

---

## 2. Preparazione dei file e Media Library

Specifiche identiche alla guida generale:

- Video desktop: 1280×720 16:9, < 2,5 MB — Video mobile: 720×1280 9:16 dedicato, < 1,5 MB
- WebM + MP4 (con `-movflags +faststart`), senza audio, loop ≤ 15s, CRF 28–32 (H.264) / 38–42 (VP9)
- Poster desktop 16:9 (1600–1920px) e mobile 9:16 (~800px), AVIF/WebP, < 100 KB, corrispondenti al primo frame

Poiché tutto passa dalla Media Library, assicurati che i formati siano caricabili:

```php
// functions.php — abilitare upload webm/avif se necessario
// (WP accetta .avif solo dalla 6.5; .webm è ammesso di default ma alcuni hosting lo bloccano)
add_filter( 'upload_mimes', function ( $mimes ) {
    $mimes['webm'] = 'video/webm';
    $mimes['avif'] = 'image/avif';
    return $mimes;
} );

// Se l'hosting ha limiti di upload bassi, alzarli da php.ini / .htaccess / pannello hosting:
// upload_max_filesize = 16M, post_max_size = 16M
```

**Consegna al cliente/redazione**: insieme al sito, fornisci le specifiche di export (risoluzioni, formati, pesi, comandi ffmpeg della guida generale). L'editabilità senza codice funziona solo se chi sostituisce i file rispetta le specifiche — un video da 40 MB caricato dal cliente vanifica tutto. Valuta di documentarle anche nelle istruzioni dei campi (ACF le supporta nativamente).

---

## 3. Scenario A — Con ACF

### 3.1 Field group

Crea un field group "Hero Video". Assegnalo a un **Options Page** (consigliato: sopravvive a cambi di template e di pagina front) oppure alla homepage (*Page = Front Page*):

| Field | Tipo | Return format | Istruzioni da inserire nel campo |
|---|---|---|---|
| `hero_video_mp4` | File | URL | "MP4 1280×720, max 2,5 MB, senza audio" |
| `hero_video_webm` | File | URL | "WebM 1280×720, max 2,5 MB" |
| `hero_video_mp4_mobile` | File | URL | "MP4 720×1280 verticale, max 1,5 MB" |
| `hero_video_webm_mobile` | File | URL | "WebM 720×1280 verticale, max 1,5 MB" |
| `hero_poster_desktop` | Image | ID | "AVIF/WebP 1920×1080, max 100 KB, primo frame del video" |
| `hero_poster_mobile` | Image | ID | "AVIF/WebP 800×1422, max 100 KB" |
| `hero_disable_video_mobile` | True/False | — | "Se attivo, su mobile viene mostrato solo il poster" |

Return format **ID** per le immagini (serve per `wp_get_attachment_image` e per il preload), **URL** per i video (vanno solo nelle sorgenti JS). Con l'Options Page, nei `get_field()` va aggiunto il secondo parametro `'option'`.

### 3.2 Template (front-page.php o template part)

```php
<?php
$poster_desktop_id = get_field( 'hero_poster_desktop', 'option' );
$poster_mobile_id  = get_field( 'hero_poster_mobile', 'option' );

$poster_desktop_url = wp_get_attachment_image_url( $poster_desktop_id, 'full' );
$poster_mobile_url  = wp_get_attachment_image_url( $poster_mobile_id, 'full' );

// Degradazione: senza poster, niente hero media (mai markup rotto)
if ( ! $poster_desktop_url ) {
    return;
}
?>
<section class="hero">
  <picture>
    <?php if ( $poster_mobile_url ) : ?>
      <source media="(max-width: 768px) and (orientation: portrait)"
              srcset="<?php echo esc_url( $poster_mobile_url ); ?>">
    <?php endif; ?>
    <?php
    echo wp_get_attachment_image( $poster_desktop_id, 'full', false, [
        'class'         => 'hero__media hero__poster',
        'alt'           => '',
        'loading'       => 'eager',        // disattiva il lazy-load nativo di WP
        'fetchpriority' => 'high',
        'decoding'      => 'async',
    ] );
    ?>
  </picture>

  <video class="hero__media hero__video"
         muted playsinline loop preload="none"
         poster="<?php echo esc_url( $poster_desktop_url ); ?>"
         aria-hidden="true" tabindex="-1"
         data-mp4="<?php echo esc_url( get_field( 'hero_video_mp4', 'option' ) ); ?>"
         data-webm="<?php echo esc_url( get_field( 'hero_video_webm', 'option' ) ); ?>"
         data-mp4-mobile="<?php echo esc_url( get_field( 'hero_video_mp4_mobile', 'option' ) ); ?>"
         data-webm-mobile="<?php echo esc_url( get_field( 'hero_video_webm_mobile', 'option' ) ); ?>"
         data-disable-mobile="<?php echo get_field( 'hero_disable_video_mobile', 'option' ) ? '1' : '0'; ?>">
  </video>

  <div class="hero__content">
    <h1><?php the_title(); ?></h1>
  </div>
</section>
```

Le URL dei video passano nei `data-*` attribute: il JS resta generico e riutilizzabile, i contenuti restano editabili da ACF. Il JS ignora i campi vuoti, quindi il cliente può caricare anche solo l'MP4 e il sito continua a funzionare.

### 3.3 Preload del poster nel `<head>`

```php
// functions.php
add_action( 'wp_head', function () {
    if ( ! is_front_page() ) {
        return;
    }
    $desktop = wp_get_attachment_image_url( get_field( 'hero_poster_desktop', 'option' ), 'full' );
    $mobile  = wp_get_attachment_image_url( get_field( 'hero_poster_mobile', 'option' ), 'full' );

    if ( ! $desktop ) {
        return;
    }
    printf(
        '<link rel="preload" as="image" href="%s"%s fetchpriority="high">' . "\n",
        esc_url( $desktop ),
        $mobile ? sprintf( ' imagesrcset="%s 800w, %s 1920w" imagesizes="100vw"', esc_url( $mobile ), esc_url( $desktop ) ) : ''
    );
}, 1 ); // priorità 1: prima possibile nel head
```

Il preload legge dagli stessi campi del template: quando il cliente sostituisce il poster in ACF, il preload si aggiorna da solo. **Nessuna URL vive nel codice.**

---

## 4. Scenario B — Senza ACF: Customizer

Senza ACF, l'equivalente nativo che rispetta il vincolo di editabilità è il **Customizer** (theme mods): il cliente sostituisce video e poster da *Aspetto → Personalizza → Hero Video*, con anteprima live, zero plugin.

```php
// functions.php — registrazione
add_action( 'customize_register', function ( $wp_customize ) {
    $wp_customize->add_section( 'hero_video', [
        'title'       => 'Hero Video',
        'description' => 'Video: MP4/WebM 1280×720 (max 2,5 MB) e 720×1280 mobile (max 1,5 MB), senza audio. Poster: AVIF/WebP max 100 KB, primo frame del video.',
    ] );

    // Video → WP_Customize_Media_Control con mime_type video (salva l'ID attachment)
    $videos = [
        'hero_video_mp4'      => 'Video MP4 desktop',
        'hero_video_webm'     => 'Video WebM desktop',
        'hero_video_mp4_mob'  => 'Video MP4 mobile (9:16)',
        'hero_video_webm_mob' => 'Video WebM mobile (9:16)',
    ];
    foreach ( $videos as $key => $label ) {
        $wp_customize->add_setting( $key, [ 'sanitize_callback' => 'absint' ] );
        $wp_customize->add_control( new WP_Customize_Media_Control(
            $wp_customize, $key,
            [ 'label' => $label, 'section' => 'hero_video', 'mime_type' => 'video' ]
        ) );
    }

    // Poster
    foreach ( [ 'hero_poster_desktop' => 'Poster desktop (16:9)', 'hero_poster_mobile' => 'Poster mobile (9:16)' ] as $key => $label ) {
        $wp_customize->add_setting( $key, [ 'sanitize_callback' => 'absint' ] );
        $wp_customize->add_control( new WP_Customize_Media_Control(
            $wp_customize, $key,
            [ 'label' => $label, 'section' => 'hero_video', 'mime_type' => 'image' ]
        ) );
    }

    // Toggle "solo poster su mobile"
    $wp_customize->add_setting( 'hero_disable_video_mobile', [ 'sanitize_callback' => 'wp_validate_boolean' ] );
    $wp_customize->add_control( 'hero_disable_video_mobile', [
        'label'   => 'Disattiva il video su mobile (solo poster)',
        'section' => 'hero_video',
        'type'    => 'checkbox',
    ] );
} );
```

`WP_Customize_Media_Control` salva **ID attachment** per tutto (video compresi), quindi nel template si convertono in URL:

```php
<?php
// Helper: uniforma l'accesso ai media della hero
function mytheme_hero_media( string $key ): string {
    $id = get_theme_mod( $key );
    return $id ? (string) wp_get_attachment_url( $id ) : '';
}
?>
<video class="hero__media hero__video"
       muted playsinline loop preload="none"
       poster="<?php echo esc_url( mytheme_hero_media( 'hero_poster_desktop' ) ); ?>"
       aria-hidden="true" tabindex="-1"
       data-mp4="<?php echo esc_url( mytheme_hero_media( 'hero_video_mp4' ) ); ?>"
       data-webm="<?php echo esc_url( mytheme_hero_media( 'hero_video_webm' ) ); ?>"
       data-mp4-mobile="<?php echo esc_url( mytheme_hero_media( 'hero_video_mp4_mob' ) ); ?>"
       data-webm-mobile="<?php echo esc_url( mytheme_hero_media( 'hero_video_webm_mob' ) ); ?>"
       data-disable-mobile="<?php echo get_theme_mod( 'hero_disable_video_mobile' ) ? '1' : '0'; ?>">
</video>
```

Il resto del markup (poster con `<picture>`, `loading="eager"`, `fetchpriority="high"`) e il preload in `wp_head` sono identici allo scenario ACF: si sostituisce `get_field( 'x', 'option' )` con `get_theme_mod( 'x' )`.

> Alternativa allo stesso livello: una pagina opzioni custom con Settings API. Funzionalmente equivalente, ma richiede più codice del Customizer senza vantaggi per questo caso d'uso. Da preferire solo se il progetto ha già una pagina impostazioni del tema.

---

## 5. Parti comuni a entrambi gli scenari

### 5.1 CSS critico inline

La geometria della hero deve arrivare **inline nel `<head>`**, non nel foglio di stile del tema (che i plugin di ottimizzazione spesso differiscono):

```php
// functions.php
add_action( 'wp_head', function () {
    if ( ! is_front_page() ) {
        return;
    }
    ?>
    <style id="hero-critical-css">
      .hero{position:relative;height:100svh;overflow:hidden}
      .hero__media{position:absolute;inset:0;width:100%;height:100%;object-fit:cover}
      .hero__video{opacity:0;transition:opacity .6s ease}
      .hero__video.is-playing{opacity:1}
      .hero__content{position:relative;z-index:1}
    </style>
    <?php
}, 2 );
```

### 5.2 Script di iniezione

```php
// functions.php
add_action( 'wp_enqueue_scripts', function () {
    if ( ! is_front_page() ) {
        return;
    }
    wp_enqueue_script(
        'hero-video',
        get_theme_file_uri( 'assets/js/hero-video.js' ),
        [],
        filemtime( get_theme_file_path( 'assets/js/hero-video.js' ) ),
        [ 'strategy' => 'defer', 'in_footer' => true ] // WP 6.3+
    );
} );
```

```js
// assets/js/hero-video.js
window.addEventListener('load', () => {
  const start = window.requestIdleCallback || ((cb) => setTimeout(cb, 200));
  start(loadHeroVideo);
});

function loadHeroVideo() {
  const video = document.querySelector('.hero__video');
  if (!video) return;

  if (window.matchMedia('(prefers-reduced-motion: reduce)').matches) return;

  const conn = navigator.connection;
  if (conn && (conn.saveData || /(^|-)2g/.test(conn.effectiveType || ''))) return;

  const isMobile = window
    .matchMedia('(max-width: 768px) and (orientation: portrait)').matches;

  if (isMobile && video.dataset.disableMobile === '1') return;

  const webm = isMobile && video.dataset.webmMobile ? video.dataset.webmMobile : video.dataset.webm;
  const mp4  = isMobile && video.dataset.mp4Mobile  ? video.dataset.mp4Mobile  : video.dataset.mp4;

  [{ src: webm, type: 'video/webm' }, { src: mp4, type: 'video/mp4' }]
    .filter(({ src }) => src)
    .forEach(({ src, type }) => {
      const s = document.createElement('source');
      s.src = src;
      s.type = type;
      video.appendChild(s);
    });

  if (!video.querySelector('source')) return;

  video.load();
  video.play()
    .then(() => video.classList.add('is-playing'))
    .catch(() => {}); // autoplay bloccato: resta il poster
}
```

Il JS legge tutto dai `data-*` attribute e ignora i campi vuoti: quando il cliente sostituisce o rimuove un file dal backend, non serve toccare nulla.

### 5.3 Neutralizzare le interferenze di WordPress e dei plugin

Questo è il punto in cui le implementazioni WordPress falliscono più spesso. Verifica ogni voce:

**Lazy-loading nativo di WP**
- WP aggiunge `loading="lazy"` alle immagini: sul poster va forzato `loading="eager"` (già fatto nei template). Se il poster viene stampato da altre funzioni, usa il filtro:

```php
add_filter( 'wp_img_tag_add_loading_attr', function ( $value, $image ) {
    return str_contains( $image, 'hero__poster' ) ? 'eager' : $value;
}, 10, 2 );
```

- WP 6.3+ assegna `fetchpriority="high"` automaticamente alla prima immagine, ma solo dentro il contenuto/loop: sul markup custom del tema va impostato esplicitamente.

**Plugin di ottimizzazione (WP Rocket, LiteSpeed Cache, Autoptimize, Perfmatters, ecc.)**
- **Escludere il poster dal lazy-load del plugin**: aggiungi `hero__poster` alle classi/keyword escluse (WP Rocket: *Media → Lazyload → Excluded images*; LiteSpeed: *Page Optimization → Media Excludes*). Usa la **classe**, non l'URL del file: le esclusioni per URL si rompono alla prima sostituzione del media dal backend.
- **Escludere `hero-video.js` da delay/defer aggressivi**: la funzione "Delay JavaScript execution" di WP Rocket rimanda lo script alla prima interazione utente — per il background va bene solo se accetti che il video parta dopo il primo scroll/tap; altrimenti escludilo.
- **Escludere il CSS critico inline dalla minificazione/combinazione**: lo `<style id="hero-critical-css">` non deve essere spostato in un file esterno differito. La maggior parte dei plugin non tocca gli style inline, ma verificalo con "Remove unused CSS" attivo.
- **Non far lazy-loadare il tag `<video>`**: con il nostro pattern il video non ha `src` nell'HTML, quindi in genere non c'è nulla da fare, ma verifica che il plugin non aggiunga attributi che interferiscono con `video.load()`.

**CDN**
- Se usi un plugin CDN/rewrite (Cloudflare, BunnyCDN, ecc.), verifica che riscriva anche le URL nei `data-*` attribute e nel `<link rel="preload">`. In caso contrario, filtra le URL verso il CDN direttamente in PHP (`wp_get_attachment_url` è filtrabile), così la riscrittura segue automaticamente ogni sostituzione del media.

**Cache di pagina**
- Il markup della hero è statico e cache-friendly: nessuna logica per-utente nel template. La selezione mobile/desktop avviene nel JS client-side, quindi **non serve** (e non va usata) una cache separata per dispositivo.
- **Dopo ogni sostituzione dei media dal backend va svuotata la page cache**, altrimenti il preload e i `data-*` in cache puntano ai vecchi file. Con ACF su Options Page o Customizer valuta un hook di purge automatico (es. `customize_save_after` / `acf/save_post` → flush della cache della front page), così il cliente non deve ricordarsene.

---

## 6. Checklist WordPress

- [ ] Video e poster preparati secondo la guida generale (formati, risoluzioni, pesi, CRF)
- [ ] Specifiche di export documentate nelle istruzioni dei campi (ACF) o nella descrizione della sezione (Customizer)
- [ ] **Zero URL di media nel codice**: tutto arriva da campi ACF o theme mods
- [ ] Upload webm/avif abilitati, limiti upload dell'hosting adeguati
- [ ] Nessun page builder / plugin video background / embed YouTube per la hero
- [ ] Poster con `loading="eager"` + `fetchpriority="high"` (lazy-load nativo WP disattivato sul poster)
- [ ] `<link rel="preload">` del poster in `wp_head`, generato dagli stessi campi del template
- [ ] CSS critico della hero inline in `wp_head`, escluso da ottimizzazioni dei plugin
- [ ] `hero-video.js` enqueued con `defer`, legge solo `data-*`, tollera campi vuoti
- [ ] Poster escluso dal lazy-load del plugin **per classe** (mai per URL)
- [ ] `hero-video.js` escluso (o consapevolmente incluso) dal "Delay JS" del plugin
- [ ] Purge automatico della page cache alla modifica dei campi hero
- [ ] `prefers-reduced-motion`, `saveData` e toggle "no video su mobile" gestiti
- [ ] Test con cache e ottimizzazioni ATTIVE: Lighthouse mobile, PSI, Safari iOS
- [ ] Test di sostituzione: cambia video e poster dal backend e verifica che preload, poster e sorgenti si aggiornino senza toccare codice

## 7. Verifica

- Testa sempre la pagina **con tutti i plugin di cache/ottimizzazione attivi e cache calda**: è la configurazione che vedono gli utenti reali e Google.
- In Lighthouse/PSI verifica che: l'elemento LCP sia il poster, il video non compaia nel waterfall prima dell'evento load, CLS = 0 sulla hero.
- Esegui il **test di sostituzione end-to-end**: un redattore cambia i media dal backend, si svuota la cache, la pagina risulta corretta e i CWV restano verdi. È il collaudo del vincolo di progetto.
- Monitora nel tempo con CrUX/PSI (dati field a 28 giorni); dopo ogni aggiornamento dei plugin di ottimizzazione, ricontrolla le esclusioni (gli update a volte le resettano).
