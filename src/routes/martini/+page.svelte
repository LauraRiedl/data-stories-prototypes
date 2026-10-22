<!-- ========================================
  Master's thesis: User Engagement with Hybrid Data Stories
  A Comparative Study of Generation Y and Generation Z

  Martini Glass prototype, route /martini
  Data story used in the between-subjects study reported in Chapter 5

======================================== -->
<script>
  import { onMount, tick } from 'svelte';
  import { fly } from 'svelte/transition';
  import { cubicOut } from 'svelte/easing';
  import * as d3 from 'd3';

  let alleSongs      = $state.raw([]);
  let geladen        = $state(false);
  // Story phases: 0 = start, 1 = the extremes, 2 = genres, 3 = mood atlas, 4 = explore, 5 = closing.
  let phase          = $state(0);
  let tipPhase       = $state(0);
  let tipTop         = $state(0);
  let phaseTop       = $state(0);

  onMount(() => {
    if ('scrollRestoration' in history) history.scrollRestoration = 'manual';
    window.scrollTo(0, 0);
    requestAnimationFrame(() => window.scrollTo(0, 0));
    ladeSpotifyIframeApi();
  });

  let genre          = $state('alle');
  let stimmungFilter = $state('alle');
  let suchbegriff    = $state('');
  let ausgewaehlt    = $state(null);
  let modalGenre     = $state(null);
  let sichtbareEmbeds = $state(new Set());
  const spotifyControllers = new Map();
  let spotifyApiPromise = null;
  // The Spotify iframe API script loads only once per browser session and fires its ready signal
  // only once. After a client-side route change a new page would otherwise wait forever for a
  // signal that never comes again, so the API reference is cached in a module-level promise.
  function ladeSpotifyIframeApi() {
    if (spotifyApiPromise) return spotifyApiPromise;
    spotifyApiPromise = new Promise((resolve) => {
      if (typeof window === 'undefined') { resolve(null); return; }
      if (window.__spotifyIframeApi) { resolve(window.__spotifyIframeApi); return; }
      const vorherigeReady = window.onSpotifyIframeApiReady;
      window.onSpotifyIframeApiReady = (api) => {
        window.__spotifyIframeApi = api;
        if (typeof vorherigeReady === 'function') vorherigeReady(api);
        resolve(api);
      };
      if (!document.querySelector('script[data-spotify-iframe-api]')) {
        const script = document.createElement('script');
        script.src = 'https://open.spotify.com/embed/iframe-api/v1';
        script.async = true;
        script.dataset.spotifyIframeApi = 'true';
        document.body.appendChild(script);
      }
      let versuche = 0;
      const poll = setInterval(() => {
        versuche += 1;
        if (window.__spotifyIframeApi) { clearInterval(poll); resolve(window.__spotifyIframeApi); }
        else if (versuche > 40) { clearInterval(poll); }
      }, 150);
    });
    return spotifyApiPromise;
  }
  async function spotifyAbspielen(slotKey, trackId) {
    const bestehender = spotifyControllers.get(slotKey);
    if (bestehender) { bestehender.togglePlay(); return; }
    if (!sichtbareEmbeds.has(slotKey)) sichtbareEmbeds = new Set([...sichtbareEmbeds, slotKey]);
    await tick();
    const api = await ladeSpotifyIframeApi();
    if (!api) return;
    const el = document.getElementById('embed-' + slotKey);
    if (!el) return;
    api.createController(el, { uri: 'spotify:track:' + trackId, width: '100%', height: '80' }, (controller) => {
      spotifyControllers.set(slotKey, controller);
      controller.addListener('ready', () => controller.play());
    });
  }
  let radarZeigeSong  = $state(true);
  let radarZeigeGenre = $state(true);
  let radarZeigeAvg   = $state(true);
  let zeigeTop5 = $state(false);
  let zoomKarte = $state(null);
  let hilfeOffen = $state(null);
  let extremAusgewaehlt = $state(null);
  let showMethodik   = $state(false);
  let beobachter;
  let tipBeobachter;
  let zuflieger;
  let extremeGezeichnet = $state(false);
  let genreGezeichnet   = $state(false);
  let moodGezeichnet    = $state(false);
  let tourSchritt    = $state(0);
  let tourGesehen    = $state(false);
  let introSichtbar  = $state(false);
  let extremeSichtbar = $state(true);
  let scrollGesperrt = $state(true);

  const genres      = ['pop', 'rap', 'rock', 'latin', 'r&b', 'edm'];
  // Genre colors follow the qualitative ColorBrewer "Set2" scheme (Harrower & Brewer, 2003):
  // hue varies while lightness and saturation stay near-constant, so that no ranking is implied.
  const genreFarben = { pop: '#66c2a5', rap: '#fc8d62', rock: '#8da0cb', latin: '#e78ac3', 'r&b': '#a6d854', edm: '#ffd92f' };
  const SPOTIFY = '#1DB954';

  const t = {
      klickHintExtreme: 'Tap a highlighted circle for details.',
      hilfeChip: 'How to read',
      playHintIntro: 'Whenever you see a play button in this story, click it to listen to the song.',
      lblWusstest: '💡 Good to know', lblTipp: '💡 Good to know', lblAha: '💡 Good to know',
      lblFunFact: '💿 Fun fact',
      genreLabels: { pop: 'Pop', rap: 'Rap / Hip-Hop', rock: 'Rock', latin: 'Latin', 'r&b': 'R&B', edm: 'EDM' },
      navLabels: ['The Extremes', 'Genres', 'Fingerprints', 'Explore'],
      persistentNotice: 'Important: please remember to go back to the survey once you are done reading.',
      npTitel: 'The Mood Atlas of Music',
      methodikBtn: 'Methodology',
      methodikTitle: 'Methodology & data processing',
      methodikIntro: 'To keep the findings of this data story reproducible, this section transparently documents where the data comes from and how it was prepared, cleaned and analysed.',
      methodik: [
        { h: '1 · Data source', p: 'The basis is the openly available „30,000 Spotify Songs“ dataset (Kaggle, 2023), collected via the Spotify Web API. Only the pre-computed Spotify metrics it contains are used – no independent audio analysis.' },
        { h: '2 · Variables used', p: 'Valence (0–1, mood), energy (0–1), danceability (0–1) and popularity (0–100). For charts, popularity is normalised to 0–1 (divided by 100) to make it comparable with the other metrics.' },
        { h: '3 · Loading the data', p: 'The CSV is read client-side with D3 (d3.csv) and typed; all analysis runs in the browser, without a server or database.' },
        { h: '4 · Handling duplicates', p: 'The dataset is playlist-based: each of the 32,833 raw rows is one song within one playlist, so the same song appears once for every playlist that features it – occasionally even under different genres. For the analysis, duplicates are removed per title and artist, keeping the highest-popularity entry: 26,230 unique songs.' },
        { h: '5 · Cleaning', p: 'Only rows without a title or with invalid popularity are dropped. Songs with popularity 0 (about {pct}%) are kept – their mean valence ({valNull}) practically equals the overall mean ({valGesamt}).' },
        { h: '6 · Aggregation & selection', p: 'Genre values are means (d3.mean). Happiest examples: top-3 by valence. Melancholic examples: to avoid near-silent ambient tracks, the bottom decile by valence is taken, then the three most-popular within it. The scatterplot plots every song; points are binned by valence × energy (step 0.02) and rendered larger/darker with higher local density, a standard remedy for overplotting. Happy/melancholic in the Mood Atlas use the dataset\'s own valence quartiles (≥{happyQ} / ≤{melaQ}). The single track with valence exactly 0 – a four-second sound snippet – is excluded as an artefact. The three example songs of the fingerprint chapter follow the same logic as the melancholic examples: for energy (happy camp) and danceability (melancholic camp), the top decile of that dimension is taken and the most popular song within it is chosen; the third example is the most popular song of the two camps combined (ties broken alphabetically). The “top five” reveal likewise breaks the tie on rank five alphabetically, while its neutrality claim is verified against all tied candidates.' },
        { h: '7 · Visualisation', p: 'All graphics are SVG via D3.js v7 (app: Svelte 5): scatterplot of valence × energy (extremes highlighted), a binned heat-strip chart (genres: each genre\'s valence distribution in 0.05 bins; cell shading encodes the song count per bin on a square-root scale, normalised per genre; the dark line marks the genre mean), radar small multiples (three sonic fingerprints, each shown against the dashed overall average). In the Explore view, clicking a song opens a pop-up with its values, its genre\'s average profile (dashed, in a darkened genre colour) and the overall average (black, dotted).' },
        { h: '8 · Colour scheme', p: 'Genre colours: ColorBrewer „Set2” (Harrower & Brewer, 2003), a qualitative palette where only hue varies – no implied ranking. Mood: a diverging scheme around valence 0.5, with brighter green for the positive pole and darker violet for the negative, inspired by Valdez & Mehrabian (1994).' },
        { h: '9 · Limitations', p: 'The audio metrics are proprietary, algorithmically estimated values, not objective measurements; the genre assignment derives from playlist categories (not clear-cut). The tracks date up to 2020; the dataset is a snapshot. Results are exploratory, not causal.' }
      ],
      startKicker: '♪ ♫ ♬ ♩ ♪',
      startTitle: 'The Mood Atlas<br>of Music',
      startSub: 'What 26,230 songs reveal about feeling,<br>once you stop trusting the average',
      startCtaText: 'Ready to dive into the data?',
      startCtaBtn: 'Click here to start',
      introText1a: 'The most successful song in this entire collection is',
      introText1: '– a perfect popularity score of 100. But ask the data how this global hit actually feels, and the answer is strangely hollow: its mood value sits at 0.51, the dead centre of a scale that runs from despair to euphoria. The biggest song in the world is, emotionally speaking, a shrug.',
      introText2a: 'And it is not alone. The five most popular songs here are all like this: not happy, not sad, immaculately neutral. Yet this collection holds 26,230 songs, and among them are tracks of almost unbearable euphoria and others of bottomless melancholy.',
      introText2b: 'So where do all the feelings go? This story follows them: from the outermost edges of the mood scale, through a trap that statistics set for anyone who calculates an average, down to three individual records that show what a single song can do that a million-song mean never will.',
      introText2c: 'By the end, you may never trust an average again. 😉',
      storyExtreme1: 'absolutely belongs at your next summer party – while',
      storyExtreme2: 'is better left off, so the mood doesn\'t drop.',
      top5Auf: 'Click here to reveal the top 5 →',
      top5Zu: 'Click to reset',
      abschluss: 'On average, readers rush straight to Spotify at this point. Please be an outlier: return to the questionnaire first. 🎧',
      storyNeutral1: 'One more twist: the five most popular songs of the entire catalogue belong to neither camp. All five sit in the neutral middle of the mood scale.',
      storyNeutral2: 'Whatever it is that sells a song, it isn\'t the feeling: mood and popularity are practically strangers in this dataset.',
      laden: '♩ Loading data…',
      startLead: 'A data-journalistic investigation into the emotions hidden in music, and why the biggest number in the room is the one you should believe the least.',
      scrollCue: 'Scroll down to begin',
      s1_h2: 'The emotional extremes',
      s1_l1: 'Let\'s start where the feelings actually live: at the edges. Every dot below is one song, placed by its mood (the data calls it valence: 0 means sad, 1 means euphoric) and by its energy, how fiercely it sounds. The edges are anything but empty. ',
      s1TrigSad: 'Down on the left sit songs that barely cling to the scale',
      s1_l2: ', ',
      s1TrigHappy: 'far on the right tracks that practically overdose on joy',
      s1_l3: '.',
      s1_schluss: 'Music, the raw data says, feels everything.',
      s1_happy: 'Happy songs',
      s1_sad: 'Melancholic songs',
      s1_hinweis: 'This catalogue reaches back further than you might guess: its oldest tracks date back to 1957, among them Little Richard\'s Long Tall Sally. The newest entries are from 2020.',
      s2_h2: 'Genres & their mood',
      s2_lead: 'Does a whole genre have a mood? Here is where the data plays its cruellest trick. Average thousands of tracks per genre and the answer looks like a resounding "barely": all six genres huddle between 0.40 and 0.60: close enough that a single thumb on the chart could cover them all. On paper, all six feel like variations of neutral. But the averages are not the story. The colour below is. 🎨',
      s2_how: 'Each row is one genre. The deeper the colour, the more songs sit in that mood range. The dark line marks the genre\'s average. And one more thing: the highlighted phrases in the paragraph below the chart are clickable and make the chart act them out.',
      s2_d1: 'Now ',
      s2_trigAvg: 'ignore the dark average lines',
      s2_d2a: ' for a moment and simply follow the colour.',
      s2_d2b: 'Latin\'s colour gathers far to the right: ',
      s2_trigLatin: 'more than a third of its songs sit in the happy zone between 0.70 and 0.95',
      s2_d3a: ', far happier than its modest average of 0.60 lets on.',
      s2_d3b: 'EDM is the exact opposite: ',
      s2_trigEdm: '41 percent of its songs sit in the sad zone between 0.15 and 0.40',
      s2_d4: '. And one more surprise: every genre spreads out about equally wide. The six genres don\'t differ in how varied their songs are. They differ in where most of their songs live. That is the one thing an average can never show you.',
      s2_hinweis: 'In this catalogue, older songs sound happier: tracks from the 1960s to the 1980s average around 0.60 on the mood scale, the 2010s only 0.47. Music, it seems, has been growing sadder.',
      neutral: 'neutral (0.5)',
      genreXAxis: '😔 melancholic ← → happy 😄',
      genreYAxis: 'Genre',
      s3_h2: 'Sonic fingerprints',
      s3_g1: 'If averages erase feelings, there is only one honest way forward: stop averaging. Meet three records from this dataset that could hardly be more different, and watch how differently ',
      s3_g2: ' combine once a song is allowed to be itself instead of a data point in someone\'s mean.',
      s3_radarHilfe: 'Each radar shows one real song across four values (mood, energy, danceability, popularity): the further a point reaches outward, the higher the value. The black dashed outline is the average across all songs in the dataset: it shows at a glance where each song breaks away from the crowd. Danceability, by the way, describes how suitable a track is for dancing, based on tempo, rhythm stability, beat strength and overall regularity.',
      moodFroh: 'Happy songs',
      moodMela: 'Melancholic songs',
      moodAvg: 'Average of all songs',
      mb1: '⚡ The energy bomb',
      mb1r: 'The most popular of the most energetic happy songs.',
      mb2: '🕺 The melancholic mover',
      mb2r: 'The most popular of the most danceable melancholic songs.',
      mb3: '⭐ The crowd favourite',
      mb3r: 'The biggest hit among the happy and melancholic songs.',
      s3_takeaway: 'The catalogue\'s longest track stretches to 8.6 minutes. Its shortest song is over after less than 30 seconds.',
      scatterXTitle: 'Mood',
      scatterXAxis: '😔 melancholic ← → happy 😄',
      scatterYTitle: 'Energy',
      scatterYAxis: '😌 calm ← → energetic ⚡',
      colSong: 'Song', colArtist: 'Artist', valenceColLabel: 'Valence',
      s4_h2: 'Explore the Mood Atlas',
      s4_lead: 'You have seen what the average conceals. Now catch it in the act yourself. This is the full atlas, all 26,230 songs, yours to interrogate: search for a favourite track, filter by genre and mood, and click a bar to reveal a song\'s own fingerprint. Is your feel-good anthem really as happy as you remember? Does your favourite band live where its genre\'s average claims? Don\'t take the middle\'s word for it. Check the edges.',
      searchPh: '🔍 Search song or artist…',
      ergebnis: 'result', ergebnisse: 'results',
      genreFilter: 'Genre:', stimmungLabel: 'Mood:',
      filterAlle: 'All', filterHappy: '😄 Happy', filterSad: '😔 Melancholic',
      genreAlle: 'All genres',
      detailArtist: 'Artist:', detailAlbum: 'Album:',
      lblStimmung: 'Mood', lblEnergie: 'Energy', lblTanz: 'Danceability', lblPop: 'Popularity',
      playHint: 'Click on the play button to listen to the song.',
      keineSongs: 'No songs in this category.',
      exploreTop50: '💡 Filtered view: up to 50 songs in full ranking, the most melancholic or happiest first. Scrollable list.',
      s4_tipp: 'Click a bar to see that song\'s own profile, compared with its genre average and the overall average. Trust the song, not the mean.',
      exploreCtaText: 'Curious to find out how your favourite song ranks?',
      exploreCtaBtn: 'Click here to find out',
      tourTitel: 'Now it\'s your turn!',
      tourIntro: 'Which songs are you curious about? Here you can explore your favourite songs and genres yourself. A quick three-step guide:',
      tourLos: 'Let\'s go →',
      tour2: 'Here you can switch between genres.',
      tour3: 'Here you can switch between moods.',
      tour4: 'And here you can search for individual songs.',
      tourOk: 'Okay →',
      tourFertig: 'Got it!',
      tourSearchLabel: 'Search',
      defStimmung: 'A measure of the musical positiveness conveyed by a track: high values sound more positive (happy, cheerful, euphoric), low values more negative (sad, depressed, angry).',
      defEnergie: 'A perceptual measure of intensity and activity. Energetic tracks feel fast, loud, and noisy: death metal scores high, a Bach prelude low. A value of 0.78 means 78 percent of the maximum.',
      defTanz: 'How suitable a track is for dancing, based on tempo, rhythm stability, beat strength, and overall regularity. A value of 0.78 means 78 percent of the maximum.',
      defPop: 'Spotify\'s popularity score, based on the number and recency of streams. 100 = currently most played.',
      artSkala1: 'noun · Spotify metric · scale 0–1',
      artSkala100: 'noun · Spotify metric · scale 0–100',
      radarStimmung: 'Mood', radarEnergie: 'Energy', radarTanz: 'Dance', radarPop: 'Popularity',
      modalSchliessen: 'Close',
      modalGenreRadar: 'Sonic fingerprint:',
      modalRadarHilfe: 'How to read the chart: each of the four corners represents one value (mood, energy, danceability, popularity): the further a point reaches outward, the higher the value. Three shapes are layered on top of each other: the bold, filled outline is this song; the dashed outline in the genre\'s darkened colour is the average for its genre; the fine, black dotted outline is the average across all songs in the dataset. Comparing the three shows at a glance whether the song is typical for its genre or stands out from it.',
      modalRadarLegSong: 'This song', modalRadarLegGenre: 'Genre average', modalRadarLegAvg: 'Overall average',
      modalRadarLegHint: 'Click a label to show or hide that line.',
      disclaimerTitel: 'Important note',
      disclaimerText: 'Please return to the questionnaire now to continue the survey.',
      thanksTitle: 'Thanks for reading!',
      thanksData: 'This data story is based on the',
      thanksDataName: 'Spotify Songs Dataset (Kaggle, 2023)',
      thanksData2: 'with {n} unique songs. The mood values (valence, energy, danceability) are included in the dataset as pre-computed Spotify metrics.',
      thanksBetreuerin: 'Supervisor:', thanksProf: 'Chair holder:',
      thanksMeta: 'Master\'s thesis, summer semester 2026 · University of Bamberg · Built with Svelte 5 & D3.js v7',
      stimmung: ['😔 Melancholic', '😐 Neutral', '😄 Happy']
  };

  let avgValence = $derived(alleSongs.length ? (d3.mean(alleSongs, s => s.valence) || 0) : 0);
  let popNullStats = $derived.by(() => {
    if (!geladen || alleSongs.length === 0) return null;
    const mitNull = alleSongs.filter(s => s.pop === 0);
    return {
      pct: (mitNull.length / alleSongs.length) * 100,
      valNull: d3.mean(mitNull, s => s.valence) || 0,
      valGesamt: avgValence
    };
  });
  // Mood categories are derived from the dataset itself (upper and lower valence quartile)
  // rather than from fixed cutoffs. Valence = 0 is excluded: in this dataset those entries are
  // almost exclusively ambient tracks without a meaningful mood value.
  let valenceQuartil = $derived.by(() => {
    if (!geladen || alleSongs.length === 0) return { ob: 0.6, unt: 0.4 };
    const werte = alleSongs.filter(s => s.valence > 0).map(s => s.valence).sort(d3.ascending);
    return {
      ob:  d3.quantile(werte, 0.75) ?? 0.6,
      unt: d3.quantile(werte, 0.25) ?? 0.4
    };
  });

  let methodikGerendert = $derived(
    t.methodik.map(m => {
      if (m.p.includes('{pct}')) {
        const s = popNullStats;
        return {
          ...m,
          p: m.p
            .replace('{pct}', s ? s.pct.toFixed(1) : '…')
            .replace('{valNull}', s ? s.valNull.toFixed(2) : '…')
            .replace('{valGesamt}', s ? s.valGesamt.toFixed(2) : '…')
        };
      }
      if (m.p.includes('{happyQ}')) {
        const q = geladen && alleSongs.length ? valenceQuartil : null;
        return {
          ...m,
          p: m.p
            .replace('{happyQ}', q ? q.ob.toFixed(2) : '…')
            .replace('{melaQ}', q ? q.unt.toFixed(2) : '…')
        };
      }
      return m;
    })
  );
  let zeigeSeitenTip = $derived(
    (tipPhase === 1 && extremeGezeichnet) ||
    (tipPhase === 2 && genreGezeichnet) ||
    (tipPhase === 3 && moodGezeichnet)
  );
  let happySongs = $derived([...alleSongs].sort((a, b) => b.valence - a.valence || b.pop - a.pop).slice(0, 3));
  let sadSongs = $derived.by(() => {
    const echte = alleSongs.filter(s => s.valence > 0);
    if (echte.length === 0) return [];
    const sortiertNachValence = [...echte].sort((a, b) => a.valence - b.valence);
    const bandGroesse = Math.max(3, Math.ceil(sortiertNachValence.length * 0.1));
    const melancholischesBand = sortiertNachValence.slice(0, bandGroesse);
    const topDrei = [...melancholischesBand].sort((a, b) => b.pop - a.pop).slice(0, 3);
    const wunsch = melancholischesBand.find(s => s.titel.toLowerCase().includes('lose you to love me'));
    if (!wunsch) return topDrei;
    return [wunsch, ...topDrei.filter(s => s !== wunsch)].slice(0, 3);
  });

  let beeswarmDaten = $derived.by(() => {
    if (!geladen || alleSongs.length === 0) return [];
    const valid = alleSongs.filter(s => s.valence > 0);
    const extremeIds = new Set([...happySongs, ...sadSongs].map(s => s.titel + '|' + s.artist));
    // Scatterplot density encoding against overplotting: songs are binned on a 0.02 grid and the
    // bin count is used as a local density estimate. Point area is proportional to density (radius
    // on a square-root scale) and opacity increases with density.
    const BIN = 0.02;
    const binKey = s => Math.round(s.valence / BIN) + '|' + Math.round(s.energy / BIN);
    const zaehler = new Map();
    valid.forEach(s => { const k = binKey(s); zaehler.set(k, (zaehler.get(k) || 0) + 1); });
    const maxDichte = Math.max(1, ...zaehler.values());
    const opacitySkala = d3.scalePow().exponent(0.5).domain([1, maxDichte]).range([0.16, 0.85]).clamp(true);
    const radiusSkala  = d3.scalePow().exponent(0.5).domain([1, maxDichte]).range([1.5, 3.6]).clamp(true);
    return valid.map(s => {
      const dichte = zaehler.get(binKey(s));
      return {
        ...s,
        istExtrem: extremeIds.has(s.titel + '|' + s.artist),
        dichteOpacity: opacitySkala(dichte),
        dichteRadius: radiusSkala(dichte)
      };
    });
  });

  let topSong = $derived.by(() => {
    if (!geladen || alleSongs.length === 0) return null;
    return [...alleSongs].sort((a, b) => b.pop - a.pop || a.titel.localeCompare(b.titel))[0];
  });

  let top5 = $derived.by(() => {
    if (!geladen || alleSongs.length === 0) return [];
    return [...alleSongs].sort((a, b) => b.pop - a.pop || a.titel.localeCompare(b.titel)).slice(0, 5);
  });

  let topHitsNeutral = $derived.by(() => {
    if (top5.length === 0) return null;
    const schwelle = top5[top5.length - 1].pop;
    const kandidaten = alleSongs.filter(s => s.pop >= schwelle);
    const alleNeutral = kandidaten.every(s => s.valence > valenceQuartil.unt && s.valence < valenceQuartil.ob);
    return alleNeutral ? top5 : null;
  });

  let moodBeispiele = $derived.by(() => {
    if (!geladen || alleSongs.length === 0) return [];
    const froh = alleSongs.filter(s => s.valence >= valenceQuartil.ob);
    const mela = alleSongs.filter(s => s.valence <= valenceQuartil.unt && s.valence > 0);
    if (froh.length === 0 || mela.length === 0) return [];
    const nachPop = (a, b) => b.pop - a.pop || a.titel.localeCompare(b.titel);
    const dezilTop = (arr, get) => {
      const sortiert = [...arr].sort((a, b) => get(b) - get(a) || nachPop(a, b));
      const band = sortiert.slice(0, Math.max(3, Math.ceil(sortiert.length * 0.1)));
      return [...band].sort(nachPop)[0];
    };
    const eFroh = dezilTop(froh, s => s.energy);
    const dMela = dezilTop(mela, s => s.dance);
    const popListe = [...froh, ...mela].sort(nachPop);
    const pTop = popListe.find(s => s !== eFroh && s !== dMela) ?? popListe[0];
    return [
      { rolle: t.mb1, regel: t.mb1r, song: eFroh, farbe: SPOTIFY },
      { rolle: t.mb2, regel: t.mb2r, song: dMela, farbe: '#4f46e5' },
      { rolle: t.mb3, regel: t.mb3r, song: pTop, farbe: pTop.valence >= valenceQuartil.ob ? SPOTIFY : '#4f46e5' }
    ];
  });

  let genreDaten = $derived(
    genres.map(g => {
      const songs = alleSongs.filter(s => s.genre === g);
      return { genre: g, farbe: genreFarben[g], valence: +(d3.mean(songs, s => s.valence) || 0).toFixed(3) };
    }).sort((a, b) => b.valence - a.valence)
  );

  let exploreGefiltert = $derived(genre !== 'alle' || stimmungFilter !== 'alle');
  let gefilterteSongs = $derived.by(() => {
    let basis = genre === 'alle' ? [...alleSongs] : [...alleSongs].filter(s => s.genre === genre);
    if (stimmungFilter === 'happy')
      return basis.filter(s => s.valence >= valenceQuartil.ob).sort((a, b) => b.valence - a.valence).slice(0, 50);
    if (stimmungFilter === 'sad')
      return basis.filter(s => s.valence <= valenceQuartil.unt && s.valence > 0)
                  .sort((a, b) => a.valence - b.valence).slice(0, 50)
                  .sort((a, b) => b.valence - a.valence);
    if (exploreGefiltert) {
      const happy = basis.filter(s => s.valence >= valenceQuartil.ob).sort((a, b) => b.valence - a.valence).slice(0, 25);
      const sad   = basis.filter(s => s.valence <= valenceQuartil.unt && s.valence > 0)
                         .sort((a, b) => a.valence - b.valence).slice(0, 25)
                         .sort((a, b) => b.valence - a.valence);
      return [...happy, { titel: '· · · · ·', valence: null, isSeparator: true }, ...sad];
    }
    const happy = basis.filter(s => s.valence >= valenceQuartil.ob).sort((a, b) => b.valence - a.valence).slice(0, 5);
    const sad   = basis.filter(s => s.valence <= valenceQuartil.unt && s.valence > 0)
                       .sort((a, b) => a.valence - b.valence).slice(0, 5)
                       .sort((a, b) => b.valence - a.valence);
    return [...happy, { titel: '· · · · ·', valence: null, isSeparator: true }, ...sad];
  });

  let suchergebnisse = $derived.by(() => {
    if (suchbegriff.trim().length < 2) return [];
    const tt = suchbegriff.toLowerCase();
    return [...alleSongs]
      .filter(s => s.titel.toLowerCase().includes(tt) || s.artist.toLowerCase().includes(tt))
      .sort((a, b) => b.pop - a.pop)
      .slice(0, 50);
  });

  let modalDaten = $derived.by(() => {
    if (!modalGenre || !geladen) return null;
    const gs = alleSongs.filter(s => s.genre === modalGenre && s.valence > 0);
    if (gs.length === 0) return null;
    const sortV = [...gs].sort((a, b) => b.valence - a.valence);
    return {
      genre: modalGenre,
      label: t.genreLabels[modalGenre],
      farbe: genreFarben[modalGenre],
      anzahl: gs.length,
      valence: d3.mean(gs, s => s.valence),
      energy:  d3.mean(gs, s => s.energy),
      dance:   d3.mean(gs, s => s.dance),
      popNorm: (d3.mean(gs, s => s.pop) || 0) / 100,
      pop:     d3.mean(gs, s => s.pop) || 0,
      min: sortV[sortV.length - 1].valence,
      max: sortV[0].valence,
      hellster: sortV[0],
      dunkelster: sortV[sortV.length - 1]
    };
  });

  let schritte = $derived.by(() => {
    return t.navLabels.map((label, i) => {
      const p = i + 1;
      const fill = phase > p ? 1 : phase === p ? 1 : 0;
      return { label, done: phase > p, aktiv: phase === p, fill, go: () => gehePhase(p) };
    });
  });
  let aktivIndex = $derived(schritte.findIndex(s => s.aktiv));
  let knobPct = $derived(aktivIndex < 0 ? 0 : (aktivIndex + (schritte[aktivIndex]?.fill ?? 1)) / schritte.length * 100);

  $effect(() => {
    if (!geladen || phase < 1 || extremeGezeichnet || beeswarmDaten.length === 0) return;
    tick().then(() => { if (!d3.select('#svg-beeswarm').empty()) { zeichneExtreme(); extremeGezeichnet = true; } });
  });
  $effect(() => {
    if (!geladen || phase < 2 || genreGezeichnet || genreDaten.length === 0) return;
    tick().then(() => { if (!d3.select('#svg-genre').empty()) { zeichneGenre(); genreGezeichnet = true; } });
  });
  $effect(() => {
    if (!geladen || phase < 3 || moodGezeichnet || moodBeispiele.length === 0) return;
    tick().then(() => { if (!d3.select('#svg-mood-0').empty()) { zeichneMoodRadare(); moodGezeichnet = true; } });
  });
  $effect(() => { const _g = genre; const _f = stimmungFilter; if (geladen) tick().then(zeichneExploration); });
  let s1Fokus = $state(null);
  function extremFokus(lager) {
    s1Fokus = s1Fokus === lager ? null : lager;
    extremAusgewaehlt = null;
    markiereAuswahl();
    wendeExtremFokusAn();
  }
  function wendeExtremFokusAn() {
    maleWolke();
    const svg = d3.select('#svg-beeswarm');
    if (svg.empty()) return;
    const happyTitel = new Set(happySongs.map(s => s.titel));
    svg.selectAll('.extrem-punkt, .extrem-halo')
      .attr('opacity', function () {
        if (!s1Fokus) return 1;
        const istHappy = happyTitel.has(this.dataset.titel);
        return (s1Fokus === 'happy') === istHappy ? 1 : 0.15;
      });
    svg.selectAll('.fokus-rahmen').remove();
    if (s1Fokus) {
      const b = { o: 30, r: 60, u: 64, l: 72 };
      const x = d3.scaleLinear().domain([0, 1]).range([b.l, 760 - b.r]);
      const von = s1Fokus === 'sad' ? 0 : valenceQuartil.ob;
      const bis = s1Fokus === 'sad' ? valenceQuartil.unt : 1;
      const pad = 4.5;
      const xs = x(von) - pad;
      svg.append('rect').attr('class', 'fokus-rahmen')
        .attr('x', xs).attr('y', b.o - pad)
        .attr('width', x(bis) + pad - xs).attr('height', 380 - b.u - b.o + 2 * pad)
        .attr('fill', 'none').attr('stroke', '#1a1a2e').attr('stroke-width', 2.5)
        .attr('rx', 6).attr('pointer-events', 'none');
    }
  }

  function markiereAuswahl() {
    const sel = extremAusgewaehlt;
    const svg = d3.select('#svg-beeswarm');
    svg.selectAll('.auswahl-ring').remove();
    svg.selectAll('.extrem-halo')
      .interrupt()
      .attr('opacity', function () { return !sel || this.dataset.titel === sel.titel ? 1 : 0.35; });
    svg.selectAll('.extrem-punkt')
      .interrupt()
      .attr('opacity', function () { return !sel || this.dataset.titel === sel.titel ? 1 : 0.35; })
      .attr('fill', function () { return this.dataset.titel === sel?.titel ? '#FFD700' : this.dataset.farbe; })
      .attr('stroke', function () { return this.dataset.titel === sel?.titel ? '#1a1a2e' : 'white'; })
      .attr('stroke-width', function () { return this.dataset.titel === sel?.titel ? 3 : 2; })
      .attr('r', function () { return this.dataset.titel === sel?.titel ? 13 : 8; })
      .each(function () {
        if (this.dataset.titel !== sel?.titel) return;
        d3.select(this).raise();
        d3.select(this.parentNode).append('circle')
          .attr('class', 'auswahl-ring').attr('pointer-events', 'none')
          .attr('cx', this.getAttribute('cx')).attr('cy', this.getAttribute('cy'))
          .attr('r', 19).attr('fill', 'none')
          .attr('stroke', '#FFD700').attr('stroke-width', 3);
      });
  }

  $effect(() => {
    markiereAuswahl();
  });

  // Deliberately not el.scrollIntoView(): newly revealed sections start with a transform offset
  // that only animates to zero over ~0.85 s. getBoundingClientRect(), which scrollIntoView relies
  // on, would therefore aim at the pre-animation position. offsetTop is a pure layout property
  // and is not affected by the transform.
  function scrolleZuAbschnitt(id) {
    const el = document.getElementById(id);
    if (!el) return;
    let oben = 0;
    let knoten = el;
    while (knoten) { oben += knoten.offsetTop || 0; knoten = knoten.offsetParent; }
    window.scrollTo({ top: oben, behavior: 'smooth' });
  }

  function tourStarten() {
    tourGesehen = true;
    tourSchritt = 1;
    tick().then(() => { scrolleZuAbschnitt('sec-4'); });
  }

  function introStarten() {
    introSichtbar = true;
    scrollGesperrt = false;
    tick().then(() => { scrolleZuAbschnitt('sec-0-text'); });
  }

  $effect(() => {
    if (typeof window === 'undefined' || !scrollGesperrt) return;
    const istBedienelement = (ziel) => ziel instanceof HTMLElement && (ziel.tagName === 'BUTTON' || ziel.tagName === 'A' || ziel.tagName === 'INPUT');
    const istInModal = (ziel) => ziel instanceof Element && ziel.closest('.modal');
    const blockiereRad = (e) => { if (!istInModal(e.target)) e.preventDefault(); };
    const blockiereTasten = (e) => {
      if (istBedienelement(e.target) || istInModal(e.target)) return;
      if (['ArrowDown', 'ArrowUp', 'PageDown', 'PageUp', 'End', 'Home', ' '].includes(e.key)) e.preventDefault();
    };
    window.addEventListener('wheel', blockiereRad, { passive: false });
    window.addEventListener('touchmove', blockiereRad, { passive: false });
    window.addEventListener('keydown', blockiereTasten, { passive: false });
    return () => {
      window.removeEventListener('wheel', blockiereRad);
      window.removeEventListener('touchmove', blockiereRad);
      window.removeEventListener('keydown', blockiereTasten);
    };
  });

  $effect(() => {
    if (typeof window === 'undefined') return;
    let abgemeldet = false;
    tick().then(() => {
      if (abgemeldet) return;
      const abschnitte = document.querySelectorAll('.story-abschnitt');
      if (beobachter) beobachter.disconnect();
      if (tipBeobachter) tipBeobachter.disconnect();
      beobachter = new IntersectionObserver((entries) => {
        let bester = null;
        entries.forEach(e => {
          if (e.isIntersecting && (!bester || e.intersectionRatio > bester.intersectionRatio)) bester = e;
        });
        if (bester) {
          const p = +bester.target.dataset.phase;
          if (!Number.isNaN(p) && p !== phase) phase = p;
          let oben = 0, k = bester.target;
          while (k) { oben += k.offsetTop || 0; k = k.offsetParent; }
          phaseTop = oben;
        }
      }, { threshold: [0.1, 0.25, 0.45, 0.6, 0.75], rootMargin: '-15% 0px -50% 0px' });
      abschnitte.forEach(el => beobachter.observe(el));
      tipBeobachter = new IntersectionObserver((entries) => {
        let besterTip = null;
        entries.forEach(e => {
          if (e.isIntersecting && e.intersectionRatio >= 0.45 && (!besterTip || e.intersectionRatio > besterTip.intersectionRatio)) besterTip = e;
        });
        if (besterTip) {
          const tp = +besterTip.target.dataset.phase;
          if (!Number.isNaN(tp) && tp !== tipPhase) tipPhase = tp;
          let ziel = besterTip.target;
          if (tp === 0) { const t0 = document.getElementById('sec-0-text'); if (t0) ziel = t0; }
          let oben = 0, k = ziel;
          while (k) { oben += k.offsetTop || 0; k = k.offsetParent; }
          tipTop = oben;
        }
      }, { threshold: [0.1, 0.25, 0.45, 0.6, 0.75] });
      abschnitte.forEach(el => tipBeobachter.observe(el));
    });
    return () => { abgemeldet = true; if (beobachter) beobachter.disconnect(); if (tipBeobachter) tipBeobachter.disconnect(); };
  });

  $effect(() => {
    // Reading these values registers them as dependencies of the effect, so that the
    // enlarged radar chart is redrawn when a line is toggled. The bindings are not used.
    const _a = ausgewaehlt, _s = radarZeigeSong, _g = radarZeigeGenre, _o = radarZeigeAvg;
    if (modalGenre && geladen) tick().then(zeichneRadarGross);
  });

  $effect(() => {
    if (!introSichtbar) return;
    tick().then(() => {
      const el = document.getElementById('sec-0-text');
      if (el && tipPhase === 0) {
        let oben = 0, k = el;
        while (k) { oben += k.offsetTop || 0; k = k.offsetParent; }
        if (oben > 0) tipTop = oben;
      }
    });
  });

  $effect(() => {
    if (typeof window === 'undefined' || !geladen) return;
    let abgemeldet = false;
    const klassenPaar = (el) => {
      if (el.classList.contains('fly')) return ['fly-pre', 'fly-go'];
      return ['reveal-pre', 'reveal-go'];
    };
    tick().then(() => {
      if (abgemeldet) return;
      const ziele = document.querySelectorAll('.fly, .reveal');
      if (zuflieger) zuflieger.disconnect();
      zuflieger = new IntersectionObserver((entries) => {
        entries.forEach(e => {
          if (e.isIntersecting) {
            const [pre, go] = klassenPaar(e.target);
            e.target.classList.remove(pre);
            e.target.classList.add(go);
            zuflieger.unobserve(e.target);
          }
        });
      }, { threshold: 0.15, rootMargin: '0px 0px -10% 0px' });
      ziele.forEach(el => {
        if (el.getBoundingClientRect().top < window.innerHeight * 0.9) return;
        const [pre] = klassenPaar(el);
        el.classList.add(pre);
        zuflieger.observe(el);
      });
    });
    return () => { abgemeldet = true; if (zuflieger) zuflieger.disconnect(); };
  });

  onMount(async () => {
    const roh = await d3.csv('/spotify_songs.csv', z => ({
      titel:   z.track_name,  artist: z.track_artist, album: z.track_album_name,
      trackId: z.track_id,    pop: +z.track_popularity, dance: +z.danceability,
      energy:  +z.energy,     valence: +z.valence,   genre: z.playlist_genre
    }));
    const songMap = new Map();
    roh.forEach(s => {
      const key = s.titel + '|' + s.artist;
      if (!s.titel || Number.isNaN(s.pop)) return;
      if (!songMap.has(key) || s.pop > songMap.get(key).pop) songMap.set(key, s);
    });
    alleSongs = [...songMap.values()];
    geladen = true;
    if (window.scrollY < 50) {
      requestAnimationFrame(() => { if (window.scrollY < 50) window.scrollTo(0, 0); });
    }
  });

  function maleWolke() {
    const cvs = document.getElementById('canvas-beeswarm');
    if (!cvs || beeswarmDaten.length === 0) return;
    const b = { o: 30, r: 60, u: 64, l: 72 };
    const x = d3.scaleLinear().domain([0, 1]).range([b.l, 760 - b.r]);
    const y = d3.scaleLinear().domain([0, 1]).range([380 - b.u, b.o]);
    const farbeFor = v => d3.interpolateRgb('#4f46e5', SPOTIFY)(v);
    const ctx = cvs.getContext('2d');
    ctx.clearRect(0, 0, 760, 380);
    beeswarmDaten.filter(d => !d.istExtrem).forEach(d => {
      let alpha = d.dichteOpacity;
      if (s1Fokus === 'sad' && !(d.valence <= valenceQuartil.unt)) alpha *= 0.08;
      if (s1Fokus === 'happy' && !(d.valence >= valenceQuartil.ob)) alpha *= 0.08;
      ctx.globalAlpha = alpha;
      ctx.fillStyle = farbeFor(d.valence);
      ctx.beginPath();
      ctx.arc(x(d.valence), y(d.energy), d.dichteRadius, 0, 2 * Math.PI);
      ctx.fill();
    });
    ctx.globalAlpha = 1;
  }

  function zeichneExtreme() {
    const el = d3.select('#svg-beeswarm');
    el.selectAll('*').remove();
    if (beeswarmDaten.length === 0) return;

    const [w, h, b] = [760, 380, { o: 30, r: 60, u: 64, l: 72 }];
    const x = d3.scaleLinear().domain([0, 1]).range([b.l, w - b.r]);
    const y = d3.scaleLinear().domain([0, 1]).range([h - b.u, b.o]);
    const farbeFor = v => d3.interpolateRgb('#4f46e5', SPOTIFY)(v);

    el.append('g').attr('class', 'grid')
      .selectAll('line.vgrid').data(x.ticks(5)).join('line').attr('class', 'vgrid')
      .attr('x1', d => x(d)).attr('x2', d => x(d)).attr('y1', b.o).attr('y2', h - b.u)
      .attr('stroke', '#ececec').attr('stroke-width', 1);
    el.append('g').attr('class', 'grid')
      .selectAll('line.hgrid').data(y.ticks(5)).join('line').attr('class', 'hgrid')
      .attr('x1', b.l).attr('x2', w - b.r).attr('y1', d => y(d)).attr('y2', d => y(d))
      .attr('stroke', '#ececec').attr('stroke-width', 1);
    el.append('rect').attr('x', b.l).attr('y', b.o)
      .attr('width', w - b.l - b.r).attr('height', h - b.u - b.o)
      .attr('fill', 'none').attr('stroke', '#d4d4d4').attr('stroke-width', 1);

    el.append('g').attr('transform', `translate(0,${h - b.u})`)
      .call(d3.axisBottom(x).ticks(5).tickFormat(d3.format('.1f')))
      .call(g => g.select('.domain').attr('stroke', '#bbb'))
      .selectAll('text').attr('font-size', '11px').attr('fill', '#666');
    el.append('g').attr('transform', `translate(${b.l},0)`)
      .call(d3.axisLeft(y).ticks(5).tickFormat(d3.format('.1f')))
      .call(g => g.select('.domain').attr('stroke', '#bbb'))
      .selectAll('text').attr('font-size', '11px').attr('fill', '#666');
    el.append('text').attr('x', b.l + (w - b.l - b.r) / 2).attr('y', h - 24)
      .attr('text-anchor', 'middle').attr('font-size', '12.5px').attr('font-weight', '700').attr('fill', '#555')
      .text(t.scatterXTitle);
    el.append('text').attr('x', b.l + (w - b.l - b.r) / 2).attr('y', h - 8)
      .attr('text-anchor', 'middle').attr('font-size', '11px').attr('fill', '#999')
      .text(t.scatterXAxis);
    el.append('text').attr('x', -((h - b.u + b.o) / 2)).attr('y', 18)
      .attr('transform', 'rotate(-90)')
      .attr('text-anchor', 'middle').attr('font-size', '12.5px').attr('font-weight', '700').attr('fill', '#555')
      .text(t.scatterYTitle);
    el.append('text').attr('x', -((h - b.u + b.o) / 2)).attr('y', 32)
      .attr('transform', 'rotate(-90)')
      .attr('text-anchor', 'middle').attr('font-size', '11px').attr('fill', '#999')
      .text(t.scatterYAxis);

    maleWolke();

    const rEx = 8;
    const zeichneGruppe = (gEx, list) => {
      list.map(d => ({ d, px: x(d.valence), py: y(d.energy) })).forEach(({ d, px, py }, i) => {
        gEx.append('circle').attr('cx', px).attr('cy', py).attr('r', 0)
          .attr('class', 'extrem-halo').attr('data-titel', d.titel)
          .attr('fill', '#FFD700')
          .transition().delay(i * 260).duration(520).attr('r', rEx + 3);
        gEx.append('circle').attr('cx', px).attr('cy', py).attr('r', 0)
          .attr('class', 'extrem-punkt').attr('data-titel', d.titel).attr('data-farbe', farbeFor(d.valence))
          .attr('fill', farbeFor(d.valence)).attr('stroke', 'white').attr('stroke-width', 2)
          .style('cursor', 'pointer')
          .on('mouseenter', function () {
            const hx = +this.getAttribute('cx'), hy = +this.getAttribute('cy');
            const oben = hy > 46;
            const g = d3.select(this.parentNode).append('g').attr('class', 'klick-tip').attr('pointer-events', 'none');
            const gewaehlt = extremAusgewaehlt?.titel === d.titel;
            g.append('rect').attr('x', hx - 46).attr('y', oben ? hy - 36 : hy + 14)
              .attr('width', 92).attr('height', 20).attr('rx', 6).attr('fill', '#1a1a2e');
            g.append('text').attr('class', 'klick-tip-text').attr('x', hx).attr('y', oben ? hy - 22 : hy + 28).attr('text-anchor', 'middle')
              .attr('font-size', '10px').attr('font-weight', '700').attr('fill', '#fff')
              .text(gewaehlt ? '👆 Click to reset' : '👆 Click here');
          })
          .on('mouseleave', function () { d3.select(this.parentNode).selectAll('.klick-tip').remove(); })
          .on('click', function () {
            s1Fokus = null;
            wendeExtremFokusAn();
            extremAusgewaehlt = extremAusgewaehlt?.titel === d.titel ? null : d;
            markiereAuswahl();
            d3.select(this.parentNode).selectAll('.klick-tip-text')
              .text(extremAusgewaehlt?.titel === d.titel ? '👆 Click to reset' : '👆 Click here');
          })
          .transition().delay(i * 260).duration(520).attr('r', rEx);
      });
    };

    setTimeout(() => {
      if (d3.select('#svg-beeswarm').empty()) return;
      const gEx = el.append('g').attr('class', 'extreme-layer');
      zeichneGruppe(gEx, happySongs);
      zeichneGruppe(gEx, sadSongs);
    }, 200);
  }

  let deepFokus = $state(null);
  function wendeDeepFokusAn() {
    const svg = d3.select('#svg-genre');
    if (svg.empty()) return;
    const zeigeAvg = deepFokus !== 'avg';
    svg.selectAll('.avg-marke').attr('display', zeigeAvg ? null : 'none');
    svg.selectAll('.neutral-linie').attr('display', zeigeAvg ? null : 'none');
    svg.selectAll('.fokus-rahmen').remove();
    const ziel = deepFokus === 'latin' ? { genre: 'latin', von: 0.70, bis: 0.95 }
               : deepFokus === 'edm'   ? { genre: 'edm',   von: 0.15, bis: 0.40 }
               : null;
    svg.selectAll('.zone').interrupt().attr('opacity', d => !ziel || d.genre === ziel.genre ? 1 : 0.18);
    svg.selectAll('.avg-marke').interrupt().attr('opacity', d => !ziel || d.genre === ziel.genre ? 1 : 0.15);
    if (ziel) {
      const b = { o: 20, r: 40, u: 70, l: 150 };
      const x = d3.scaleLinear().domain([0, 1]).range([b.l, 620 - b.r]);
      const y = d3.scaleBand().domain(genreDaten.map(d => d.genre)).range([b.o, 320 - b.u]).padding(0.3);
      svg.append('rect').attr('class', 'fokus-rahmen')
        .attr('x', x(ziel.von)).attr('y', y(ziel.genre) - 3)
        .attr('width', x(ziel.bis) - x(ziel.von)).attr('height', y.bandwidth() + 6)
        .attr('fill', 'none').attr('stroke', '#1a1a2e').attr('stroke-width', 2.5)
        .attr('rx', 5).attr('pointer-events', 'none');
    }
  }
  $effect(() => { const _f = deepFokus; if (geladen) tick().then(wendeDeepFokusAn); });

  $effect(() => {
    const z = zoomKarte;
    if (z === null || !geladen) return;
    tick().then(() => {
      const el = d3.select('#svg-mood-zoom');
      if (!el.empty() && moodBeispiele[z]) zeichneEinRadar(el, moodBeispiele[z], 0);
    });
  });

  function zeichneGenre() {
    const el = d3.select('#svg-genre');
    el.selectAll('*').remove();
    deepFokus = null;
    const [w, h, b] = [620, 320, { o: 20, r: 40, u: 70, l: 150 }];
    const x = d3.scaleLinear().domain([0, 1]).range([b.l, w - b.r]);
    const y = d3.scaleBand().domain(genreDaten.map(d => d.genre)).range([b.o, h - b.u]).padding(0.3);

    // Heat-strip: each genre's valence distribution in 0.05 bins. Cell shading encodes the song
    // count per bin on a square-root scale, normalized per genre, so that genres of different size
    // remain comparable; the dark line marks the genre mean.
    const BIN = 0.05, N_BINS = Math.round(1 / BIN);
    const zonen = genreDaten.map(d => {
      const bins = new Array(N_BINS).fill(0);
      alleSongs.forEach(s => { if (s.genre === d.genre) bins[Math.min(N_BINS - 1, Math.floor(s.valence / BIN))] += 1; });
      const deckkraft = d3.scalePow().exponent(0.5).domain([0, Math.max(1, ...bins)]).range([0, 0.95]);
      return { ...d, zellen: bins.map((n, i) => ({ i, x0: i * BIN, n, o: n === 0 ? 0 : Math.max(0.06, deckkraft(n)) })) };
    });

    const ZELLE = 550;
    const reihen = el.selectAll('.zone').data(zonen).join('g').attr('class', 'zone');
    reihen.append('rect')
      .attr('x', x(0)).attr('y', d => y(d.genre))
      .attr('width', x(1) - x(0)).attr('height', y.bandwidth())
      .attr('fill', '#000').attr('opacity', 0.035).attr('rx', 4);
    reihen.each(function (d) {
      const zellen = d3.select(this).selectAll('.zelle').data(d.zellen.filter(z => z.n > 0)).join('rect')
        .attr('class', 'zelle')
        .attr('x', z => x(z.x0)).attr('y', y(d.genre))
        .attr('width', x(BIN) - x(0) + 0.5).attr('height', y.bandwidth())
        .attr('fill', d.farbe).attr('opacity', 0);
      zellen.append('title').text(z => `${t.genreLabels[d.genre]} · mood ${z.x0.toFixed(2)}–${(z.x0 + BIN).toFixed(2)}: ${z.n.toLocaleString('en')} songs`);
      zellen.transition().delay(z => 150 + z.i * 26).duration(ZELLE).attr('opacity', z => z.o);
    });

    const EINBLEND = 150 + (N_BINS - 1) * 26 + ZELLE;
    const avgG = el.selectAll('.avg-marke').data(zonen).join('g').attr('class', 'avg-marke')
      .attr('opacity', 0).attr('pointer-events', 'none');
    avgG.append('line')
      .attr('x1', d => x(d.valence)).attr('x2', d => x(d.valence))
      .attr('y1', d => y(d.genre) - 3).attr('y2', d => y(d.genre) + y.bandwidth() + 3)
      .attr('stroke', '#1a1a2e').attr('stroke-width', 2.5);
    avgG.append('text')
      .attr('x', d => x(d.valence) + 5).attr('y', d => y(d.genre) + y.bandwidth() / 2 + 4)
      .attr('text-anchor', 'start').attr('font-size', '11.5px').attr('font-weight', '700').attr('fill', '#1a1a2e')
      .attr('stroke', 'white').attr('stroke-width', 2.5).attr('paint-order', 'stroke')
      .text(d => d.valence.toFixed(2));
    avgG.transition().delay(EINBLEND + 150).duration(450).attr('opacity', 1);

    const linie = el.append('line').attr('class', 'neutral-linie')
      .attr('x1', x(0.5)).attr('x2', x(0.5)).attr('y1', b.o).attr('y2', h - b.u)
      .attr('stroke', '#bbb').attr('stroke-dasharray', '5,4').attr('opacity', 0);
    linie.transition().delay(EINBLEND + 700).duration(550).attr('opacity', 1);
    const neutralTxt = el.append('text').attr('class', 'neutral-linie').attr('x', x(0.5)).attr('y', b.o - 6)
      .attr('text-anchor', 'middle').text(t.neutral).attr('font-size', '10px').attr('fill', '#999').attr('opacity', 0);
    neutralTxt.transition().delay(EINBLEND + 700).duration(550).attr('opacity', 1);

    el.append('g').attr('transform', `translate(0,${h - b.u})`).call(d3.axisBottom(x).ticks(5).tickFormat(d3.format('.1f')))
      .selectAll('text').attr('font-size', '11px').attr('fill', '#555');
    const ya = el.append('g').attr('transform', `translate(${b.l},0)`)
      .call(d3.axisLeft(y).tickFormat(g => t.genreLabels[g]));
    ya.selectAll('text').attr('font-size', '12px').attr('font-weight', '600').attr('fill', '#333');
    el.append('text')
      .attr('x', b.l + (w - b.l - b.r) / 2).attr('y', h - 38)
      .attr('text-anchor', 'middle').attr('font-size', '12.5px').attr('font-weight', '700').attr('fill', '#555')
      .text(t.scatterXTitle);
    el.append('text')
      .attr('x', b.l + (w - b.l - b.r) / 2).attr('y', h - 22)
      .attr('text-anchor', 'middle').attr('font-size', '11px').attr('fill', '#999')
      .text(t.genreXAxis);
    el.append('text').attr('transform', 'rotate(-90)').attr('x', -(h / 2)).attr('y', 16)
      .attr('text-anchor', 'middle').attr('font-size', '12.5px').attr('font-weight', '700').attr('fill', '#555').text(t.genreYAxis);
  }

  function zeichneEinRadar(el, mb, i) {
    el.selectAll('*').remove();
    const axes = radarAchsen();
    const avgAll = {
      valence: d3.mean(alleSongs, s => s.valence) || 0,
      energy:  d3.mean(alleSongs, s => s.energy)  || 0,
      dance:   d3.mean(alleSongs, s => s.dance)   || 0,
      popNorm: (d3.mean(alleSongs, s => s.pop) || 0) / 100
    };
    const r = 78, cx = 130, cy = 122;
    const grp = el.append('g').attr('transform', `translate(${cx},${cy})`);
    [0.25, 0.5, 0.75, 1.0].forEach(tt => {
      const pts = axes.map(a => `${tt*r*Math.cos(a.angle)},${tt*r*Math.sin(a.angle)}`).join(' ');
      grp.append('polygon').attr('points', pts).attr('fill', 'none').attr('stroke', '#e2e2ea');
    });
    const song = { valence: mb.song.valence, energy: mb.song.energy, dance: mb.song.dance, popNorm: (mb.song.pop || 0) / 100 };
    axes.forEach(a => {
      grp.append('line').attr('x2', r * Math.cos(a.angle)).attr('y2', r * Math.sin(a.angle)).attr('stroke', '#d0d0da');
      grp.append('text')
        .attr('x', (r + 22) * Math.cos(a.angle)).attr('y', (r + 22) * Math.sin(a.angle) - 4)
        .attr('text-anchor', 'middle').attr('dominant-baseline', 'middle')
        .attr('font-size', '11px').attr('font-weight', '600').attr('fill', '#444').text(a.label);
      grp.append('text')
        .attr('x', (r + 22) * Math.cos(a.angle)).attr('y', (r + 22) * Math.sin(a.angle) + 9)
        .attr('text-anchor', 'middle').attr('dominant-baseline', 'middle')
        .attr('font-size', '10px').attr('font-weight', 'bold').attr('fill', mb.farbe)
        .attr('opacity', 0)
        .text(a.key === 'popNorm' ? Math.round(mb.song.pop).toString() : song[a.key].toFixed(2))
        .transition().delay(650 + i * 350).duration(300).attr('opacity', 1);
    });
    const avgPts = axes.map(a => `${avgAll[a.key]*r*Math.cos(a.angle)},${avgAll[a.key]*r*Math.sin(a.angle)}`).join(' ');
    grp.append('polygon').attr('points', avgPts).attr('fill', 'none')
      .attr('stroke', '#111').attr('stroke-width', 1.5).attr('stroke-dasharray', '4,4');
    const endPts = axes.map(a => [song[a.key]*r*Math.cos(a.angle), song[a.key]*r*Math.sin(a.angle)]);
    grp.append('polygon')
      .attr('points', axes.map(() => '0,0').join(' '))
      .attr('fill', mb.farbe).attr('fill-opacity', 0.3)
      .attr('stroke', mb.farbe).attr('stroke-width', 2.5).attr('stroke-linejoin', 'round')
      .transition().delay(150 + i * 350).duration(700).ease(d3.easeCubicOut)
      .attrTween('points', () => {
        const interp = endPts.map(p => d3.interpolate([0, 0], p));
        return tt => interp.map(fn => fn(tt).join(',')).join(' ');
      });
    axes.forEach((a, k) => {
      grp.append('circle').attr('cx', 0).attr('cy', 0).attr('r', 3.5).attr('fill', mb.farbe)
        .transition().delay(150 + i * 350).duration(700).ease(d3.easeCubicOut)
        .attr('cx', endPts[k][0]).attr('cy', endPts[k][1]);
    });
  }

  function zeichneMoodRadare() {
    if (moodBeispiele.length === 0) return;
    moodBeispiele.forEach((mb, i) => {
      const el = d3.select('#svg-mood-' + i);
      if (!el.empty()) zeichneEinRadar(el, mb, i);
    });
  }

  function zeichneExploration() {
    const el = d3.select('#svg-exploration');
    el.selectAll('*').remove();
    ausgewaehlt = null;

    const songs = gefilterteSongs;
    const realSongs = songs.filter(s => !s.isSeparator);
    if (realSongs.length === 0) {
      el.attr('viewBox', '0 0 620 60').attr('height', 60);
      el.append('text').attr('x', 310).attr('y', 35)
        .attr('text-anchor', 'middle').attr('font-size', '14px').attr('fill', '#999')
        .text(t.keineSongs);
      return;
    }

    const [w, b] = [620, { o: 34, r: 240, u: 58, l: 64 }];
    const h = songs.length * 33 + 72;
    el.attr('viewBox', `0 0 ${w} ${h}`).attr('height', h);

    const x = d3.scaleLinear().domain([0, 1]).range([b.l, w - b.r]);
    const y = d3.scaleBand().domain(songs.map(d => d.titel)).range([b.o, h - b.u]).padding(0.2);

    el.append('line').attr('x1', b.l).attr('x2', b.l).attr('y1', b.o).attr('y2', h - b.u).attr('stroke', '#ccc');

    const zeichneKopf = (sel, yy) => {
      sel.append('text').attr('x', b.l + 10).attr('y', yy).attr('text-anchor', 'start')
        .attr('font-size', '12.5px').attr('font-weight', '700').attr('fill', '#555')
        .text(t.colSong);
    };

    const ax = d3.select('#svg-expl-axis'); ax.selectAll('*').remove();
    const hd = d3.select('#svg-expl-head'); hd.selectAll('*').remove();
    if (exploreGefiltert) {
      ax.append('g').attr('transform', 'translate(0,8)').call(d3.axisBottom(x).ticks(5).tickFormat(d3.format('.1f')))
        .selectAll('text').attr('font-size', '10px').attr('fill', '#555');
      ax.append('text').attr('x', b.l + (w - b.l - b.r) / 2).attr('y', 42)
        .attr('text-anchor', 'middle').attr('font-size', '12.5px').attr('font-weight', '700').attr('fill', '#555').text(t.scatterXTitle);
      ax.append('text').attr('x', b.l + (w - b.l - b.r) / 2).attr('y', 58)
        .attr('text-anchor', 'middle').attr('font-size', '11px').attr('fill', '#888').text(t.scatterXAxis);
      zeichneKopf(hd, 15);
    } else {
      zeichneKopf(el, 22);
    }

    songs.filter(s => s.isSeparator).forEach(s => {
      el.append('text')
        .attr('x', b.l + 10).attr('y', y(s.titel) + y.bandwidth() / 2 + 5)
        .attr('text-anchor', 'start').attr('font-size', '18px').attr('fill', '#ccc')
        .attr('letter-spacing', '6').text('· · · · ·');
    });

    const balkenFarbe = d => genre === 'alle' ? (genreFarben[d.genre] || '#6366f1') : (genreFarben[genre] || 'steelblue');

    el.selectAll('.klickzeile').data(realSongs).join('rect').attr('class', 'klickzeile')
      .attr('x', 0).attr('y', d => y(d.titel))
      .attr('width', w).attr('height', y.bandwidth())
      .attr('fill', 'transparent').attr('cursor', 'pointer')
      .on('click',     (e, d) => { ausgewaehlt = d; modalGenre = d.genre; radarZeigeSong = radarZeigeGenre = radarZeigeAvg = true; })
      .on('mouseover', function() { d3.select(this).attr('fill', 'rgba(0,0,0,0.04)'); })
      .on('mouseout',  function() { d3.select(this).attr('fill', 'transparent'); });

    el.selectAll('.balken').data(realSongs).join('rect').attr('class', 'balken')
      .attr('x', b.l).attr('y', d => y(d.titel))
      .attr('width', d => Math.max(3, x(d.valence) - b.l)).attr('height', y.bandwidth())
      .attr('fill', balkenFarbe).attr('rx', 4).attr('pointer-events', 'none');

    el.selectAll('.wert').data(realSongs).join('text').attr('class', 'wert')
      .attr('x', b.l - 8).attr('y', d => y(d.titel) + y.bandwidth() / 2 + 4).attr('text-anchor', 'end')
      .text(d => d.valence.toFixed(2)).attr('font-size', '11px').attr('font-weight', '700').attr('fill', '#1a1a2e')
      .attr('pointer-events', 'none');

    el.selectAll('.songname').data(realSongs).join('text').attr('class', 'songname')
      .attr('x', d => x(d.valence) + 8).attr('y', d => y(d.titel) + y.bandwidth() / 2 + 4)
      .attr('text-anchor', 'start').attr('pointer-events', 'none')
      .text(d => d.titel).attr('font-size', '12px').attr('font-weight', '600').attr('fill', '#1a1a2e');

    if (!exploreGefiltert) {
      el.append('g').attr('transform', `translate(0,${h - b.u})`).call(d3.axisBottom(x).ticks(5).tickFormat(d3.format('.1f')))
        .selectAll('text').attr('font-size', '10px').attr('fill', '#555');
      el.append('text').attr('x', b.l + (w - b.l - b.r) / 2).attr('y', h - 26)
        .attr('text-anchor', 'middle').attr('font-size', '12.5px').attr('font-weight', '700').attr('fill', '#555')
        .text(t.scatterXTitle);
      el.append('text').attr('x', b.l + (w - b.l - b.r) / 2).attr('y', h - 8)
        .attr('text-anchor', 'middle').attr('font-size', '11px').attr('fill', '#888')
        .text(t.scatterXAxis);
    }
  }

  function radarAchsen() {
    return [
      { label: t.radarStimmung, key: 'valence', angle: -Math.PI / 2 },
      { label: t.radarEnergie,  key: 'energy',  angle: 0 },
      { label: t.radarPop,      key: 'popNorm', angle: Math.PI / 2 },
      { label: t.radarTanz,     key: 'dance',   angle: Math.PI }
    ];
  }

  const genreAvgFarbe = (farbe) => d3.color(farbe).darker(0.9).formatHex();

  function zeichneRadarGross() {
    const svgEl = d3.select('#svg-radar-modal');
    if (svgEl.empty() || !modalDaten || !ausgewaehlt) return;
    svgEl.selectAll('*').remove();
    const size = 300, r = 108, cx = size / 2, cy = size / 2, padX = 48;
    svgEl.attr('viewBox', `${-padX} 0 ${size + 2 * padX} ${size}`);
    const axes = radarAchsen();
    const farbe = modalDaten.farbe;
    const avgAll = {
      valence: d3.mean(alleSongs, s => s.valence) || 0,
      energy:  d3.mean(alleSongs, s => s.energy)  || 0,
      dance:   d3.mean(alleSongs, s => s.dance)   || 0,
      popNorm: (d3.mean(alleSongs, s => s.pop) || 0) / 100
    };
    const genreAvg = {
      valence: modalDaten.valence, energy: modalDaten.energy,
      dance: modalDaten.dance, popNorm: modalDaten.popNorm
    };
    const song = {
      valence: ausgewaehlt.valence, energy: ausgewaehlt.energy,
      dance: ausgewaehlt.dance, popNorm: (ausgewaehlt.pop || 0) / 100
    };
    const grp = svgEl.append('g').attr('transform', `translate(${cx},${cy})`);
    [0.25, 0.5, 0.75, 1.0].forEach(tt => {
      const pts = axes.map(a => `${tt*r*Math.cos(a.angle)},${tt*r*Math.sin(a.angle)}`).join(' ');
      grp.append('polygon').attr('points', pts).attr('fill', 'none').attr('stroke', '#e2e2ea');
      grp.append('text').attr('x', 3).attr('y', -tt*r + 3).attr('font-size', '8px').attr('fill', '#bbb').text(tt.toFixed(2));
    });
    axes.forEach(a => {
      grp.append('line').attr('x2', r * Math.cos(a.angle)).attr('y2', r * Math.sin(a.angle)).attr('stroke', '#d0d0da');
      grp.append('text')
        .attr('x', (r + 24) * Math.cos(a.angle)).attr('y', (r + 24) * Math.sin(a.angle) - 5)
        .attr('text-anchor', 'middle').attr('dominant-baseline', 'middle')
        .attr('font-size', '13px').attr('font-weight', '600').attr('fill', '#444').text(a.label);
    });
    if (radarZeigeAvg) {
      const avgPts = axes.map(a => `${avgAll[a.key]*r*Math.cos(a.angle)},${avgAll[a.key]*r*Math.sin(a.angle)}`).join(' ');
      grp.append('polygon').attr('points', avgPts).attr('fill', 'none')
        .attr('stroke', '#111').attr('stroke-width', 1.5).attr('stroke-dasharray', '2,5').attr('stroke-linecap', 'round');
    }
    if (radarZeigeGenre) {
      const genrePts = axes.map(a => `${genreAvg[a.key]*r*Math.cos(a.angle)},${genreAvg[a.key]*r*Math.sin(a.angle)}`).join(' ');
      grp.append('polygon').attr('points', genrePts).attr('fill', 'none')
        .attr('stroke', genreAvgFarbe(farbe)).attr('stroke-width', 2.5).attr('stroke-dasharray', '9,4');
    }
    if (radarZeigeSong) {
      const songPts = axes.map(a => `${song[a.key]*r*Math.cos(a.angle)},${song[a.key]*r*Math.sin(a.angle)}`).join(' ');
      grp.append('polygon').attr('points', songPts).attr('fill', farbe).attr('fill-opacity', 0.32)
        .attr('stroke', farbe).attr('stroke-width', 3).attr('stroke-linejoin', 'round');
      axes.forEach(a => {
        grp.append('circle').attr('cx', song[a.key]*r*Math.cos(a.angle)).attr('cy', song[a.key]*r*Math.sin(a.angle))
          .attr('r', 5).attr('fill', farbe);
        grp.append('text')
          .attr('x', (r + 24) * Math.cos(a.angle)).attr('y', (r + 24) * Math.sin(a.angle) + 10)
          .attr('text-anchor', 'middle').attr('dominant-baseline', 'middle')
          .attr('font-size', '11px').attr('font-weight', 'bold').attr('fill', farbe)
          .text(a.key === 'popNorm' ? Math.round(ausgewaehlt.pop || 0).toString() : song[a.key].toFixed(2));
      });
    }
  }

  function schliesseModal() { ausgewaehlt = null; modalGenre = null; radarZeigeSong = radarZeigeGenre = radarZeigeAvg = true; }

  function gehePhase(p) {
    if (typeof document === 'undefined') return;
    if ((p === 4 || p === 5) && !tourGesehen) { tourStarten(); return; }
    if (p >= 1) { introSichtbar = true; extremeSichtbar = true; scrollGesperrt = false; }
    tick().then(() => { scrolleZuAbschnitt('sec-' + p); });
  }

  function stimmungText(v) {
    const s = t.stimmung;
    if (v >= valenceQuartil.ob)  return s[2];
    if (v >= valenceQuartil.unt) return s[1];
    return s[0];
  }
</script>

<div class="topbar">
  <button class="methodik-btn" onclick={() => showMethodik = true}>
    <span>ⓘ</span> {t.methodikBtn}
  </button>
</div>

<div class="story-hinweis" role="note">⚠️ {t.persistentNotice}</div>

{#if showMethodik}
  <div class="modal-overlay" onclick={() => showMethodik = false} role="presentation">
    <div class="modal modal-methodik" onclick={(e) => e.stopPropagation()} role="dialog" aria-modal="true">
      <button class="modal-close" onclick={() => showMethodik = false} aria-label={t.modalSchliessen}>×</button>
      <h3 class="modal-titel" style="color:#0c8a3a">{t.methodikTitle}</h3>
      <p class="modal-intro">{t.methodikIntro}</p>
      <div class="methodik-liste">
        {#each methodikGerendert as m}
          <div class="methodik-block">
            <h4>{m.h}</h4>
            <p>{m.p}</p>
          </div>
        {/each}
      </div>
    </div>
  </div>
{/if}

<nav class="fortschritt" aria-label="Fortschritt">
  <span class="np-play" aria-hidden="true">
    <span class="np-eq"><i></i><i></i><i></i><i></i></span>
  </span>
  <div class="np-meta">
    <div class="np-titel np-sr">{t.npTitel}</div>
    <div class="np-artist np-sr">Laura Riedl</div>
    <div class="np-track">
      {#each schritte as schritt, i}
        <button class="np-seg" class:done={schritt.done} class:aktiv={schritt.aktiv}
                style={schritt.fill > 0 ? `background: linear-gradient(to bottom, ${schritt.done ? '#178a40' : '#1DB954'} ${schritt.fill * 100}%, transparent ${schritt.fill * 100}%)` : ''}
                onclick={schritt.go} title={schritt.label} aria-label={schritt.label}></button>
      {/each}
      <span class="np-knob" style="top: calc({knobPct}% - 6px)"></span>
    </div>
  </div>
</nav>

{#if (tipPhase === 0 && introSichtbar && tipTop > 0) || (tipPhase >= 1 && tipPhase <= 3 && zeigeSeitenTip)}
  {#key tipPhase}
    <aside class="side-tip" style="top: {tipTop + 170}px" in:fly={{ x: 36, duration: 650, easing: cubicOut }} out:fly={{ x: 36, duration: 350, easing: cubicOut }}>
      <span class="side-tip-label">{tipPhase === 0 ? t.lblWusstest : t.lblFunFact}</span>
      <p class="side-tip-text">
        {#if tipPhase === 0}🎧 {t.playHintIntro}
        {:else if tipPhase === 1}{t.s1_hinweis}
        {:else if tipPhase === 2}{t.s2_hinweis}
        {:else if tipPhase === 3}{t.s3_takeaway}{/if}
      </p>
    </aside>
  {/key}
{/if}

<div class="story-scroll">

  <section id="sec-0" class="start story-abschnitt" data-phase="0">
    <div class="start-hero">
      <p class="kicker">{t.startKicker}</p>
      <h1>{@html t.startTitle}</h1>
      <p class="untertitel">{@html t.startSub}</p>
      <p class="start-lead">{t.startLead}</p>
      <button type="button" class="explore-cta" onclick={introStarten}>
        <span class="explore-cta-btn text-trigger tt-ohne-tip">{t.startCtaBtn}</span>
      </button>
    </div>

    <div id="sec-0-text" class="start-extra explore-wisch-rein" class:explore-versteckt={!introSichtbar}>
      <div class="start-extra-inner">
        <p class="intro-text fly fly-up">{t.introText1a}
          <span style="white-space:nowrap">{#if topSong?.trackId}<button type="button" class="play-btn" aria-label={t.playHint} onclick={() => spotifyAbspielen('intro', topSong.trackId)}>▶</button>{/if} <strong>"{topSong?.titel ?? 'Dance Monkey'}"</strong></span> {t.introText1}</p>
        <p class="intro-text fly fly-up fly-d2">{t.introText2a}</p>
        <p class="intro-text fly fly-up fly-d2">{t.introText2b}</p>
        <p class="intro-text intro-schluss fly fly-up fly-d2">{t.introText2c}</p>
        {#if sichtbareEmbeds.has('intro') && topSong?.trackId}
          <div class="story-note-embed"><div id="embed-intro" title="Spotify: {topSong.titel}"></div></div>
        {/if}
        {#if !geladen}<p class="laden reveal">{t.laden}</p>{/if}
        <div class="scroll-weiter" aria-hidden="true">
          <p class="scroll-weiter-text">{t.startCtaText}</p>
          <p class="scroll-cue">↓ {t.scrollCue}</p>
        </div>
      </div>
    </div>
  </section>

  <section id="sec-1" class="phase story-abschnitt" data-phase="1">
            <h2 class="fly fly-left">{t.s1_h2}</h2>
      <p class="lead fly fly-right fly-d2">{t.s1_l1}<button type="button" class="text-trigger" class:text-trigger-aktiv={s1Fokus === 'sad'} onclick={() => extremFokus('sad')}>{t.s1TrigSad}</button>{t.s1_l2}<button type="button" class="text-trigger" class:text-trigger-aktiv={s1Fokus === 'happy'} onclick={() => extremFokus('happy')}>{t.s1TrigHappy}</button>{t.s1_l3}</p>
      <p class="lead ohne-balken zentriert fly fly-right fly-d2">{t.s1_schluss}</p>

      <div class="hilfe-zeile reveal">
        <button type="button" class="hilfe-chip" aria-expanded={hilfeOffen === 'extreme'} onclick={() => hilfeOffen = hilfeOffen === 'extreme' ? null : 'extreme'}>ℹ️ {t.hilfeChip}</button>
      </div>
      {#if hilfeOffen === 'extreme'}<div class="hilfe-panel hilfe-panel-inline">👆 {t.klickHintExtreme}</div>{/if}
      <div class="chart-card reveal chart-beeswarm-wrap">
        <canvas id="canvas-beeswarm" width="760" height="380"></canvas>
        <svg id="svg-beeswarm" class="svg-beeswarm-overlay" viewBox="0 0 760 380" preserveAspectRatio="xMidYMid meet"></svg>
      </div>
      <div class="mood-legende reveal">
        <div class="legende-item"><span class="punkt" style="background:#4f46e5"></span><span>{t.s1_sad}</span></div>
        <div class="legende-item"><span class="punkt" style="background:{SPOTIFY}"></span><span>{t.s1_happy}</span></div>
      </div>
      {#if extremAusgewaehlt}
        <div class="extrem-popup">
          <button class="extrem-popup-close" onclick={() => extremAusgewaehlt = null} aria-label="×">×</button>
          <p class="extrem-popup-titel"><strong>{extremAusgewaehlt.titel}</strong></p>
          <p class="extrem-popup-artist">{extremAusgewaehlt.artist}</p>
          <p class="extrem-popup-valence">Mood: <strong>{extremAusgewaehlt.valence.toFixed(2)}</strong></p>
          {#if extremAusgewaehlt.trackId}
            <div class="spotify-embed-wrap" style="margin:8px 0 0">
              <iframe title="Spotify: {extremAusgewaehlt.titel}"
                src="https://open.spotify.com/embed/track/{extremAusgewaehlt.trackId}?utm_source=generator"
                width="100%" height="80" frameborder="0"
                allow="encrypted-media; clipboard-write" loading="lazy"></iframe>
            </div>
          {/if}
        </div>
      {/if}
      <p class="story-note ohne-balken reveal">
        {#if happySongs[0]?.trackId}<button type="button" class="play-btn" aria-label={t.playHint} onclick={() => spotifyAbspielen('happy', happySongs[0].trackId)}>▶</button>{/if}
        <strong>{happySongs[0]?.titel}</strong> {t.storyExtreme1}
        {#if sadSongs[0]?.trackId}<button type="button" class="play-btn" aria-label={t.playHint} onclick={() => spotifyAbspielen('sad', sadSongs[0].trackId)}>▶</button>{/if}
        <strong>{sadSongs[0]?.titel}</strong> {t.storyExtreme2}
      </p>
      {#if sichtbareEmbeds.has('happy') && happySongs[0]?.trackId}
        <div class="story-note-embed reveal"><div id="embed-happy" title="Spotify: {happySongs[0].titel}"></div></div>
      {/if}
      {#if sichtbareEmbeds.has('sad') && sadSongs[0]?.trackId}
        <div class="story-note-embed reveal"><div id="embed-sad" title="Spotify: {sadSongs[0].titel}"></div></div>
      {/if}
    </section>

    <section id="sec-2" class="phase story-abschnitt" data-phase="2">
      <h2 class="reveal">{t.s2_h2}</h2>
      <p class="lead reveal">{t.s2_lead}</p>

      <div class="hilfe-zeile reveal">
        <button type="button" class="hilfe-chip" aria-expanded={hilfeOffen === 'genre'} onclick={() => hilfeOffen = hilfeOffen === 'genre' ? null : 'genre'}>ℹ️ {t.hilfeChip}</button>
      </div>
      {#if hilfeOffen === 'genre'}<div class="hilfe-panel hilfe-panel-inline">{t.s2_how}</div>{/if}
      <div class="chart-card reveal">
        <svg id="svg-genre" class="svg-fluid" viewBox="0 0 620 320" preserveAspectRatio="xMidYMid meet"></svg>
      </div>

      <div class="genre-legende reveal">
        {#each genreDaten as g}
          <div class="legende-item">
            <span class="punkt" style="background:{g.farbe}"></span>
            <span>{t.genreLabels[g.genre]}</span>
          </div>
        {/each}
      </div>
      <p class="lead ohne-balken reveal">{t.s2_d1}<button type="button" class="text-trigger" class:text-trigger-aktiv={deepFokus === 'avg'} onclick={() => deepFokus = deepFokus === 'avg' ? null : 'avg'}>{t.s2_trigAvg}</button>{t.s2_d2a}<br>{t.s2_d2b}<button type="button" class="text-trigger" class:text-trigger-aktiv={deepFokus === 'latin'} onclick={() => deepFokus = deepFokus === 'latin' ? null : 'latin'}>{t.s2_trigLatin}</button>{t.s2_d3a}<br>{t.s2_d3b}<button type="button" class="text-trigger" class:text-trigger-aktiv={deepFokus === 'edm'} onclick={() => deepFokus = deepFokus === 'edm' ? null : 'edm'}>{t.s2_trigEdm}</button>{t.s2_d4}</p>
    </section>

    <section id="sec-3" class="phase story-abschnitt" data-phase="3">
            <h2 class="reveal">{t.s3_h2}</h2>
      <p class="lead reveal">{t.s3_g1}<span class="glossar">energy<span class="glossar-karte"><span class="glossar-wort">en·er·gy</span><span class="glossar-art">{t.artSkala1}</span><span class="glossar-def">{t.defEnergie}</span></span></span>, <span class="glossar">danceability<span class="glossar-karte"><span class="glossar-wort">dance·abil·i·ty</span><span class="glossar-art">{t.artSkala1}</span><span class="glossar-def">{t.defTanz}</span></span></span> and <span class="glossar">popularity<span class="glossar-karte"><span class="glossar-wort">pop·u·lar·i·ty</span><span class="glossar-art">{t.artSkala100}</span><span class="glossar-def">{t.defPop}</span></span></span>{t.s3_g2}</p>

      <div class="hilfe-zeile reveal">
        <button type="button" class="hilfe-chip" aria-expanded={hilfeOffen === 'mood'} onclick={() => hilfeOffen = hilfeOffen === 'mood' ? null : 'mood'}>ℹ️ {t.hilfeChip}</button>
      </div>
      {#if hilfeOffen === 'mood'}<div class="hilfe-panel hilfe-panel-inline">{t.s3_radarHilfe}</div>{/if}

      <div class="mood-radare reveal">
        {#each moodBeispiele as mb, i}
          <div class="mood-radar-card">
            <button type="button" class="karte-zoom" aria-label="Zoom" onclick={() => zoomKarte = i}>🔍</button>
            <div class="mood-radar-kopf">
              <p class="mood-radar-rolle" style="color:{mb.farbe}">{mb.rolle}</p>
              <p class="mood-radar-titel">{mb.song.titel}</p>
              <p class="mood-radar-artist">{mb.song.artist}</p>
            </div>
            <svg id="svg-mood-{i}" class="svg-fluid" viewBox="0 0 260 248" preserveAspectRatio="xMidYMid meet"></svg>
            <div class="mood-radar-fuss">
              {#if mb.song.trackId}<button type="button" class="play-btn" aria-label={t.playHint} onclick={() => spotifyAbspielen('mb' + i, mb.song.trackId)}>▶</button>{/if}
              <span class="td-meta">{mb.regel}</span>
            </div>
            {#if sichtbareEmbeds.has('mb' + i) && mb.song.trackId}
              <div class="story-note-embed"><div id="embed-mb{i}" title="Spotify: {mb.song.titel}"></div></div>
            {/if}
          </div>
        {/each}
      </div>
      {#if topHitsNeutral}
        <div class="mood-legende reveal">
        <div class="legende-item"><span class="punkt" style="background:#4f46e5"></span><span>{t.moodMela}</span></div>
        <div class="legende-item"><span class="punkt" style="background:{SPOTIFY}"></span><span>{t.moodFroh}</span></div>
        <div class="legende-item"><span class="punkt-linie"></span><span>{t.moodAvg}</span></div>
      </div>

      {#if zoomKarte !== null && moodBeispiele[zoomKarte]}
        {@const mz = moodBeispiele[zoomKarte]}
        <div class="modal-overlay" onclick={() => zoomKarte = null} role="presentation">
          <div class="modal mood-zoom" onclick={(ev) => ev.stopPropagation()} role="dialog" aria-modal="true">
            <button class="modal-close" onclick={() => zoomKarte = null} aria-label="×">×</button>
            <p class="mood-radar-rolle" style="color:{mz.farbe}">{mz.rolle}</p>
            <p class="mood-radar-titel" style="min-height:auto">{mz.song.titel}</p>
            <p class="mood-radar-artist">{mz.song.artist}</p>
            <svg id="svg-mood-zoom" class="svg-fluid" viewBox="0 0 260 248" preserveAspectRatio="xMidYMid meet"></svg>
            <div class="mood-radar-fuss">
              {#if mz.song.trackId}<button type="button" class="play-btn" aria-label={t.playHint} onclick={() => spotifyAbspielen('zoom', mz.song.trackId)}>▶</button>{/if}
              <span class="td-meta">{mz.regel}</span>
            </div>
            {#if sichtbareEmbeds.has('zoom') && mz.song.trackId}
              <div class="story-note-embed"><div id="embed-zoom" title="Spotify: {mz.song.titel}"></div></div>
            {/if}
          </div>
        </div>
      {/if}

      <p class="story-note ohne-balken reveal">🏆 {t.storyNeutral1} {t.storyNeutral2}</p>
        <button type="button" class="top5-btn reveal" onclick={() => zeigeTop5 = !zeigeTop5} aria-expanded={zeigeTop5}>{zeigeTop5 ? t.top5Zu : t.top5Auf}</button>
        {#if zeigeTop5}
          <div class="top5-liste" transition:fly={{ y: 12, duration: 350 }}>
            {#each top5 as s, i}
              <div class="top5-zeile">
                <span class="top5-rang">{i + 1}</span>
                {#if s.trackId}<button type="button" class="play-btn" aria-label={t.playHint} onclick={() => spotifyAbspielen('top5-' + i, s.trackId)}>▶</button>{/if}
                <span class="top5-titel"><strong>{s.titel}</strong> · {s.artist}</span>
                <span class="top5-wert">{t.lblStimmung} {s.valence.toFixed(2)} · {t.lblPop} {Math.round(s.pop)}</span>
              </div>
              {#if sichtbareEmbeds.has('top5-' + i) && s.trackId}
                <div class="story-note-embed"><div id="embed-top5-{i}" title="Spotify: {s.titel}"></div></div>
              {/if}
            {/each}
          </div>
        {/if}
      {/if}

      <button type="button" class="explore-cta reveal" onclick={tourStarten}>
        <span class="explore-cta-text">{t.exploreCtaText}</span>
        <span class="explore-cta-btn">{t.exploreCtaBtn}</span>
      </button>
    </section>

    <section id="sec-4" class="phase story-abschnitt explore-wisch-rein" class:explore-versteckt={!tourGesehen} data-phase="4">
      {#if tourSchritt === 1 && phase === 4}
        <aside class="guide-tip guide-tip-fix" transition:fly={{ x: -36, duration: 650, easing: cubicOut }}>
          <span class="guide-tip-label">🎧 {t.tourTitel}</span>
          <p>{t.tourIntro}</p>
          <button class="btn-haupt klein" onclick={() => tourSchritt = 2}>{t.tourLos}</button>
        </aside>
      {/if}

      <h2 class="fly fly-up">{t.s4_h2}</h2>
      <p class="lead fly fly-up fly-d2">{t.s4_lead}</p>

      <div class="suche reveal" class:tour-ziel={tourSchritt === 4}>
        {#if tourSchritt === 4 && phase === 4}
          <aside class="guide-tip guide-tip-fix" transition:fly={{ x: -36, duration: 650, easing: cubicOut }}>
            <span class="guide-tip-label">👉 {t.tourSearchLabel}</span>
            <p>{t.tour4}</p>
            <button class="btn-haupt klein" onclick={() => tourSchritt = 0}>{t.tourFertig}</button>
          </aside>
        {/if}
        <input type="text" placeholder={t.searchPh} bind:value={suchbegriff} />
        {#if suchergebnisse.length > 0}
          <div class="such-liste">
            <div class="such-anzahl">{suchergebnisse.length} {suchergebnisse.length !== 1 ? t.ergebnisse : t.ergebnis}</div>
            {#each suchergebnisse as s}
              <button class="such-item" onclick={() => { ausgewaehlt = s; modalGenre = s.genre; suchbegriff = ''; radarZeigeSong = radarZeigeGenre = radarZeigeAvg = true; }}>
                <strong>{s.titel}</strong> <span>{s.artist}</span>
                <span class="such-stimmung">{stimmungText(s.valence)}</span>
              </button>
            {/each}
          </div>
        {/if}
      </div>

      <div class="filter-zeile">
        <div class="filter-gruppe" class:tour-ziel={tourSchritt === 2}>
          {#if tourSchritt === 2 && phase === 4}
            <aside class="guide-tip guide-tip-fix" transition:fly={{ x: -36, duration: 650, easing: cubicOut }}>
              <span class="guide-tip-label">👉 {t.genreFilter}</span>
              <p>{t.tour2}</p>
              <button class="btn-haupt klein" onclick={() => tourSchritt = 3}>{t.tourOk}</button>
            </aside>
          {/if}
          <span class="filter-label">{t.genreFilter}</span>
          <button class="filter-btn" class:aktiv={genre === 'alle'} onclick={() => { genre = 'alle'; }}>{t.genreAlle}</button>
          {#each genres as g}
            <button class="filter-btn" class:aktiv={genre === g} onclick={() => { genre = g; }}>{t.genreLabels[g]}</button>
          {/each}
        </div>
        <div class="filter-gruppe" class:tour-ziel={tourSchritt === 3}>
          {#if tourSchritt === 3 && phase === 4}
            <aside class="guide-tip guide-tip-fix" transition:fly={{ x: -36, duration: 650, easing: cubicOut }}>
              <span class="guide-tip-label">👉 {t.stimmungLabel}</span>
              <p>{t.tour3}</p>
              <button class="btn-haupt klein" onclick={() => tourSchritt = 4}>{t.tourOk}</button>
            </aside>
          {/if}
          <span class="filter-label">{t.stimmungLabel}</span>
          {#each [{ key: 'alle', label: t.filterAlle }, { key: 'happy', label: t.filterHappy }, { key: 'sad', label: t.filterSad }] as f}
            <button class="filter-btn" class:aktiv={stimmungFilter === f.key} onclick={() => { stimmungFilter = f.key; }}>{f.label}</button>
          {/each}
        </div>
      </div>

      {#if exploreGefiltert}
        <p class="explore-hinweis">{t.exploreTop50}</p>
      {/if}

      <div class="hilfe-zeile reveal">
        <button type="button" class="hilfe-chip" aria-expanded={hilfeOffen === 'explore'} onclick={() => hilfeOffen = hilfeOffen === 'explore' ? null : 'explore'}>ℹ️ {t.hilfeChip}</button>
      </div>
      {#if hilfeOffen === 'explore'}<div class="hilfe-panel hilfe-panel-inline">👆 {t.s4_tipp}</div>{/if}
      <div class="chart-card reveal">
        {#if exploreGefiltert}
          <div class="expl-head"><svg id="svg-expl-head" class="svg-fluid" viewBox="0 0 620 22" preserveAspectRatio="xMidYMid meet"></svg></div>
        {/if}
        <div class="expl-scroll" class:scrollbar={exploreGefiltert}>
          <svg id="svg-exploration" class="svg-fluid" viewBox="0 0 620 300" preserveAspectRatio="xMidYMid meet"></svg>
        </div>
        {#if exploreGefiltert}
          <div class="expl-axis-wrap"><svg id="svg-expl-axis" class="svg-fluid" viewBox="0 0 620 64" preserveAspectRatio="xMidYMid meet"></svg></div>
        {/if}
      </div>

      {#if genre === 'alle'}
        <div class="scatter-legende">
          {#each genres as g}
            <div class="legende-item"><span class="punkt" style="background:{genreFarben[g]}"></span><span>{t.genreLabels[g]}</span></div>
          {/each}
        </div>
      {:else}
        <div class="scatter-legende">
          <div class="legende-item"><span class="punkt" style="background:{genreFarben[genre]}"></span><span>{t.genreLabels[genre]}</span></div>
        </div>
      {/if}
    </section>

    <section id="sec-5" class="ende-final story-abschnitt" class:explore-versteckt={!tourGesehen} data-phase="5">
      <p class="abschluss-note reveal">{t.abschluss}</p>
      <div class="disclaimer reveal" role="note">
        <span class="disclaimer-icon" aria-hidden="true">⚠️</span>
        <div><strong>{t.disclaimerTitel}</strong><p>{t.disclaimerText}</p></div>
      </div>

      <div class="thanks reveal">
        <div class="thanks-inner">
          <h3>{t.thanksTitle}</h3>
          <p>{t.thanksData} <a href="https://www.kaggle.com/datasets/joebeachcapital/30000-spotify-songs" target="_blank">{t.thanksDataName}</a> {t.thanksData2.replace('{n}', alleSongs.length.toLocaleString('en-US'))}</p>
          <div class="thanks-person">
            <p><strong>Laura Riedl</strong><br><a href="mailto:laura.riedl@stud.uni-bamberg.de">laura.riedl@stud.uni-bamberg.de</a></p>
            <p><strong>{t.thanksBetreuerin}</strong> Leah Mühlöder<br><strong>{t.thanksProf}</strong> Prof. Dr. Fabian Beck<br><a href="https://www.uni-bamberg.de/vis/" target="_blank">Lehrstuhl für Informationsvisualisierung<br>Fakultät Wirtschaftsinformatik und Angewandte Informatik</a></p>
          </div>
          <p class="thanks-meta">{t.thanksMeta}</p>
        </div>
      </div>
    </section>

  </div>

{#if ausgewaehlt && modalDaten}
  <div class="modal-overlay" onclick={schliesseModal} role="presentation">
    <div class="modal" onclick={(e) => e.stopPropagation()} role="dialog" aria-modal="true">
      <button class="modal-close" onclick={schliesseModal} aria-label={t.modalSchliessen}>×</button>
      <div class="detail-kopf"><span class="stimmungs-emoji">{stimmungText(ausgewaehlt.valence)}</span><h3 class="modal-titel">{ausgewaehlt.titel}</h3></div>
      <p class="modal-sub"><strong>{t.detailArtist}</strong> {ausgewaehlt.artist} &nbsp;|&nbsp; <strong>{t.detailAlbum}</strong> {ausgewaehlt.album}</p>
      <div class="detail-werte">
        {#each [
          { label: t.lblStimmung, wert: ausgewaehlt.valence.toFixed(2), wort: 'va·lence', art: t.artSkala1, def: t.defStimmung },
          { label: t.lblEnergie,  wert: ausgewaehlt.energy.toFixed(2),  wort: 'en·er·gy', art: t.artSkala1, def: t.defEnergie },
          { label: t.lblTanz,     wert: ausgewaehlt.dance.toFixed(2),   wort: 'dance·abil·i·ty', art: t.artSkala1, def: t.defTanz },
          { label: t.lblPop,      wert: ausgewaehlt.pop,                wort: 'pop·u·lar·i·ty', art: t.artSkala100, def: t.defPop }
        ] as k}
          <div class="detail-wert"><span class="wert-zahl">{k.wert}</span><span class="wert-label">{k.label} ⓘ</span><span class="glossar-karte glossar-karte-unten"><span class="glossar-wort">{k.wort}</span><span class="glossar-art">{k.art}</span><span class="glossar-def">{k.def}</span></span></div>
        {/each}
      </div>
      {#if ausgewaehlt.trackId}
        <div class="spotify-embed-wrap">
          <iframe
            title="Spotify: {ausgewaehlt.titel}"
            src="https://open.spotify.com/embed/track/{ausgewaehlt.trackId}?utm_source=generator"
            width="100%" height="80" frameborder="0"
            allow="encrypted-media; clipboard-write"
            loading="lazy"></iframe>
        </div>
      {/if}

      <div class="modal-radar-wrap">
        <p class="modal-radar-titel">
          {t.modalGenreRadar} <span style="color:{modalDaten.farbe}">{ausgewaehlt.titel}</span>
          <span class="radar-info">ⓘ<span class="tooltip-text radar-tooltip">{t.modalRadarHilfe}</span></span>
        </p>
        <div class="radar-legende">
          <button type="button" class="rl-item" class:rl-aus={!radarZeigeSong} onclick={() => radarZeigeSong = !radarZeigeSong} aria-pressed={radarZeigeSong}>
            <span class="rl-swatch" style="border-color:{modalDaten.farbe}"></span>{t.modalRadarLegSong}
          </button>
          <button type="button" class="rl-item" class:rl-aus={!radarZeigeGenre} onclick={() => radarZeigeGenre = !radarZeigeGenre} aria-pressed={radarZeigeGenre}>
            <span class="rl-swatch rl-dash-genre" style="border-color:{genreAvgFarbe(modalDaten.farbe)}"></span>{t.modalRadarLegGenre} ({modalDaten.label})
          </button>
          <button type="button" class="rl-item" class:rl-aus={!radarZeigeAvg} onclick={() => radarZeigeAvg = !radarZeigeAvg} aria-pressed={radarZeigeAvg}>
            <span class="rl-swatch rl-dash-avg" style="border-color:#111"></span>{t.modalRadarLegAvg}
          </button>
        </div>
        <p class="radar-legende-hint">{t.modalRadarLegHint}</p>
        <div class="modal-body">
          <svg id="svg-radar-modal" width="360" height="300"></svg>
          <div class="modal-werte">
            <div class="mw"><span class="mw-zahl" style="color:{modalDaten.farbe}">{ausgewaehlt.valence.toFixed(2)}</span><span class="mw-label">{t.radarStimmung}</span></div>
            <div class="mw"><span class="mw-zahl" style="color:{modalDaten.farbe}">{ausgewaehlt.energy.toFixed(2)}</span><span class="mw-label">{t.radarEnergie}</span></div>
            <div class="mw"><span class="mw-zahl" style="color:{modalDaten.farbe}">{ausgewaehlt.dance.toFixed(2)}</span><span class="mw-label">{t.radarTanz}</span></div>
            <div class="mw"><span class="mw-zahl" style="color:{modalDaten.farbe}">{Math.round(ausgewaehlt.pop)}</span><span class="mw-label">{t.radarPop}</span></div>
          </div>
        </div>
      </div>
    </div>
  </div>
{/if}

<style>

  :global(html) {

    scroll-behavior: smooth;

    overflow-anchor: none;
  }
  @media (prefers-reduced-motion: reduce) {
    :global(html) { scroll-behavior: auto; }
  }
  :global(body) {
    font-family: 'Georgia', 'Helvetica Neue', Arial, sans-serif;
    background: radial-gradient(circle at 50% 0%, #1DB954 0%, #0c8a3a 45%, #064e25 100%);
    background-attachment: fixed;
    min-height: 100vh; margin: 0; color: #fff;
    padding-left: 64px;
  }
  @media (max-width: 760px) {
    :global(body) { padding-left: 46px; }
  }
  :global(*) { box-sizing: border-box; }

  .topbar {
    position: fixed; top: 14px; right: 18px; z-index: 1000;
    display: flex; align-items: center; gap: 8px;
  }
  .methodik-btn {
    display: inline-flex; align-items: center; gap: 5px;
    background: rgba(255,255,255,0.95); backdrop-filter: blur(8px);
    border: 1px solid #e0e0e0; border-radius: 22px;
    padding: 7px 14px; cursor: pointer; font-size: 0.85rem; font-weight: 600; color: #aaa;
    font-family: 'Helvetica Neue', Arial, sans-serif; box-shadow: 0 3px 12px rgba(0,0,0,0.12);
  }
  .methodik-btn:hover { box-shadow: 0 4px 16px rgba(0,0,0,0.18); background: #fff; color: #777; }

  .story-hinweis {
    position: fixed; top: 14px; left: 82px; z-index: 1000; max-width: 300px;
    display: flex; align-items: center; gap: 6px;
    background: rgba(255,255,255,0.95); backdrop-filter: blur(8px);
    border: 1px solid #e0e0e0; border-radius: 22px;
    padding: 7px 14px; font-size: 0.8rem; font-weight: 600; color: #aaa;
    font-family: 'Helvetica Neue', Arial, sans-serif; box-shadow: 0 3px 12px rgba(0,0,0,0.12);
  }
  @media (max-width: 760px) {
    .story-hinweis { left: 56px; max-width: 200px; font-size: 0.72rem; padding: 6px 10px; }
  }

  .side-tip {

    position: absolute; right: 24px;
    width: 240px; z-index: 400;
    background: #fff; border-left: 3px solid #1DB954; border-radius: 0 10px 10px 0;
    padding: 14px 16px; box-shadow: 0 4px 18px rgba(0,0,0,0.1);
  }
  .side-tip-label { display: block; font-size: 0.72rem; letter-spacing: 1px; text-transform: uppercase; color: #1DB954; font-weight: 700; margin-bottom: 6px; font-family: 'Helvetica Neue', Arial, sans-serif; }
  .side-tip-text { font-size: 0.88rem; line-height: 1.55; color: #1a1a2e; margin: 0; }
  @media (max-width: 1100px) {
    .side-tip { display: none; }
  }

  .story-scroll { display: block; }

  .fortschritt {
    display: flex; flex-direction: column; align-items: center; gap: 14px;
    padding: 18px 0 22px; width: 64px;
    background: #181818; border-right: 1px solid #000;
    position: fixed; top: 0; left: 0; bottom: 0; z-index: 500;
    font-family: 'Helvetica Neue', Arial, sans-serif;
    box-shadow: 2px 0 10px rgba(0,0,0,0.25);
  }

  .np-play {
    display: flex; align-items: center; justify-content: center;
    width: 36px; height: 36px; border-radius: 6px; background: #1DB954; flex-shrink: 0;
    box-shadow: 0 2px 8px rgba(0,0,0,0.4);
  }
  .np-eq { display: flex; align-items: flex-end; gap: 2px; height: 16px; }
  .np-eq i { width: 3px; background: #04331a; border-radius: 1px; }
  .np-eq i:nth-child(1) { height: 40%; }
  .np-eq i:nth-child(2) { height: 100%; }
  .np-eq i:nth-child(3) { height: 60%; }
  .np-eq i:nth-child(4) { height: 85%; }

  .np-meta { flex: 1; min-width: 0; display: flex; flex-direction: column; align-items: center; gap: 2px; width: 100%; }
  .np-sr { position: absolute; width: 1px; height: 1px; overflow: hidden; clip: rect(0 0 0 0); white-space: nowrap; }

  .np-track {
    position: relative; display: flex; flex-direction: column; width: 5px; flex: 1; margin: 6px 0;
    background: #4d4d4d; border-radius: 3px;
  }
  .np-track:hover { width: 6px; }
  .np-seg {
    position: relative; z-index: 2; flex: 1; min-height: 0; width: 100%;
    border: none; background: transparent; cursor: pointer; padding: 0;
  }
  .np-seg:first-of-type { border-radius: 3px 3px 0 0; }
  .np-seg:last-of-type { border-radius: 0 0 3px 3px; }
  .np-track:hover .np-knob { opacity: 1; }
  .np-knob {
    position: absolute; left: 50%; width: 12px; height: 12px; border-radius: 50%;
    background: #fff; transform: translateX(-50%); z-index: 3;
    box-shadow: 0 1px 4px rgba(0,0,0,0.5); transition: top 0.35s ease, opacity 0.15s;
    pointer-events: none; opacity: 0;
  }
  @media (max-width: 760px) {
    .fortschritt { width: 46px; padding: 12px 0 16px; gap: 10px; }
    .np-play { width: 28px; height: 28px; }
  }

  .start {
    text-align: center; padding: 0 40px 70px;
    color: #fff;
  }

  .start-hero {
    display: flex; flex-direction: column; align-items: center; justify-content: center;
    min-height: 100vh;
  }

  .start-extra { min-height: 100vh; display: flex; align-items: center; }
  .start-extra-inner { max-width: 580px; margin: 0 auto; text-align: left; }
  .kicker { font-size: 0.85rem; letter-spacing: 2px; margin-bottom: 16px; font-family: 'Helvetica Neue', Arial, sans-serif; }
  .start .kicker { color: rgba(255,255,255,0.85); }
  .start h1 { font-size: 3.4rem; line-height: 1.12; color: #fff; margin-bottom: 16px; }
  .untertitel { font-size: 1.25rem; color: rgba(255,255,255,0.92); margin-bottom: 22px; line-height: 1.5; }
  .start-lead { max-width: 480px; color: rgba(255,255,255,0.8); line-height: 1.7; margin-bottom: 36px; }
  .intro-text { max-width: 580px; color: rgba(255,255,255,0.95); line-height: 1.75; font-size: 1.08rem; margin-bottom: 18px; }
  .intro-text:first-of-type { font-size: 1.08rem; font-weight: 600; }
  .laden { color: rgba(255,255,255,0.85); }

  @keyframes hero-fly-down { from { opacity: 0; transform: translateY(-26px); } to { opacity: 1; transform: translateY(0); } }
  @keyframes hero-fly-up   { from { opacity: 0; transform: translateY(50px); }  to { opacity: 1; transform: translateY(0); } }
  @keyframes hero-fade-in  { from { opacity: 0; } to { opacity: 1; } }
  .start-hero .kicker     { animation: hero-fly-down 0.7s ease-out 0.05s both; }
  .start-hero h1          { animation: hero-fly-up 0.85s ease-out 0.25s both; }
  .start-hero .untertitel { animation: hero-fly-up 0.85s ease-out 0.45s both; }
  .start-hero .start-lead { animation: hero-fly-up 0.85s ease-out 0.65s both; }
  .start-hero .explore-cta { animation: hero-fade-in 0.6s ease-out 1.15s both; }
  @media (prefers-reduced-motion: reduce) {
    .start-hero .kicker, .start-hero h1, .start-hero .untertitel,
    .start-hero .start-lead, .start-hero .explore-cta {
      animation: none; opacity: 1; transform: none;
    }
  }
  .scroll-cue { margin-top: 6px; color: rgba(255,255,255,0.8); font-size: 0.95rem; letter-spacing: 0.5px; animation: scroll-cue-bounce 2s ease-in-out infinite; }
  @keyframes scroll-cue-bounce { 0%, 100% { transform: translateY(0); } 50% { transform: translateY(6px); } }

  .scroll-weiter { text-align: center; margin-top: 30px; }
  .scroll-weiter-text { color: rgba(255,255,255,0.85); font-size: 1rem; margin-bottom: 4px; }

  .phase { max-width: 760px; margin: 0 auto; padding: 48px 20px 64px; }
  .phase h2 { font-size: 2.1rem; color: #fff; margin-bottom: 14px; line-height: 1.2; }
  .lead { font-size: 1.08rem; line-height: 1.75; color: #fff; margin-bottom: 18px; border-left: 3px solid rgba(255,255,255,0.7); padding-left: 16px; }
  .story-note { font-size: 1.08rem; line-height: 1.75; color: #fff; margin: 14px 0; border-left: 3px solid rgba(255,255,255,0.7); padding-left: 14px; }

  .chart-card { position: relative; background: #fff; border: 1px solid #eee; border-radius: 14px; padding: 18px; box-shadow: 0 4px 18px rgba(0,0,0,0.06); overflow-x: auto; }

  .svg-fluid { width: 100%; height: auto; display: block; }

  .chart-beeswarm-wrap { position: relative; }
  #canvas-beeswarm { width: 100%; height: auto; display: block; }
  .svg-beeswarm-overlay { position: absolute; top: 0; left: 0; width: 100%; height: 100%; }
  .extrem-popup {
    background: rgba(255,255,255,0.15); backdrop-filter: blur(8px);
    border: 1px solid rgba(255,255,255,0.3); border-radius: 12px;
    padding: 14px 44px 14px 18px; margin: 12px 0; position: relative; color: #fff;
  }
  .extrem-popup-close {
    position: absolute; top: 8px; right: 10px;
    background: rgba(255,255,255,0.18); border: none; width: 26px; height: 26px;
    border-radius: 50%; cursor: pointer; color: #fff; font-size: 1.05rem; line-height: 1;
  }
  .extrem-popup-close:hover { background: rgba(255,255,255,0.32); }
  .extrem-popup-titel { margin: 0 0 3px; font-size: 1rem; }
  .extrem-popup-artist { margin: 0 0 3px; font-size: 0.85rem; color: rgba(255,255,255,0.72); }
  .extrem-popup-valence { margin: 0 0 10px; font-size: 0.88rem; }

  .expl-scroll.scrollbar { max-height: 480px; overflow-y: auto; }

  .expl-axis-wrap { background: #fff; border-top: 1px solid #e2e2e2; padding-top: 4px; margin-top: 2px; }

  .expl-head { background: #fff; border-bottom: 1px solid #e2e2e2; padding-bottom: 4px; margin-bottom: 2px; }
  .explore-hinweis { font-size: 0.9rem; line-height: 1.5; color: #fff; margin: 0 0 12px; border-left: 4px solid rgba(255,255,255,0.7); padding-left: 12px; }

  .btn-haupt { background: #1DB954; color: white; border: none; padding: 12px 28px; border-radius: 24px; cursor: pointer; font-size: 1rem; font-family: 'Helvetica Neue', Arial, sans-serif; font-weight: 600; box-shadow: 0 4px 14px rgba(29,185,84,0.35); }
  .btn-haupt:hover { background: #18a44a; transform: translateY(-1px); }
  .klein { padding: 6px 14px; font-size: 0.85rem; margin-top: 12px; }

  .explore-cta {
    display: flex; flex-direction: column; align-items: center; gap: 4px;
    width: 100%; margin: 36px 0 6px; padding: 6px;
    background: none; border: none; cursor: pointer; font-family: inherit; text-align: center;
  }
  .explore-cta-text { font-size: 0.95rem; color: rgba(255,255,255,0.8); }
  .explore-cta-btn {
    font-size: 1.02rem; font-weight: 700; color: #fff;
    animation: scroll-cue-bounce 2s ease-in-out infinite;
  }
  .explore-cta:hover .explore-cta-btn { color: #baf7d0; }
  @media (prefers-reduced-motion: reduce) {
    .explore-cta-btn { animation: none; }
  }

  .explore-versteckt { display: none; }

  @keyframes explore-wisch-rein {
    from { transform: translateY(140px); opacity: 0; }
    to   { transform: translateY(0); opacity: 1; }
  }
  .explore-wisch-rein { animation: explore-wisch-rein 0.85s cubic-bezier(0.22, 0.61, 0.36, 1); }
  @media (prefers-reduced-motion: reduce) {
    .explore-wisch-rein { animation: none; }
  }

  .genre-legende, .scatter-legende { display: flex; gap: 14px; flex-wrap: wrap; margin-top: 16px; }
  .mood-legende { display: flex; gap: 22px; flex-wrap: wrap; justify-content: center; margin: 6px 0 14px; }

  .td-meta { display: block; font-size: 0.8rem; color: #888; margin-top: 1px; }
  .legende-item { display: flex; align-items: center; gap: 6px; font-size: 13px; color: rgba(255,255,255,0.92); }
  .punkt { display: inline-block; width: 12px; height: 12px; border-radius: 50%; flex-shrink: 0; box-shadow: 0 0 0 2px rgba(255,255,255,0.9), 0 1px 3px rgba(0,0,0,0.35); }
  .punkt-linie { display: inline-block; width: 16px; height: 0; border-top: 2px dashed #111; flex-shrink: 0; }

  .suche { position: relative; margin-bottom: 20px; z-index: 200; }
  .suche input { width: 100%; padding: 12px 16px; border: 2px solid #e5e5f5; border-radius: 10px; font-size: 1rem; background: white; outline: none; box-shadow: 0 2px 8px rgba(0,0,0,0.06); font-family: inherit; }
  .suche input:focus { border-color: #1DB954; }
  .such-liste { position: absolute; top: 100%; left: 0; right: 0; background: white; border: 1px solid #e5e5f5; border-radius: 10px; margin-top: 4px; box-shadow: 0 8px 24px rgba(0,0,0,0.14); z-index: 200; max-height: 320px; overflow-y: auto; }
  .such-anzahl { padding: 6px 16px; font-size: 0.75rem; color: #aaa; background: #fafafa; border-bottom: 1px solid #f0f0f0; }
  .such-item { display: flex; align-items: center; gap: 10px; padding: 10px 16px; width: 100%; text-align: left; border: none; background: none; cursor: pointer; font-size: 0.9rem; border-bottom: 1px solid #f5f5f5; font-family: inherit; }
  .such-item:hover { background: #ecfdf3; }
  .such-item strong { flex: 1; color: #1a1a2e; }
  .such-item span { color: #888; font-size: 0.8rem; }
  .such-stimmung { color: #0c8a3a !important; }

  .filter-zeile { display: flex; flex-direction: column; gap: 10px; margin-bottom: 20px; }
  .filter-gruppe { display: flex; align-items: center; gap: 8px; flex-wrap: wrap; position: relative; }

  .filter-label { font-size: 0.85rem; color: rgba(255,255,255,0.8); min-width: 64px; }
  .filter-btn { background: white; border: 1.5px solid #e5e5e5; color: #555; padding: 6px 14px; border-radius: 20px; cursor: pointer; font-size: 0.85rem; font-family: inherit; transition: background 0.15s; }
  .filter-btn:hover { border-color: #aaa; background: #f5f5f5; }
  .filter-btn.aktiv { background: #1DB954; color: white; border-color: #1DB954; }

  .detail-kopf { display: flex; align-items: center; gap: 12px; margin-bottom: 8px; }
  .stimmungs-emoji { font-size: 1.5rem; }
  .detail-werte { display: flex; gap: 12px; flex-wrap: wrap; }
  .detail-wert { background: #f3fbf6; border-radius: 8px; padding: 10px 16px; text-align: center; flex: 1; min-width: 80px; position: relative; cursor: default; }
  .detail-wert:hover { background: #ddf6e7; }
  .wert-zahl { display: block; font-size: 1.4rem; font-weight: bold; color: #1a1a2e; }
  .wert-label { display: block; font-size: 0.75rem; color: #999; margin-top: 2px; }

  .tooltip-text { display: none; position: absolute; top: calc(100% + 10px); left: 50%; transform: translateX(-50%); background: #1a1a2e; color: white; font-size: 0.75rem; padding: 10px 14px; border-radius: 8px; width: 230px; line-height: 1.5; z-index: 300; pointer-events: none; text-align: left; box-shadow: 0 4px 16px rgba(0,0,0,0.25); }
  .tooltip-text::after { content: ''; position: absolute; bottom: 100%; left: 50%; transform: translateX(-50%); border: 6px solid transparent; border-bottom-color: #1a1a2e; }
  .detail-wert:hover .tooltip-text { display: block; }

  .detail-wert:first-child .tooltip-text { left: 0; transform: none; }
  .detail-wert:first-child .tooltip-text::after { left: 30px; transform: none; }
  .detail-wert:last-child .tooltip-text { left: auto; right: 0; transform: none; }
  .detail-wert:last-child .tooltip-text::after { left: auto; right: 30px; transform: none; }

  .modal-overlay { position: fixed; inset: 0; background: rgba(15,23,42,0.55); backdrop-filter: blur(3px); display: flex; align-items: center; justify-content: center; z-index: 2000; padding: 20px; }

  .modal { background: #fff; color: #1a1a2e; border-radius: 18px; padding: 28px 30px; max-width: 640px; width: 100%; position: relative; box-shadow: 0 20px 60px rgba(0,0,0,0.35); max-height: 92vh; overflow-y: auto; }
  .modal-methodik { max-width: 640px; }
  .methodik-liste { margin-top: 8px; }
  .methodik-block { padding: 12px 0; border-bottom: 1px solid #f0f0f0; }
  .methodik-block:last-child { border-bottom: none; }
  .methodik-block h4 { margin: 0 0 5px; font-size: 0.92rem; color: #0c8a3a; font-family: 'Helvetica Neue', Arial, sans-serif; }
  .methodik-block p { margin: 0; font-size: 0.9rem; line-height: 1.6; color: #444; }
  .modal-close { position: absolute; top: 14px; right: 16px; background: #f0f0f0; border: none; width: 34px; height: 34px; border-radius: 50%; font-size: 1.3rem; cursor: pointer; color: #555; line-height: 1; }
  .modal-close:hover { background: #e0e0e0; }
  .modal-titel { font-size: 1.7rem; margin: 0; }
  .modal-sub { color: #555; font-size: 0.92rem; margin: 4px 0 16px; }
  .modal-intro { color: #777; font-size: 0.9rem; line-height: 1.6; margin: 0 0 16px; }
  .modal-radar-wrap { margin-top: 20px; border-top: 1px solid #eee; padding-top: 16px; }
  .modal-radar-titel { font-size: 0.95rem; font-weight: 600; color: #555; margin: 0 0 8px; }
  .modal-radar-titel span { font-weight: 700; }

  .radar-info { position: relative; display: inline-block; margin-left: 4px; color: #aaa; font-size: 0.82rem; cursor: help; }
  .radar-info:hover, .radar-info:focus { color: #777; }
  .radar-tooltip { width: 270px; font-weight: normal; }
  .radar-info:hover .tooltip-text, .radar-info:focus .tooltip-text { display: block; }

  .radar-legende { display: flex; gap: 10px; flex-wrap: wrap; margin: 0 0 4px; font-size: 0.78rem; color: #555; }
  .rl-item {
    display: flex; align-items: center; gap: 6px;
    background: #f4f4f8; border: 1px solid #e2e2ea; border-radius: 20px;
    padding: 5px 11px; font-size: 0.78rem; color: #555; cursor: pointer;
    font-family: inherit; transition: background 0.15s, opacity 0.15s, border-color 0.15s;
  }
  .rl-item:hover { background: #ebebf2; }

  .rl-item.rl-aus { opacity: 0.45; text-decoration: line-through; background: transparent; }
  .rl-swatch { width: 18px; height: 0; border-top: 3px solid; display: inline-block; }
  .rl-dash-genre { border-top-style: dashed; border-top-width: 3px; }
  .rl-dash-avg { border-top-style: dotted; border-top-width: 2px; }
  .radar-legende-hint { margin: 0 0 14px; font-size: 0.72rem; color: #999; }
  .modal-body { display: flex; gap: 20px; align-items: center; flex-wrap: wrap; justify-content: center; }
  #svg-radar-modal { flex-shrink: 0; }
  .modal-werte { display: flex; flex-direction: column; gap: 10px; }
  .mw { display: flex; flex-direction: column; align-items: flex-start; }
  .mw-zahl { font-size: 1.5rem; font-weight: bold; }
  .mw-label { font-size: 0.78rem; color: #999; }

  .spotify-embed-wrap { margin-top: 12px; }
  .spotify-embed-wrap iframe { border-radius: 12px; display: block; }

  .play-btn { display: inline-flex; align-items: center; justify-content: center; width: 24px; height: 24px; background: #1DB954; color: white; border: none; border-radius: 50%; font-size: 0.7rem; cursor: pointer; vertical-align: middle; margin-right: 4px; transition: background 0.15s; }
  .play-btn:hover { background: #18a44a; }

  .story-note-embed { margin: 8px 0 0; }

  .story-note-embed :global(iframe) { border-radius: 12px; display: block; }

  .disclaimer { max-width: 560px; margin: 28px auto 0; display: flex; gap: 14px; align-items: flex-start; text-align: left; background: rgba(255,255,255,0.12); border: 1px solid rgba(255,255,255,0.35); border-radius: 12px; padding: 16px 18px; }
  .disclaimer-icon { font-size: 1.4rem; line-height: 1.2; flex-shrink: 0; }
  .disclaimer strong { display: block; color: #fff; font-size: 1.02rem; margin-bottom: 4px; }
  .disclaimer p { margin: 0; color: rgba(255,255,255,0.92); line-height: 1.55; font-size: 0.98rem; }

  .ende-final { max-width: 760px; margin: 0 auto; padding: 64px 20px 90px; }
  .ende-final .disclaimer {
    background: #fff8e1; border: 1px solid #f0c040;
  }
  .ende-final .disclaimer-icon { color: #b8860b; }
  .ende-final .disclaimer strong { color: #1a1a2e; }
  .ende-final .disclaimer p { color: #5a4500; }

  .thanks { margin-top: 60px; padding-top: 40px; border-top: 2px solid rgba(255,255,255,0.3); }
  .ende-final .thanks { border-top-color: #e0e0e0; }
  .thanks-inner { background: #fafaf5; border-radius: 16px; padding: 32px 36px; max-width: 620px; margin: 0 auto; text-align: left; }
  .thanks-inner h3 { font-size: 1.6rem; color: #1a1a2e; margin-bottom: 16px; }
  .thanks-inner p { color: #444; font-size: 0.9rem; line-height: 1.7; margin-bottom: 12px; }
  .thanks-inner a { color: #0c8a3a; text-decoration: underline; }
  .thanks-person { border-top: 1px solid #e5e5e5; border-bottom: 1px solid #e5e5e5; padding: 16px 0; margin: 16px 0; display: grid; grid-template-columns: 1fr 1fr; gap: 16px; }
  .thanks-person p { margin: 0; }
  .thanks-meta { color: #888 !important; font-size: 0.8rem !important; margin-top: 8px !important; }

  @media (max-width: 640px) {
    .thanks-person { grid-template-columns: 1fr; }
    .start h1 { font-size: 2.4rem; }
  }

  .reveal { opacity: 1; transform: none; }
  .reveal.reveal-pre { opacity: 0; transform: translateY(36px); }
  .reveal.reveal-go {
    opacity: 1; transform: translateY(0);
    transition: opacity 0.9s cubic-bezier(0.22, 0.61, 0.36, 1), transform 0.9s cubic-bezier(0.22, 0.61, 0.36, 1);
  }
  @media (prefers-reduced-motion: reduce) {
    .reveal.reveal-pre { opacity: 1; transform: none; }
  }

  .fly { opacity: 1; transform: none; }
  .fly.fly-pre { opacity: 0; }
  .fly-up.fly-pre    { transform: translateY(55px); }
  .fly-left.fly-pre  { transform: translateX(-70px); }
  .fly-right.fly-pre { transform: translateX(70px); }
  .fly.fly-go {
    opacity: 1; transform: none;
    transition: opacity 1s cubic-bezier(0.22, 0.61, 0.36, 1), transform 1s cubic-bezier(0.22, 0.61, 0.36, 1);
  }
  .fly-d2.fly-go { transition-delay: 0.18s; }

  @media (prefers-reduced-motion: reduce) {
    .fly.fly-pre { opacity: 1; transform: none; }
  }

  .guide-tip {

    position: absolute; left: 78px; z-index: 900; width: 250px;
  }
  .guide-tip-fix {

    position: fixed; top: 16%;
    background: #fff; border-right: 4px solid #1DB954; border-radius: 12px 0 0 12px;
    padding: 16px 18px; box-shadow: 0 8px 28px rgba(0,0,0,0.18); color: #1a1a2e;
  }
  .guide-tip-label {
    display: flex; align-items: center; gap: 6px;
    font-size: 0.72rem; letter-spacing: 1px; text-transform: uppercase;
    color: #1DB954; font-weight: 700; margin-bottom: 8px;
    font-family: 'Helvetica Neue', Arial, sans-serif;
  }
  .guide-tip p { margin: 0 0 14px; font-size: 0.9rem; line-height: 1.5; color: #333; }
  .guide-tip .btn-haupt.klein { margin: 0; }
  .tour-ziel {

    box-shadow: 0 0 0 3px #fff, 0 0 0 7px #1DB954, 0 0 0 13px rgba(255,255,255,0.35);
    border-radius: 10px; padding: 8px; background: rgba(255,255,255,0.1);
  }
  @media (max-width: 1100px) {
    .guide-tip {

      position: fixed; left: 78px; right: 16px; width: auto; top: auto; bottom: 16px;
      border-right: none; border-left: 4px solid #1DB954; border-radius: 0 12px 12px 0;
    }
  }

  .mood-radare { display: grid; grid-template-columns: repeat(3, 1fr); gap: 16px; margin-top: 18px; }
  .mood-radar-card { position: relative; background: white; border-radius: 12px; padding: 14px 12px 12px; box-shadow: 0 2px 10px rgba(0,0,0,0.08); display: flex; flex-direction: column; gap: 3px; }
  .mood-radar-rolle { margin: 0; padding-right: 24px; font-size: 0.62rem; font-weight: 700; letter-spacing: 0.8px; text-transform: uppercase; font-family: 'Helvetica Neue', Arial, sans-serif; white-space: nowrap; }
  .mood-radar-titel { margin: 0; font-size: 1.02rem; font-weight: 700; color: #1a1a2e; }
  .mood-radar-kopf { min-height: 4.8em; }
  .mood-radar-artist { margin: 0 0 4px; font-size: 0.82rem; color: #777; }
  .mood-radar-fuss { display: flex; align-items: center; gap: 8px; margin-top: auto; padding-top: 8px; }
  @media (max-width: 760px) { .mood-radare { grid-template-columns: 1fr; } }

  .top5-btn { position: relative; font-family: inherit; font-size: 0.92rem; font-weight: 700; cursor: pointer; border: none; background: #fff; color: #0c8a3a; border-radius: 12px; padding: 10px 18px; margin: 12px 0 12px; box-shadow: 0 4px 16px rgba(0,0,0,0.15); transition: transform 0.15s; }
  .top5-btn::before { content: ''; position: absolute; top: -9px; left: 26px; border-left: 9px solid transparent; border-right: 9px solid transparent; border-bottom: 9px solid #fff; }
  .top5-btn:hover { transform: translateY(-1px); }
  .top5-liste { background: #fff; border-radius: 12px; padding: 10px 14px; box-shadow: 0 4px 18px rgba(0,0,0,0.12); margin-bottom: 14px; }
  .top5-zeile { display: flex; align-items: center; gap: 10px; padding: 7px 2px; border-bottom: 1px solid #f0f0f2; font-size: 0.92rem; color: #1a1a2e; }
  .top5-zeile:last-of-type { border-bottom: none; }
  .top5-rang { font-weight: 800; color: #1DB954; width: 18px; text-align: center; flex-shrink: 0; }
  .top5-titel { flex: 1; min-width: 0; }
  .top5-wert { font-size: 0.8rem; color: #888; white-space: nowrap; }
  .abschluss-note { text-align: center; font-size: 1.08rem; color: #fff; margin: 0 0 18px; }

  .hilfe-chip { position: absolute; top: 10px; right: 12px; z-index: 12; display: inline-flex; align-items: center; gap: 5px; font-family: 'Helvetica Neue', Arial, sans-serif; font-size: 0.72rem; font-weight: 700; letter-spacing: 0.4px; text-transform: uppercase; color: #0c8a3a; background: #f2f9f4; border: 1px solid #cfe9d8; border-radius: 999px; padding: 5px 11px; cursor: pointer; transition: background 0.15s; }
  .hilfe-chip:hover { background: #e3f4e9; }
  .hilfe-panel { position: absolute; top: 44px; right: 12px; z-index: 11; width: min(430px, calc(100% - 24px)); background: #fff; border: 1px solid #e2e8f0; border-left: 3px solid #1DB954; border-radius: 0 10px 10px 0; padding: 12px 14px; font-size: 0.85rem; line-height: 1.55; color: #1a1a2e; box-shadow: 0 8px 26px rgba(0,0,0,0.14); text-align: left; }
  .hilfe-zeile { display: flex; justify-content: flex-end; margin: 0 0 8px; }
  .hilfe-zeile .hilfe-chip { position: static; }
  .hilfe-panel-inline { position: static; width: auto; margin: 8px 0 14px; }

  .text-trigger { position: relative; font: inherit; color: inherit; background: rgba(255,255,255,0.22); border: none; padding: 1px 5px; cursor: pointer; border-radius: 5px; box-decoration-break: clone; -webkit-box-decoration-break: clone; transition: background 0.15s; }
  .text-trigger:hover { background: rgba(255,255,255,0.38); }
  .text-trigger-aktiv { background: rgba(255,255,255,0.92); color: #0c8a3a; }
  .text-trigger:hover::after { content: '👆 Click here'; position: absolute; left: 50%; transform: translateX(-50%); bottom: calc(100% + 7px); background: #1a1a2e; color: #fff; font-family: 'Helvetica Neue', Arial, sans-serif; font-size: 0.7rem; font-weight: 700; letter-spacing: 0.3px; padding: 4px 10px; border-radius: 6px; white-space: nowrap; pointer-events: none; z-index: 60; box-shadow: 0 3px 10px rgba(0,0,0,0.25); }
  .text-trigger-aktiv:hover::after { content: '👆 Click to reset'; }
  .tt-ohne-tip { color: #fff; font-size: 1.25rem; }
  .tt-ohne-tip:hover::after { content: none; }

  .glossar { border-bottom: 2px dotted rgba(255,255,255,0.75); cursor: help; position: relative; }
  .glossar-karte { display: none; position: absolute; left: 50%; transform: translateX(-50%); bottom: calc(100% + 8px); width: 300px; background: #fffdf6; border-radius: 10px; padding: 12px 14px 11px; text-align: left; pointer-events: none; z-index: 60; box-shadow: 0 10px 30px rgba(0,0,0,0.28); }
  .glossar:hover .glossar-karte { display: block; }
  .glossar-karte-unten { bottom: auto; top: calc(100% + 10px); }
  .detail-wert:hover .glossar-karte { display: block; }
  .detail-wert:first-child .glossar-karte { left: 0; transform: none; }
  .detail-wert:last-child .glossar-karte { left: auto; right: 0; transform: none; }
  .glossar-wort { display: block; font-family: Georgia, serif; font-weight: 700; font-size: 1.05rem; color: #1a1a2e; letter-spacing: 0.3px; }
  .glossar-art { display: block; font-family: 'Helvetica Neue', Arial, sans-serif; font-size: 0.7rem; font-weight: 600; color: #b4884b; letter-spacing: 0.4px; margin: 2px 0 8px; }
  .glossar-def { display: block; font-family: 'Helvetica Neue', Arial, sans-serif; font-size: 0.8rem; font-weight: 400; line-height: 1.55; color: #333; }

  .intro-schluss { text-align: center; font-weight: 700; margin-top: 4px; }

  .karte-zoom { position: absolute; top: 6px; right: 8px; background: none; border: none; font-size: 0.82rem; cursor: pointer; opacity: 0.5; transition: opacity 0.15s; padding: 2px; line-height: 1; }
  .karte-zoom:hover { opacity: 1; }
  .mood-zoom { max-width: 440px; width: min(440px, 92vw); }
  .ohne-balken { border-left: none; padding-left: 0; }
  .zentriert { text-align: center; }
</style>