# StudentOS – produktbeskrivelse

## Formål
Et personlig dashboard for studenter som samler alt studierelatert på ett
sted: oppgaver, frister, timeplan og karakterstatistikk — uten å måtte hoppe
mellom Canvas, Google Kalender og Karakterweb.

## Faser
1. **Personlig bruk** — kun for deg selv, full funksjonalitet
2. **Venner** — venner på andre universiteter kobler seg til
3. **Eventuelt offentlig lansering** — se CLAUDE.md for hva som kreves da

---

## Onboarding
- E-post/passord-registrering (Supabase Auth)
- Steg: 1) opprett konto → 2) velg universitet fra liste → 3) lim inn Canvas
  access token → 4) koble til Google-konto via OAuth → 5) sett
  arbeidstidspreferanse → 6) ferdig, gå til dashboard
- Canvas-token og Google-tilkobling er **per bruker**, lagres knyttet til
  brukerens profil — aldri delt/globalt
- Bruker kan koble til/fra og oppdatere Canvas-token og Google-tilkobling
  senere via en Innstillinger-side
- Arbeidstidspreferanse: fast ønsket sluttklokkeslett (ikke timetall) —
  f.eks. "vil være ferdig med studier kl 16:00". Brukes til "Ferdig for
  dagen"-tidspunktet i AI-planen
  - AI-en må avslutte studieblokker med god margin før andre faste
    hendelser i Google Calendar samme dag (jobb, trening osv.) — ikke bare
    unngå å overlappe dem, men gi realistisk tid til å faktisk rekke dit
  - Studenten velger selv hvilke ukedager som er "arbeidsdager" i
    onboarding/Innstillinger — AI-en genererer kun studieplan for disse
    dagene. Andre dager vises kun kalenderhendelser, ingen "jobbe med"-blokker
- Karakterweb-nøkkelen er derimot **én app-wide nøkkel** (miljøvariabel),
  siden dataene er generell kullstatistikk, ikke personlig

## 1. Dashboard (forside)
- Live-oppdaterende klokke og dato i header på alle sider — én felles kilde
  til "nå-tidspunkt" som både bruker og AI-logikk bruker
- "I dag" og "denne uken"-visning
- Kommende frister (nærmeste 5–7) med emnekode, oppgavenavn, forfallsdato,
  status (ikke startet / pågår / levert)
  - "Levert" settes automatisk fra Canvas' submission-status
  - "Ikke startet" → "Pågår" settes manuelt av studenten (enkel toggle/klikk
    på oppgaven) — ingen automatikk basert på tid eller dagslogg
- Dagens/ukens kalenderhendelser
- Kort statusoversikt per emne (f.eks. "3 aktive oppgaver, snittkarakter B")

### 1b. Klokke som delt datakilde
- Serverens klokke er autoritativ kilde for "nå" (ikke klientens
  nettleserklokke), slik at tidsstempler og frist-beregninger er konsistente
- Gjeldende dato/klokkeslett sendes eksplisitt inn i AI-prompten som kontekst
  hver gang en anbefaling genereres
- Klokken i UI og AI-ens tidsstempel viser alltid samme tid/tidssone
  (Europe/Oslo)

## 2. Canvas-integrasjon
- Autentisert med personlig access token (bruker genererer selv i Canvas:
  Account → Settings → New Access Token)
- Hent aktive emner: `GET /api/v1/courses`
- Hent oppgaver/innleveringer: `GET /api/v1/courses/:course_id/assignments`
  (frist `due_at`, leverings-status fra `submission`, direktelenke `html_url`)
- Kun leverings-status vises (levert/ikke levert) — Canvas' egen poengsum og
  tilbakemelding vises IKKE i appen (studenten sjekker det i Canvas selv)
- Token og Canvas-domene lagres kryptert, per bruker
- Filtrering: per emne, forfalt/kommende/levert
- Visuell markering: rødt innen 48 timer, gult innen en uke
- Periodisk henting (hvert 15.–30. minutt) eller on-demand med kort cache

## 3. Google Calendar-integrasjon
- OAuth 2.0, `calendar.readonly`-scope (ingen skrivetilgang nødvendig)
- Access + refresh token lagres per bruker
- Hent hendelser: `GET /calendar/v3/calendars/primary/events`
- Uke- og dagsvisning
- "Neste hendelse"-widget på dashboardet

## 4. Karakterweb-integrasjon
- Offentlig REST-API: `https://api.karakterweb.no`
- Endepunkt: `GET /v1/courses/{institusjon}/{emnekode}?include=grades,evaluations`
- Auth: header `X-Client-Key: kw_pk_xxx`
- Rate-limit 120 req/min — bygg inn caching (maks 1 kall/time per emne,
  delt cache på tvers av brukere)
- Kompakt kursvurderings-widget per emne (2–3 nøkkeltall: vanskelighetsgrad,
  arbeidsmengde vs. studiepoeng, generelt inntrykk) — **ikke** en egen
  dedikert emneside som ligner Karakterwebs egen layout (brudd på vilkårene)
- Vis alltid "Kilde: Karakterweb" som synlig attribusjon
- Gir kun kullstatistikk (gjennomsnitt for alle studenter), ikke studentens
  egne personlige karakterer — appen har ikke og skal ikke ha tilgang til det

## 5. AI-assistent: "Hva bør jeg jobbe med i dag?"
- AI-drevet anbefaling basert på ekte data, ikke statisk tekst
- **Planlegger fremover, ikke bare reaktivt:**
  - Ser på frister 2–3 uker frem, fordeler arbeidsmengde utover dagene slik
    at ingen dag blir overbelastet rett før en frist
  - Ved kolliderende/tette frister: legger inn arbeidsøkter på den ene
    tidligere, venter ikke til dagen før
  - Vekter arbeidsmengde per emne mot Karakterweb-data (vanskelighetsgrad,
    arbeidsmengde vs. studiepoeng) — tidkrevende emner får jevnlige økter
    over tid i stedet for én lang økt rett før frist
  - Tar hensyn til kommende kalenderhendelser (eksamener, tette
    undervisningsuker) — flytter forberedelse tidligere ved behov
  - Dagens konkrete plan er alltid basert på denne fremtidsrettede
    prioriteringen
- Vurderer flere kilder samlet: Canvas-frister/status + Karakterweb-data
  (høy vanskelighetsgrad/arbeidsmengde → høyere prioritet)
- **Presenteres som konkret tidsplan, ikke tekstbeskrivelse:**
  - Kronologisk liste med tidsblokker, f.eks.:
    - `09:00–10:00` — Forelesning: DAT200 (fra Canvas sin kalender)
    - `10:15–12:00` — Jobb med innlevering: DAT200-oppgave 3 (frist om 2 dager)
    - `12:00–13:00` — Lunsj
    - `13:00–14:30` — Jobb med innlevering: INF100-oppgave 2
    - `14:30` — **Ferdig for dagen**
  - Kalenderhendelser (forelesninger, øvingstimer, jobbvakter, alt) er faste,
    ikke-flyttbare blokker — AI planlegger studiearbeid rundt disse
  - "Jobbe med"-blokker fylles i ledig tid, basert på frist + vanskelighetsgrad
    + gårsdagens fremgang fra dagsloggen (5b)
  - Planlegges innenfor brukerens arbeidstidsramme, viser tydelig sluttidspunkt
  - Hvis ledig tid ikke strekker til (f.eks. eksamensperioder): AI kan
    foreslå å utvide dagen — vises eksplisitt som varsel-boks
    ("Utvidet til kl. 17:00 i dag — for mye på fristen ellers"), aldri stille
  - Hver "jobbe med"-blokk viser kort *hvorfor* som undertekst (f.eks.
    "frist om 2 dager"), fullt resonnement kan vises ved å utvide blokken
- Regnes automatisk på nytt: (1) hver gang dashboardet åpnes, (2) umiddelbart
  når en oppgave endrer status til levert (manuelt eller via Canvas'
  submission-status) — ingen manuell knapp nødvendig, men fint som fallback

### Lagret økt-plan ("planned sessions")
- Når AI-en fordeler arbeid fremover (f.eks. "jobb med DAT200 hver onsdag i
  3 uker"), lagres dette som en egen "tildelt økt"-historikk i databasen —
  ikke noe som regnes helt fra bunnen hver dag
- Ved ny beregning: AI-en tar hensyn til allerede planlagte fremtidige økter
  og justerer/utvider planen i stedet for å overskrive den fra scratch,
  med mindre noe vesentlig har endret seg (f.eks. ny frist, endret kalender)
- Studenten kan manuelt flytte eller justere en fremtidig planlagt økt
  (f.eks. bytte dag på en "jobb med DAT200"-økt) — dette lagres og AI-en
  respekterer justeringen ved neste omberegning i stedet for å overstyre den
- Dagens konkrete plan bygges fra den delen av økt-planen som gjelder i dag

### Eksamener
- AI-en legger inn egne repetisjons-/leseblokker i ukene før en eksamen
  (samme logikk som for innleveringer: fordelt over tid, ikke én stor økt
  rett før)
- **v1: eksamensdato registreres manuelt av studenten** (i emnet sine
  innstillinger, eller når emnet legges til) — det finnes ikke noe pålitelig
  API for eksamensdatoer (de ligger typisk på emnets side på universitetets
  nettsider eller i Studentweb, ikke i Canvas eller Google Calendar)
  - Fremtidig forbedring (ikke v1): la studenten evt. legge eksamen inn i
    Google Calendar i stedet, og la appen lese den derfra som en vanlig
    kalenderhendelse merket som eksamen

## 5b. Start dag / Avslutt dag
- "Start dagen"-knapp: logger tidspunkt (fra 1b), låser inn dagens plan
- "Avslutt dagen" → oppsummeringsskjerm:
  - Oppgaver levert i Canvas i løpet av dagen listes automatisk som fullført
    (ingen manuell bekreftelse nødvendig)
  - For oppgaver fra dagens plan som ikke er levert: spør om status
    (0/25/50/75/nesten ferdig + valgfritt kommentarfelt)
  - Lagres som dagslogg-post (dato, oppgave, fremgang %, kommentar)
- Dagsloggen inngår som kontekst i AI-prompten for **neste dags** plan
  (f.eks. "Fortsett på DAT200-oppgaven — du kom halvveis i går")
- Hvis "Avslutt dagen" ikke trykkes: AI skal IKKE anta at ingenting ble
  gjort — vekt Canvas' faktiske leveringsstatus tyngst, behandle manglende
  dagslogg som "ukjent", ikke "0% gjort"

## 5c. Varsler
- Nettleservarsler for kommende blokker (f.eks. "Forelesning DAT200 om 15 min")
- Standard varseltid: 15 minutter før blokk starter — studenten kan justere
  dette selv i Innstillinger (ett felles tidspunkt for alle varsler, ikke
  per blokk-type i v1)
- **Av som standard** — må aktiveres i Innstillinger + nettleser-tillatelse

## 6. Notatsystem
- Notion-inspirert, **Markdown-basert** redigering (som Notion/Obsidian) —
  ren markdown lagres, rendres pent i UI (overskrifter, lister, sjekklister,
  fet/kursiv), studenten kan skrive rå markdown eller bruke enkle
  formaterings-snarveier
- Manuell organisering — studenten velger emne og oppretter notat der,
  ingen automatisk AI-plassering
- Automatisk forslag til filnavn/dato ved nytt notat (ikke-AI, f.eks.
  "DAT200 – 02.09.2026")
- To visninger: (1) per emne som mappe med kun det emnets notater,
  (2) samlet "Alle notater" fra sidemenyen, sortert på dato, emnekode
  som filter/tag
- AI-assistenten kan bruke notatinnhold som kontekst (f.eks. foreslå
  repetisjon, koble til kommende oppgaver) — plassering styres alltid
  av studenten

## Kontosletting
- Når en student sletter kontoen: Canvas-token, notater, dagslogger og
  økt-historikk fjernes/anonymiseres helt — ingen "myk sletting" eller
  oppbevaringsperiode

## Gamle semestre / arkiv
- Emner som ikke lenger er aktive i Canvas (fullførte/gamle semestre) vises
  fortsatt i en egen arkiv-visning, sammen med tilhørende notater
- Arkiverte emner får ingen nye AI-anbefalinger eller frist-varsler, men
  notater og historikk forblir tilgjengelig og søkbare

## Datakonsistens
- Emner hentes/utledes fra Canvas — dashboard, kalender, karakterer,
  kursvurderinger og notater bruker samme emner som er aktive i Canvas
- Canvas-emnekoder mappes til Karakterweb-format (f.eks. "DAT200" →
  `uib/dat200`) — se institusjons-mapping i CLAUDE.md

## Design
- Mørk modus som standard, lys modus som alternativ
- Card-basert layout
- Sidemeny: Dashboard, Emner, Kalender, Karakterer, Alle notater
- Oversiktlighet fremfor informasjonstetthet — skal sjekkes raskt mellom
  forelesninger
- Kun desktop — ikke prioriter mobilresponsivitet

## Data
- Norske datoer (dd.mm.åååå) og karakterskala (A–F)
- Ingen mock-data i ferdig app — alle tre integrasjoner henter ekte, live
  data

## Nødvendige nøkler/tilganger
1. **Canvas access token** — Account → Settings → New Access Token i Canvas,
   sammen med institusjonens Canvas-domene
2. **Google Calendar OAuth-klient** — opprettes i Google Cloud Console:
   prosjekt, aktiver Calendar API, OAuth-klient-ID/secret med
   `calendar.readonly`-scope
3. **Karakterweb API-nøkkel** (`kw_pk_xxx`) — søk på karakterweb.no/api-docs.
   Viktig fra vilkårene:
   - Ikke lov å gjenskape Karakterwebs egne emnesider
   - Lov å integrere emnevurderinger i "et helt annet verktøy"
   - Krav om synlig kildehenvisning der data vises
   - Rate-limit 120 kall/min — bygg inn caching