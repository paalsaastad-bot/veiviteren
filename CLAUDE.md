# CLAUDE.md – VeiViteren

Stedbasert lydguide for bilreiser. Appen følger bilens GPS, finner severdigheter i en
korridor foran bilen, og leser opp en kort, muntlig fortelling om dem – automatisk, håndfritt,
på brukerens språk. Fungerer i hele Europa fordi innholdet genereres fra åpne data, ikke
håndlages per rute.

## Gyllen regel
Når jeg retter deg på noe, legg til en regel her så feilen ikke gjentas. Hold fila kort
(~100 linjer). Spør heller enn å gjette når noe er uklart.

## Hva vi bygger nå (MVP-avgrensning)
- **Marked:** Norge først. **Språk:** norsk + engelsk (arkitekturen må tåle flere).
- **Plattform:** iOS + Android fra én kodebase.
- **Kjerneopplevelse:** start app → kjør → bli fortalt. Ingen ruteplanlegging, ingen skjermbruk i fart.
- IKKE bygg ennå: offline-pakker, CarPlay/Android Auto-flate, abonnement/betaling, B2B-innhold.
  Arkitekturen skal *tillate* dem senere, men de er utenfor MVP.

## Arkitektur – bærende prinsipper (ikke bryt disse)
1. **Generer én gang, cache for alle.** Fortellinger lages av en LLM første gang et sted
   besøkes på et gitt språk, og gjenbrukes deretter. Marginalkostnaden skal falle mot null.
   Aldri regenerer noe som finnes i cache.
2. **Ferdigbygd POI-database, ikke live-søk.** Klienten slår opp mot vår egen POI-tjeneste,
   som er bygget på forhånd fra Wikipedia/Wikidata + OpenStreetMap. Live web-søk var kun
   for prototypen; det skal IKKE inn i produktet (for tregt, upresist, dyrt).
3. **Klienten styrer timingen.** En korridor 1–4 km foran bilen (posisjon + kurs + fart)
   avgjør hva som fortelles og når. Fortellingen skal treffe idet stedet blir synlig.
4. **Retningsfilter er obligatorisk.** Aldri fortell om ting bak bilen eller på parallellvei.
   Filtrer på vinkel mellom kurs og retning-til-punkt (±~80°).
5. **Lyd-ducking, ikke stopp.** Fortellerstemmen demper annen lyd (musikk/podkast) og lar
   den fortsette etterpå – som en GPS-stemme. iOS: AVAudioSession `.duckOthers`.
   Android: AudioFocus `GAIN_TRANSIENT_MAY_DUCK`.
6. **Faktatro innhold.** Fortellinger holdes tett på kildeteksten. Aldri dikt opp navn, tall
   eller datoer. Oppgi kilde. La brukere rapportere feil.

## Tech-stack (foreslått – bekreft med meg før store valg)
- **Klient:** React Native (kryssplattform). Native moduler kreves for: bakgrunns-GPS,
  lyd-ducking, wake-lock.
- **Backend:** Python (FastAPI) eller Node/TypeScript. PostgreSQL + PostGIS for POI-er.
- **Genereringspipeline:** LLM (Claude) for omskriving → tekst-cache; TTS (Azure/Google/
  ElevenLabs, bak et byttbart grensesnitt) → lyd-cache i objektlager (S3/GCS) bak CDN.
- **Betaling (senere):** RevenueCat mot App Store / Play.
- Ikke lås oss til én skyleverandør unødvendig. TTS-leverandør skal være enkel å bytte.

## Datamodell – POI (kjerne)
`id` (wikidata-id der mulig) · `geom` (PostGIS-punkt) · `kategori` · `navn` (jsonb per språk) ·
`kilder` (jsonb) · `score` (0–100 interessanthet) · `trigger_radius` · `sist_oppdatert`.
Dedupliser på tvers av kilder; Wikidata-id er primærnøkkel når den finnes.

## Fortellings-prompt (regler som funker – behold disse)
- Muntlig stil: korte setninger, skrevet for øret, varm men faktatro.
- Åpne med det føreren ser/nærmer seg NÅ. Aldri åpne med en leksikondefinisjon.
- Én rød tråd per fortelling – ikke alt kilden vet.
- Gjør abstrakte tall konkrete ("én meter opp for hver fjerde meter frem", ikke "stigning 1:4").
- To lengder: kort (~70–90 ord) og lang (~200–240 ord, for "fortell mer").
- Bare fakta fra kilden. Nevn ikke søk/Wikipedia/instruksjoner i selve fortellingen.

## Personvern og jus (ta på alvor fra dag én)
- Posisjonsdata er sensitivt. Dataminimering: korridor-oppslag lagres ikke per bruker.
  Reiselogg ligger primært lokalt på enheten; skysynk er eksplisitt opt-in (GDPR).
- Wikipedia-tekst er CC BY-SA, OSM er ODbL → attribusjon kreves. Avklar lisens for avledet
  (omskrevet) tekst juridisk før lansering. Bygg attribusjon inn i app + reiselogg.
- Trafikksikkerhet: ingen skjerminteraksjon i fart. Stemme/ett-trykk-styring når fart > ~10 km/t.

## Arbeidsmåte
- Før store features: still meg oppklarende spørsmål, foreslå en kort plan, vent på ok.
- Bygg i små, testbare steg. Forklar hva du gjør før du endrer flere filer.
- Bruk Git. Commit ofte med tydelige meldinger. Aldri commit hemmeligheter/API-nøkler
  (bruk .env, som ligger i .gitignore).
- Start read-only / forklarende når du er usikker, før du får lov å endre.

## Gjør aldri
- Aldri legg API-nøkler eller hemmeligheter i kildekode eller commits.
- Aldri bruk live web-søk som POI-kilde i produktet (kun prototypen gjorde det).
- Aldri regenerer en fortelling som finnes i cache.
- Aldri fortell om et punkt uten å ha bestått retningsfilteret.
- Aldri la fortellinger inneholde fakta som ikke står i kilden.
- Aldri krev skjerminteraksjon mens bilen er i bevegelse.

## Referansemateriale i repoet
- `docs/konsept.md` – konseptbeskrivelse
- `docs/arkitektur.md` – teknisk arkitekturskisse (autoritativ på systemdesign)
- `docs/prototyper/` – de tre HTML-prototypene. Viser ØNSKET oppførsel (auto-triggering,
  retningsfilter, fortellingstone), IKKE ønsket teknologi. Produktet er native + POI-database.

## Åpne spørsmål (avklar med meg, ikke anta)
- Én norsk fortellerstemme til å begynne med, eller flere?
- Hvor aggressiv skal pre-generering av POI-er være ved lansering (kostnad vs. førstegangsopplevelse)?
- Skal fortellinger kunne si hvilken side av veien stedet er på?
