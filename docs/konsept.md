# Konseptbeskrivelse: «VeiViteren»
### Automatisk lydguide for bilreiser – hør historien om det du passerer

*Arbeidstittel. Versjon 0.1 – utkast til diskusjon.*

---

## 1. Idéen i én setning

En app som automatisk og håndfritt forteller deg – på ditt eget språk – om severdigheter, bygninger, naturformasjoner og historiske hendelser i det du nærmer deg dem langs veien, uten at du trenger å planlegge, laste ned ruter eller ta hendene fra rattet.

## 2. Problemet

Alle som har kjørt gjennom Norge eller Europa kjenner situasjonen: du passerer en imponerende stavkirke, en gammel festning, et merkelig fjell eller et krigsminnesmerke – og lurer på hva det er. Å stoppe og google er upraktisk og trafikkfarlig, og i etterkant husker man ikke hvor det var. Informasjonen finnes (Wikipedia alene har artikler med kartkoordinater om hundretusenvis av steder i Europa), men den er ikke tilgjengelig i det øyeblikket den er relevant: bak rattet, i fart, forbi stedet på 30 sekunder.

Eksisterende lydguide-apper har forsøkt å løse dette med manuelt produsert innhold. Det gir høy kvalitet der det finnes, men dekningen blir flekkvis – noen turistruter og byer, og ingenting imellom. Voice of Norway, som var nærmest dette konseptet i Norge, er borte fra markedet. GuideAlong dekker primært Nord-Amerika. izi.TRAVEL er bygget for byvandringer og museer, ikke bilkjøring.

## 3. Løsningen

VeiViteren snur modellen: i stedet for å håndprodusere innhold for utvalgte ruter, genereres innholdet automatisk fra åpne, strukturerte datakilder som allerede dekker hele Europa. Kort fortalt:

1. Appen følger bilens GPS-posisjon, fartsretning og hastighet i bakgrunnen.
2. Den slår kontinuerlig opp interessepunkter (POI-er) langs ruten foran bilen.
3. Når et relevant punkt nærmer seg, hentes kildeteksten (f.eks. Wikipedia-artikkelen om Borgund stavkirke), en språkmodell komprimerer den til en fortelling på 30–90 sekunder tilpasset lytting i bil, og en tekst-til-tale-stemme leser den opp på brukerens valgte språk.
4. Lyden spilles over bilens anlegg (Bluetooth, CarPlay eller Android Auto) og demper eventuell musikk mens den snakker – akkurat som stemmen i et navigasjonssystem.

Brukeropplevelsen skal være: start appen, kjør, og bli fortalt. Ingen planlegging, ingen nedlasting av «ruter», ingen skjermbruk under kjøring.

## 4. Målgrupper

- **Bilturister i eget land og utlandet** – hovedmålgruppen. Nordmenn på norgesferie, europeere på roadtrip, bobilturister.
- **Familier** – «hvorfor heter det Besseggen?» besvart automatisk; egen barnemodus kan være en senere funksjon.
- **Yrkessjåfører og pendlere** – folk som kjører samme strekninger ofte, men aldri har visst hva de passerer.
- **Turistnæringen (B2B, senere fase)** – destinasjonsselskaper, museer og kommuner som vil ha sine egne fortellinger inn i plattformen, slik Voice of Norways kunder gjorde.

## 5. Kjernefunksjonalitet (MVP)

**Automatisk fortelling.** Stedbaserte fortellinger utløses av posisjon, retning og fart. Appen prioriterer det mest interessante når flere punkter ligger nær hverandre, og holder pauser mellom fortellingene så det ikke blir masete.

**Språkvalg.** Brukeren velger språk én gang. Innholdet hentes fra riktig språkversjon av kilden der den finnes (Wikipedia finnes på over 300 språk), ellers oversettes det automatisk. Fortellerstemmen følger språket.

**Interessefilter.** Brukeren kan velge kategorier: historie, arkitektur, natur/geologi, krigshistorie, kirker, industri, folklore, «alt». Og et detaljnivå: «bare høydepunkter» eller «fortell mye».

**Håndfri betjening.** Alt skal kunne styres med stemme eller ett trykk: «hopp over», «fortell mer», «gjenta», «lagre dette stedet».

**Reiselogg.** Alt som er fortalt lagres med kart og lenker, så man etter turen kan se hvor man var og lese mer.

**Offline-modus.** Før en tur kan man laste ned et geografisk område (f.eks. «Vestlandet» eller «Sør-Tyskland») med ferdiggenererte fortellinger, siden mobildekning ikke kan forutsettes i fjell og daler.

## 6. Datakilder

Styrken i konseptet er at innholdet ikke må produseres manuelt:

- **Wikipedia / Wikidata** – geotaggede artikler på alle språk, med koordinater, kategorier og bilder. Fri lisens (CC BY-SA) som tillater gjenbruk med kildehenvisning.
- **OpenStreetMap** – finmasket POI-database: utsiktspunkter, minnesmerker, tunneler, broer, naturformasjoner.
- **Nasjonale åpne kulturminnedata** – i Norge f.eks. Riksantikvarens Askeladden/Kulturminnesøk og Digitalt Museum; tilsvarende registre finnes i de fleste europeiske land.
- **Egenprodusert og partnerprodusert innhold (senere)** – destinasjonsselskaper og lokalhistorikere kan levere kuraterte fortellinger som prioriteres foran de automatiske.

## 7. Teknisk skisse (overordnet)

**Klient (iOS/Android):** posisjonssporing i bakgrunn, lydavspilling med «ducking» av annen lyd, CarPlay/Android Auto-støtte, offline-pakker, enkel kartvisning når bilen står stille.

**Backend:**
- *POI-tjeneste:* database over interessepunkter med koordinater, kategori og «interessantshetsscore» (basert på f.eks. artikkellengde, antall språkversjoner, besøkstall), bygget fra kildene over og oppdatert periodisk.
- *Fortellingsgenerator:* en språkmodell som omskriver kildetekst til muntlig fortelling på ønsket språk og lengde, med faktatro tone og kildehenvisning. Fortellinger caches, slik at populære steder bare genereres én gang per språk – det holder kostnadene lave.
- *Tekst-til-tale:* moderne TTS-stemmer (f.eks. ElevenLabs, Azure eller Google) med naturlig norsk og andre europeiske språk. Lydfiler caches på samme måte.
- *Ruteprediksjon:* enkel logikk som ser på retning og fart for å hente punkter 1–3 km foran bilen, slik at fortellingen starter idet stedet blir synlig – ikke etter at man har passert.

**Om Google Maps:** Google tillater ikke tillegg inne i selve Maps-appen. Løsningen er at VeiViteren kjører parallelt i bakgrunnen og kun leverer lyd, mens Maps (eller hvilken som helst navigasjon) styrer kjøreruten. Dette er teknisk uproblematisk og samme modell som podkast-apper bruker.

## 8. Hva skiller dette fra det som har eksistert

1. **Full dekning fra dag én** – automatisk generert innhold betyr at appen virker på en fylkesvei i Telemark like godt som ved Colosseum, uten at noen har «laget en rute» der.
2. **Ekte flerspråklighet** – språk er et konfigurasjonsvalg, ikke et innholdsprosjekt.
3. **Alltid oppdatert** – kildene vedlikeholdes av verdens største dugnadsprosjekter.
4. **Lave innholdskostnader** – generering + caching koster brøkdeler av manuell produksjon, som trolig var en del av det som gjorde forgjengerne økonomisk sårbare.

## 9. Forretningsmodell (skisse)

- **Freemium:** gratis med f.eks. 10 fortellinger per dag; abonnement (måned/år) for ubegrenset bruk, offline-pakker og premiumstemmer.
- **Reiselivspakke (B2B):** destinasjonsselskaper og museer betaler for kuratert innhold og synlighet i appen.
- Engangskjøp av offline-regioner som alternativ for de som ikke vil ha abonnement.

## 10. Viktigste risikoer og motsvar

- **Trafikksikkerhet:** appen må designes som ren lydopplevelse under kjøring – null skjerminteraksjon i fart. Dette er også et salgsargument.
- **Faktafeil fra AI-generering:** hold fortellingene tett på kildeteksten, oppgi kilde («Kilde: Wikipedia»), og la brukere rapportere feil. Ikke la modellen «dikte» utover kilden.
- **Lisens og attribusjon:** CC BY-SA krever kildeangivelse – løses med kildehenvisning i app og reiselogg.
- **Batteri- og databruk:** kontinuerlig GPS i bakgrunn må optimaliseres; offline-pakker avlaster data.
- **Kostnadskontroll på TTS/AI:** aggressiv caching gjør at marginalkostnaden per bruker faller raskt.
- **Hvorfor forsvant forgjengeren?** Verdt å undersøke: var det innholdskostnader, marked eller finansiering? Lærdommen bør inn i forretningsplanen.

## 11. Forslag til vei videre

1. **Fase 0 – validering (4–6 uker):** enkel prototype som henter geotaggede Wikipedia-artikler langs en kjørerute og leser opp AI-genererte sammendrag. Testes på én strekning, f.eks. Oslo–Lærdal (forbi Borgund stavkirke).
2. **Fase 1 – MVP i Norge:** app i begge appbutikker, norsk + engelsk, hele Norge dekket, freemium.
3. **Fase 2 – Europa og flere språk**, offline-pakker, CarPlay/Android Auto.
4. **Fase 3 – B2B-plattform** for kuratert innhold fra reiselivet.
