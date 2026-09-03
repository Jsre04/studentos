@AGENTS.md

# StudentOS – prosjektkontekst

## Hva appen er
Dashboard for studenter som samler Canvas-oppgaver, Google Calendar-hendelser
og Karakterweb-kursstatistikk, og gir en AI-generert dagsplan. Se PRODUCT.md
for full produktbeskrivelse (alle funksjoner, datastrukturer for AI-planen,
onboarding-flyt osv.) — dette dokumentet dekker teknisk arkitektur og
byggerekkefølge.

## Faser (viktig for hva som prioriteres når)
1. **Nå: personlig bruk** — kun for deg selv. Bygg full funksjonalitet, men
   ikke bruk tid på ting som bare trengs ved skalering (se "Krav for offentlig
   lansering" nederst — utsett dette).
2. **Neste: venner** — legges til som "test users" i Google Cloud Console
   (fungerer for inntil 100 kjente brukere uten full OAuth-verifisering; de
   ser en "usikker app"-advarsel de må klikke forbi, helt greit blant venner).
   Siden venner sannsynligvis går på andre universiteter enn deg, bygg
   multi-institusjon-støtte inn i arkitekturen fra start (se under) — det
   koster lite ekstra nå, men er dyrt å legge til i etterkant.
3. **Eventuelt senere: offentlig lansering** for alle studenter. Da trengs
   Google OAuth-verifisering, personvernerklæring, sikker nøkkelhåndtering
   utover det som trengs for få brukere, osv. Ikke bygg dette nå — men ikke
   lås arkitekturen til kun én institusjon heller, siden det gjør fase 3 dyr.

## Stack
- Next.js (App Router) + TypeScript + Tailwind
- Supabase: auth (e-post/passord + Google OAuth med calendar.readonly-scope),
  Postgres, Row Level Security (RLS) for per-bruker-isolasjon, EU-region
- Anthropic API for AI-anbefalingen (server-side kall)
- Hosting: Vercel

## Multi-institusjon (viktig)
- `canvas_domain` er et fritekstfelt per bruker (f.eks. `uib.instructure.com`,
  `uio.instructure.com`, `ntnu.instructure.com`) — ikke hardkod ett domene
- Karakterweb bruker institusjons-slugs (`uio`, `uib`, `ntnu`, `kristiania` osv.)
  som ikke nødvendigvis matcher Canvas-domenet 1:1 — trenger en enkel mapping-
  tabell fra "universitet valgt i onboarding" → riktig Karakterweb-slug
- Onboarding bør la studenten velge universitet fra en liste (samme liste som
  Karakterweb har: UiO, UiB, NTNU, UiS, USN, UiT, NORD, UiA, HVL, NMBU,
  OsloMet, BI, Kristiania, NHH, VID, HiØ, INN, HiMolde, NLA, ONH, Noroff, HVO)
  i stedet for fritekst, for å unngå feilstavede Canvas-domener

## Datamodell (Supabase/Postgres)
- `profiles`: id (auth.users), institution, canvas_domain, canvas_token
  (kryptert via Supabase Vault eller tilsvarende), work_hours_per_day /
  preferred_end_time
- `notes`: id, user_id, course_code, title, content, created_at
- `day_logs`: id, user_id, date, assignment_id, progress_pct, comment
- `karakterweb_cache`: course_code, institute, data (jsonb), fetched_at
  (delt cache på tvers av brukere — mange studenter ser samme emne)
- `planned_sessions`: id, user_id, course_code, assignment_id (nullable, for
  eksamens-repetisjon uten konkret oppgave), planned_date, reason (kort
  tekst, f.eks. "frist om 2 dager"), moved_by_user (bool), status
- `exams`: id, user_id, course_code, exam_date (registreres manuelt av
  studenten i v1 — ikke hentet fra noe API)
- `work_days`: hvilke ukedager (0–6) studenten regner som arbeidsdager —
  kan ligge som array-felt på `profiles` i stedet for egen tabell

## Viktige regler
- Canvas-token og Google-tokens er PER BRUKER, aldri globale secrets
- Canvas-token kryptert med ordentlig nøkkelhåndtering (Supabase Vault e.l.),
  ikke en enkel kryptert kolonne — dette er ekte studentdata på skala
- Karakterweb-nøkkelen (X-Client-Key) ER en global env-variabel (delt data)
- Alle eksterne API-kall (Canvas, Google, Karakterweb, Anthropic) skjer
  server-side (Route Handlers / Server Actions), aldri fra klienten
- Cache Karakterweb-data maks 1 kall/time per emne, delt på tvers av brukere
  (rate-limit 120/min blir reelt med mange brukere på samme emner)
- Norsk dato-format (dd.mm.åååå), norsk karakterskala A–F
- Kun desktop — ikke prioriter mobilresponsivitet
- Mørk modus som standard
- Ved kontosletting: fjern/anonymiser Canvas-token, notater, dagslogger og
  planned_sessions helt — ingen myk sletting

## Bygg i denne rekkefølgen (fase 1–2: personlig + venner)
1. Supabase-oppsett + auth (e-post/passord, deretter Google OAuth i testmodus,
   legg deg selv og venner til som test users i Google Cloud Console)
2. Dashboard-skjelett med sidemeny (Dashboard, Emner, Kalender, Karakterer, Alle notater)
3. Canvas-integrasjon (velg universitet + lim inn token i onboarding)
4. Google Calendar-integrasjon
5. Karakterweb-integrasjon (widget i emnekort + institusjons-mapping)
6. AI-anbefaling ("Hva bør jeg jobbe med i dag?")
7. Notatsystem
8. Start dag / Avslutt dag + dagslogg

## Krav KUN ved eventuell offentlig lansering (fase 3 — ikke prioriter nå)
- **Google OAuth-verifisering**: calendar.readonly er et "sensitive scope".
  Uverifisert app er begrenset til 100 brukere totalt. Trengs først når dere
  passerer ~100 brukere eller vil fjerne "usikker app"-advarselen
- **Personvernerklæring**: påkrevd for Google-verifisering ved offentlig lansering
- **E-post til Karakterweb**: informer dem hvis/når dette blir en tjeneste for
  mange studenter utenfor vennekretsen