# Teknisk arkitekturskisse: VeiViteren
### Stedbasert, automatisk lydguide for bilreiser

*Versjon 0.1 – utkast. Bygger på konseptbeskrivelsen v0.1.*

---

## 1. Arkitektur på øverste nivå

Systemet består av tre hoveddeler:

```
┌─────────────────────────────────────────────────────────────┐
│  MOBILKLIENT (iOS / Android)                                 │
│  Posisjon · Ruteprediksjon · Lydavspilling · Offline-lager   │
└───────────────▲─────────────────────────┬───────────────────┘
                │ lyd/metadata            │ posisjon/forespørsler
                │                         ▼
┌─────────────────────────────────────────────────────────────┐
│  API-LAG (backend)                                           │
│  POI-oppslag · Fortellingstjeneste · Brukere/abonnement      │
└───────▲───────────────▲───────────────▲─────────────────────┘
        │               │               │
┌───────┴─────┐ ┌───────┴───────┐ ┌─────┴──────────┐
│ POI-database│ │ Genererings-  │ │ Innholds-cache │
│ (PostGIS)   │ │ pipeline      │ │ (tekst + lyd)  │
│             │ │ (LLM + TTS)   │ │                │
└───────▲─────┘ └───────────────┘ └────────────────┘
        │
┌───────┴──────────────────────────────────────────┐
│ DATAINNHENTING (batch, kjøres periodisk)          │
│ Wikipedia/Wikidata · OpenStreetMap · Kulturminne- │
│ registre · Partnerinnhold                         │
└───────────────────────────────────────────────────┘
```

Et bærende prinsipp: **så mye som mulig genereres og caches på forhånd**, slik at det som skjer i sanntid mens noen kjører, stort sett er billige databaseoppslag og nedlasting av ferdige lydfiler.

---

## 2. Datainnhenting og POI-databasen

### 2.1 Kilder og innhenting

| Kilde | Hva hentes | Hvordan | Frekvens |
|---|---|---|---|
| **Wikidata** | Alle entiteter med koordinater i dekningsområdet: type (Q-klasse), koordinater, lenker til Wikipedia-artikler per språk, bilde | SPARQL-endepunkt / dumps | Ukentlig |
| **Wikipedia** | Artikkeltekst (innledning + relevante seksjoner) per språk | REST API / dumps | Ved generering + ukentlig sjekk av endringer |
| **OpenStreetMap** | POI-er med tags som `historic=*`, `tourism=attraction/viewpoint`, `natural=*` (fosser, tinder), `man_made=bridge/lighthouse` osv. | Overpass API / planet-utdrag (Geofabrik) | Ukentlig |
| **Kulturminneregistre** | Norge: Riksantikvarens åpne data (Askeladden/Kulturminnesøk). Senere: tilsvarende per land | Åpne API-er / datasett | Månedlig |
| **Partnerinnhold** | Kuraterte fortellinger med egne geofences | Eget CMS/admin-API | Løpende |

Innhentingen kjøres som batch-jobber (f.eks. Airflow eller enkle cron-baserte jobber i starten) som normaliserer alt til én felles POI-modell og **dedupliserer på tvers av kilder** (samme kirke finnes gjerne i alle fire kildene – Wikidata-ID brukes som primærnøkkel der den finnes, ellers matching på navn + avstand).

### 2.2 POI-datamodell (forenklet)

```
poi
├── id  (uuid; wikidata_id der den finnes)
├── geom  (PostGIS-punkt, ev. polygon for store objekter)
├── kategori  (kirke, festning, naturformasjon, minnesmerke, …)
├── navn  (per språk, jsonb)
├── kilder  (jsonb: wikipedia-titler per språk, osm-id, askeladden-id, …)
├── score  (0–100, «hvor interessant», se 2.3)
├── synlighet  (synlig fra vei? avstand til nærmeste vei; beregnes mot OSM-veinett)
├── trigger_radius  (dynamisk: stor for fjell, liten for bygning)
└── sist_oppdatert
```

Lagres i **PostgreSQL + PostGIS**. Geografiske spørringer («alt innen 3 km foran bilen, i kjøreretningen») er nettopp det PostGIS er laget for, og hele Europas POI-mengde (grovt anslag 2–5 mill. rader) er uproblematisk for én database.

### 2.3 Interessantshetsscore

Når flere POI-er ligger langs samme veistrekning må appen velge. Hver POI får en forhåndsberegnet score basert på f.eks.:

- antall Wikipedia-språkversjoner (godt proxy for global interesse)
- artikkellengde og antall sidevisninger (Wikimedia pageviews-API)
- kildetype (fredet kulturminne > generisk OSM-punkt)
- kategori-boost fra brukerens interessefilter (legges på i sanntid)

Partnerinnhold får alltid høyest prioritet i sitt område.

---

## 3. Genereringspipeline (LLM + TTS)

### 3.1 Flyt

```
POI valgt for generering
   │
   ▼
Hent kildetekst (Wikipedia på målspråk; fallback: engelsk/norsk + oversettelse)
   │
   ▼
LLM-omskriving → muntlig fortelling i 2 lengder:
   • kort (~30 sek / ~80 ord)
   • lang (~90 sek / ~230 ord, for «fortell mer»)
   │
   ▼
Kvalitetssjekk (automatisk):
   • faktatro mot kilde (LLM-basert sjekk)
   • ingen tall/navn som ikke finnes i kilden
   • lengde, leservennlighet
   │
   ▼
TTS → lydfil (mp3/aac, mono, ~48 kbps holder for tale)
   │
   ▼
Lagres i cache: (poi_id, språk, lengde, versjon) → tekst + lyd-URL
```

### 3.2 Nøkkelvalg

- **Prompt-prinsipp:** modellen får kun kildeteksten og instrueres strengt om å ikke tilføre fakta utover den. Fortellingen skal åpne med det man ser («Kirken du nå passerer på høyre side …» genereres ikke – siden vi ikke vet siden – men «Foran deg ligger…» kan styres av klienten), og avslutte kort. Kildehenvisning lagres som metadata og vises i reiseloggen.
- **Generering skjer lazily med cache:** første gang noen i verden passerer et POI på et gitt språk, genereres fortellingen (2–5 sek – klienten ber om punkter 1–3 km frem, så det rekker). Deretter ligger den i cache for alle. De 50 000 mest populære POI-ene per språk kan pre-genereres.
- **TTS:** start med en skyleverandør med god norsk (Azure Neural, Google WaveNet/Chirp eller ElevenLabs). Abstraher bak et internt grensesnitt slik at leverandør kan byttes – prisene i dette markedet faller raskt.
- **Kostnadsbilde (grovt):** en fortelling på 230 ord koster i størrelsesorden øre å generere (LLM) og noen øre å syntetisere (TTS). Med caching er dette en engangskostnad per POI/språk – totalkostnaden for å dekke f.eks. hele Norge på norsk (~50–100k relevante POI-er) er i titusenkronersklassen, ikke millioner. Dét er hele forretningsfordelen mot manuelt innhold.

---

## 4. Mobilklienten

### 4.1 Plattformvalg

Kryssplattform (**React Native** eller **Flutter**) er fornuftig for å holde ett kodebase, men de tre kritiske modulene må håndteres med native-moduler uansett:

1. **Bakgrunnsposisjon** – iOS og Android har strenge og ulike regler for GPS i bakgrunn. På iOS: `CLLocationManager` med `allowsBackgroundLocationUpdates` og «significant location»-strategier; på Android: foreground service med vedvarende varsel (samme mønster som treningsapper).
2. **Lydavspilling med ducking** – appen skal senke (ikke stoppe) musikk/podkast mens fortelleren snakker: iOS `AVAudioSession` med `.duckOthers`, Android `AudioFocus` med `AUDIOFOCUS_GAIN_TRANSIENT_MAY_DUCK`. Dette er samme mekanisme navigasjonsapper bruker og fungerer sømløst over Bluetooth/CarPlay/Android Auto.
3. **CarPlay / Android Auto** – i MVP holder det at lyden går over bilanlegget (krever ingenting spesielt). Egen CarPlay-app-flate (kart + «hopp over»-knapp) er fase 2; merk at CarPlay krever egen tillatelse fra Apple (audio- eller navigasjonskategori).

### 4.2 Ruteprediksjon og triggering (kjernen i klientlogikken)

Klienten – ikke serveren – avgjør *når* noe spilles. Enkel og robust algoritme:

```
hvert 15.–30. sekund (eller ved 500 m bevegelse):
  1. Beregn «korridor»: en kjegle 1–4 km foran bilen,
     basert på posisjon, kurs (bearing) og fart.
     Høy fart (E-vei) → lengre og smalere korridor.
  2. Spør API-et (eller lokal offline-database) om POI-er
     i korridoren, sortert på score.
  3. Filtrer: allerede fortalt på denne turen? kategori slått av?
     for langt fra veien til å ses?
  4. Legg beste kandidat i kø. Startavspilling beregnes slik at
     fortellingen er ferdig omtrent når man er nærmest punktet.
  5. Regler mot mas: min. X sekunder stillhet mellom fortellinger
     (brukerstyrt: «rolig / normal / ivrig»), aldri avbryt en
     pågående fortelling for en med lavere score.
```

Retningsfilteret er viktig: uten det får man fortellinger om ting bak seg eller på parallellveier. Kurs fra GPS ved fart > 20 km/t er pålitelig nok.

### 4.3 Offline-pakker

En offline-region = nedlastbar SQLite-fil med POI-ene (geometri + metadata) **pluss** ferdige lydfiler for kort-versjonen på valgt språk. Grovt estimat: et fylke med 5 000 POI-er à ~250 kB lyd ≈ 1,2 GB – i praksis pakkes bare de N høyest scorede (f.eks. topp 2 000 ≈ 500 MB), resten strømmes ved dekning. Klienten bruker samme triggering-logikk mot lokal database; sømløs overgang online/offline.

### 4.4 Lokal tilstand

- Reiselogg (SQLite): tidspunkt, posisjon, poi_id, lyttet helt/hoppet over → grunnlag for «les mer»-visning etter turen og for forbedring av scoring.
- Innstillinger: språk, stemme, kategorier, hyppighet, detaljnivå.

---

## 5. API-laget

Lite og enkelt i MVP – tre hovedendepunkter:

```
POST /v1/pois/along-route
  inn: posisjon, kurs, fart, filtre, språk
  ut:  liste av POI-er i korridoren med score, trigger-info
       og URL-er til tekst/lyd (genererer ved cache-miss)

GET  /v1/story/{poi_id}?lang=no&length=long
  ut:  tekst + lyd-URL («fortell mer»-funksjonen)

GET  /v1/offline/regions  +  GET /v1/offline/regions/{id}
  ut:  tilgjengelige pakker / nedlasting
```

I tillegg standard bruker-/abonnementshåndtering (kan outsources til f.eks. RevenueCat for kvitteringshåndtering mot App Store/Play).

**Teknologivalg backend:** udramatisk – f.eks. Python (FastAPI) eller Node/TypeScript, kjørende på en enkel skyplattform (Cloud Run/Fly.io/ECS). Lydfiler i objektlager (S3/GCS) bak CDN (CloudFront/Cloudflare) – det er CDN-en som tar trafikken, ikke API-et.

---

## 6. Personvern og regulatorisk

- **Posisjonsdata er sensitivt.** Design for dataminimering: korridor-oppslagene trenger ikke lagres knyttet til bruker; reiseloggen bør primært ligge lokalt på telefonen, med skysynk som eksplisitt opt-in. GDPR-messig er dette den enkleste og ærligste modellen.
- **Lisenser:** Wikipedia-tekst er CC BY-SA → attribusjon i app/reiselogg og deling av avledet tekst under samme lisens må avklares juridisk (omskrevne sammendrag er avledede verk). OSM-data er ODbL. Dette er håndterbart, men må inn i vilkårene fra start.
- **Trafikksikkerhet:** ingen skjerminteraksjon i fart – klienten låser UI til store «hopp over/mer»-knapper eller stemmestyring når fart > ~10 km/t.

---

## 7. Skalering og kostnadsdrivere

| Komponent | Driver | Kommentar |
|---|---|---|
| PostGIS-oppslag | aktive biler × oppslag/min | Én ordentlig database + read replicas tar dette langt; oppslagene er indekserte og billige |
| LLM-generering | nye (poi, språk)-par | Faller mot null over tid pga. cache |
| TTS | samme | Samme |
| CDN/lyd | lytteminutter | Størst løpende kostnad, men tale-bitrate er lav |
| GPS/batteri (klient) | – | Viktigste «kostnad» for brukeren; optimaliser med adaptiv oppslagsfrekvens |

---

## 8. Forslag til MVP-kutt (fase 0-prototype)

For å validere på 4–6 uker, kutt alt som ikke trengs:

1. **Kun Norge, kun norsk**, kun Wikidata/Wikipedia som kilde.
2. Ingen offline, ingen abonnement, ingen CarPlay-flate (lyd over Bluetooth holder).
3. Triggering med enkel radius i stedet for korridor (aksepter noen «feil side av veien»-fortellinger).
4. Generering on-demand uten pre-caching.
5. Test på strekningen Oslo–Lærdal og mål: andel fortellinger som oppleves relevante, timing (før/etter passering), og hvor ofte det blir «for mye prat».

Det som *ikke* bør kuttes selv i prototypen: lyd-ducking og retningsfilter – det er de to tingene som avgjør om opplevelsen føles magisk eller irriterende.

---

## 9. Åpne spørsmål til neste runde

- Stemmevalg: én gjennomarbeidet norsk stemme, eller flere fra start?
- Skal fortellinger kunne nevne siden av veien («på venstre side»)? Krever at klienten sender relativ posisjon og at teksten genereres i to varianter eller med plassholder.
- Hvor aggressiv skal pre-generering være ved lansering (kost vs. førstegangsopplevelse)?
- B2B-CMS: bygge selv eller bruke hyllevare (f.eks. headless CMS med geo-felter)?
