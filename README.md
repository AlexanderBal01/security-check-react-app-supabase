# Security-checklist React + Supabase

Bij een React-app met Supabase praat de browser rechtstreeks met de database, zonder eigen backend ertussen. De beveiliging zit dus vooral in Postgres zelf: Row Level Security, keys en functies. Vink elk punt af voor een release en herhaal de controle na elke databasemigratie. Vaktermen staan uitgelegd in de [woordenlijst](#woordenlijst) onderaan.

## 1. Row Level Security (RLS)

RLS is de belangrijkste beveiliging in Supabase. Zonder RLS kan iedereen met de publieke key je volledige tabel lezen.

- [ ] RLS staat aan op elke tabel in het `public` schema (`alter table ... enable row level security;`)
- [ ] Elke tabel heeft een aparte policy per actie die nodig is: `SELECT`, `INSERT`, `UPDATE`, `DELETE`
- [ ] Policies gebruiken `auth.uid()` om de gebruiker te bepalen, nooit een `user_id` die de client meestuurt
- [ ] Elke `UPDATE`-policy heeft naast `using` ook `with check`, zodat niemand `user_id` of eigenaar kan wijzigen
- [ ] Rollen (admin enz.) staan in een aparte tabel of in `app_metadata`, niet in `user_metadata` (die kan de gebruiker zelf aanpassen)
- [ ] Policies getest met twee gebruikers: gebruiker A kan de data van B niet lezen of wijzigen (IDOR)
- [ ] Policies getest zonder login (alleen de anon key)
- [ ] De Security Advisor in het dashboard toont geen waarschuwingen

## 2. Keys

De publieke key mag in de frontend, de geheime key nooit: die omzeilt RLS volledig.

- [ ] De frontend gebruikt alleen de anon/publishable key (`sb_publishable_…`)
- [ ] De service_role/secret key (`sb_secret_…`) staat nergens in de React-code
- [ ] Geen geheime key in een `VITE_`- of `REACT_APP_`-variabele, want die komen in de JavaScript-bundle terecht
- [ ] Geen keys of `.env`-bestanden in git (`.env` staat in `.gitignore`)
- [ ] De secret key wordt alleen server-side gebruikt (Edge Functions, CI)
- [ ] Er is een plan om keys te roteren als er toch een lekt

## 3. Database-functies, views en schema's

Functies en views kunnen RLS ongemerkt omzeilen.

- [ ] `security definer`-functies alleen waar echt nodig, met een vaste `search_path` en eigen rechtencontrole in de functie
- [ ] Functies die niet publiek mogen zijn: `revoke execute` voor `anon` (en eventueel `authenticated`)
- [ ] Views aangemaakt met `with (security_invoker = true)`
- [ ] Interne tabellen staan in een apart schema dat niet via de API exposed is
- [ ] Triggers en database-functies valideren input zelf, niet alleen de frontend

## 4. Authenticatie

Login regelt Supabase Auth, maar de instellingen bepaal je zelf.

- [ ] E-mailbevestiging staat aan
- [ ] Redirect URLs in de Auth-instellingen staan beperkt tot je eigen domeinen
- [ ] Ongebruikte loginproviders (Google, GitHub, …) staan uit
- [ ] Minimale wachtwoordsterkte ingesteld en, waar mogelijk, bescherming tegen gelekte wachtwoorden aan
- [ ] MFA aangeboden aan gebruikers en verplicht voor admins
- [ ] Rate limits op login en e-mails nagekeken (tegen brute force en spam)
- [ ] Protected routes in React gelden alleen als UX, de echte controle zit in RLS

## 5. Storage (bestanden)

Buckets hebben hun eigen policies, los van je tabellen.

- [ ] Policies op `storage.objects` voor elke bucket: wie mag uploaden, lezen, verwijderen
- [ ] Een bucket is alleen public als iedereen de bestanden mag zien
- [ ] Privébestanden worden gedeeld via signed URLs met een korte geldigheid
- [ ] Toegelaten bestandstypes (MIME) en maximale grootte ingesteld per bucket
- [ ] Bestandspaden bevatten de gebruiker (bv. `<user_id>/bestand.pdf`) en de policy controleert dat

## 6. Edge Functions

Edge Functions zijn je enige echte backend, dus ze krijgen dezelfde controles als een API.

- [ ] Elke functie controleert de JWT van de gebruiker (tenzij bewust publiek, zoals een webhook)
- [ ] Webhooks controleren de handtekening van de afzender (Stripe, Resend, …)
- [ ] Alle input gevalideerd, bv. met Zod
- [ ] API-keys van derden staan in Supabase secrets, niet in code of frontend
- [ ] Foutmeldingen naar de client bevatten geen stack traces of interne details
- [ ] CORS-headers staan alleen je eigen domein(en) toe

## 7. React-frontend

De sessie staat standaard in `localStorage`, dus XSS voorkomen is extra belangrijk.

- [ ] Geen `dangerouslySetInnerHTML`, of alleen met DOMPurify ervoor
- [ ] Geen gebruikersinput in `href` zonder controle (blokkeer `javascript:`-links)
- [ ] Content Security Policy (CSP) en andere security headers ingesteld op de hosting (Vercel, Netlify, …)
- [ ] Overal HTTPS
- [ ] Geen gevoelige data of logica die enkel "verborgen" is in de frontend
- [ ] Source maps niet publiek in productie (of bewust gekozen)

## 8. Operationeel, dependencies en logging

Deze punten gaan over het project en het team, niet over de code.

- [ ] MFA aan op alle Supabase-accounts, en op GitHub als je daarmee inlogt
- [ ] Minstens twee eigenaars in de Supabase-organisatie
- [ ] Aparte projecten voor dev en productie, zonder echte klantdata in dev (GDPR)
- [ ] Backups gecontroleerd en een herstel eens getest (point-in-time recovery waar nodig)
- [ ] Dependencies gescand met `npm audit` of Dependabot
- [ ] API-, Auth- en Postgres-logs in het dashboard regelmatig bekeken
- [ ] Geen wachtwoorden, tokens of persoonsgegevens in logs

## 9. Zelftest

Probeer je eigen app aan te vallen zoals een buitenstaander dat zou doen.

1. Zoek tabellen zonder RLS in de SQL-editor:

   ```sql
   select tablename from pg_tables
   where schemaname = 'public' and rowsecurity = false;
   ```

2. Open de devtools van je app en kopieer de project-URL en de anon key.
3. Probeer zonder login een tabel te lezen:

   ```bash
   curl 'https://<project>.supabase.co/rest/v1/<tabel>?select=*' \
     -H "apikey: <anon key>"
   ```

4. Probeer hetzelfde met een `insert`, `update` en `delete`.
5. Log in als gebruiker A en probeer data van gebruiker B te lezen of te wijzigen.
6. Probeer als gewone gebruiker een admin-RPC aan te roepen of je eigen rol aan te passen.
7. Draai de Security Advisor en los elke waarschuwing op.

Lukt een van deze aanvallen, dan klopt de policy of het recht op die plek niet.

## Woordenlijst

| Term | Betekenis |
| --- | --- |
| Anon/publishable key | Publieke Supabase-key voor de frontend; geeft alleen toegang binnen de RLS-regels |
| Attack surface | Alles wat een aanvaller kan aanspreken: endpoints, formulieren, uploads |
| Authenticatie | Vaststellen wie iemand is (login) |
| Autorisatie | Bepalen wat iemand mag doen (rechten) |
| Brute force | Eindeloos wachtwoorden proberen tot er een werkt |
| Bucket | Map voor bestanden in Supabase Storage |
| CORS | Regels die bepalen welke andere domeinen je API vanuit de browser mogen aanroepen |
| CSP (Content Security Policy) | Header met een whitelist van bronnen waaruit scripts en stijlen mogen laden |
| CVE | Publiek nummer van een gekend beveiligingslek in software |
| Edge Function | Serverless functie die op Supabase draait; je eigen stukje backend |
| Exposed schema | Databaseschema dat via de Supabase-API bereikbaar is |
| GDPR/AVG | Europese privacywetgeving over persoonsgegevens |
| IDOR | Lek waarbij je door een id te wijzigen data van een ander kan zien of aanpassen |
| JWT (JSON Web Token) | Ondertekend token dat bewijst wie de gebruiker is; inhoud is leesbaar voor iedereen |
| Least privilege | Elk account krijgt alleen de rechten die het echt nodig heeft |
| MFA | Tweede inlogfactor naast het wachtwoord, bv. een code in een app |
| Policy | Een RLS-regel op een tabel voor een bepaalde actie |
| Rate limiting | Beperken hoeveel verzoeken iemand per tijdseenheid mag doen |
| RLS (Row Level Security) | Postgres-regels die per rij bepalen wie die mag lezen of wijzigen |
| RPC | Een Postgres-functie die je aanroept via `supabase.rpc()` |
| `security definer` | Functie die draait met de rechten van de eigenaar, en dus RLS omzeilt |
| `security_invoker` | Optie waardoor een view de rechten en RLS van de aanroeper gebruikt |
| Service_role/secret key | Geheime Supabase-key die RLS volledig omzeilt; alleen server-side |
| Signed URL | Tijdelijke link naar een privébestand die na een ingestelde tijd vervalt |
| Supply chain attack | Aanval via een kwaadaardige of gekaapte dependency |
| XSS | Aanvaller injecteert JavaScript in je pagina dat bij andere gebruikers draait |

## Bronnen en verder lezen

- [Supabase Production security checklist](https://supabase.com/docs/guides/security/production-checklist) — officiële checklist voor database, account en Auth
- [supabase/splinter](https://github.com/supabase/splinter) — de regels achter de Security Advisor
- [OWASP ASVS](https://github.com/OWASP/ASVS) — standaard met afvinkbare eisen per niveau; niveau 1 als basis voor de React-app
- [OWASP Cheat Sheet Series](https://github.com/OWASP/CheatSheetSeries) — korte gidsen per onderwerp (XSS, sessies, autorisatie, …)
- [0xRadi/OWASP-Web-Checklist](https://github.com/0xRadi/OWASP-Web-Checklist) — checklist vanuit pentest-oogpunt
