# Handoff pro Claude Code – Support Q reporting (Yanfeng Planá)

## Cíl
Nasadit hotovou single-file aplikaci **Support Q – reporting hodin** na **Firebase Hosting** (projekt `support-q-working-hours`). Data jsou ve Firebase Realtime Database a přihlašuje se e-mailem a heslem. Uživatel (Milan, Deputy Plant Manager) neumí pracovat s příkazovým řádkem. Proveď ho krok za krokem, česky, nebo kroky udělej za něj.

## Co aplikace dělá
- Externí firma Support Q každý týden (do úterý 12:00) nahraje CSV „Přehled hodin za poskytnuté služby“ z docházky. Sloupce: Datum; Projekt; Popis projektu; Poznámka; Denní hodiny; Noční hodiny; Hodiny celkem; poslední řádek „Sumar“ = kontrolní součet.
- Support Q dále vyplní headcount týdne (operátoři, koordinátoři, projektový manažer, administrativa) a počet lidí na každém projektu.
- **Úvodní obrazovka „Týden po týdnu“:**
  - aktuální CW a stav reportu (nahrán / chybí / po termínu),
  - pruh s verdiktem (zlepšení / zhoršení proti minulému týdnu),
  - 6 dlaždic s porovnáním proti minulému týdnu a sparkline: hodiny, headcount, hodiny na osobu, podíl nočních hodin, víkendové hodiny, projekty s nárůstem >5 %,
  - největší nárůsty a poklesy, graf hodin s čárou headcountu, tabulka projekt × týden (hodiny a lidé).
- **Další sekce:** Detail týdne (po dnech), Měsíc, Objednávky/budget (EUR), Import, Metodika (správa dat, záloha a obnova JSON, uživatelé a přístupy).
- **Report pro Plant Managera:**
  - formátovaný e-mail bezpečný pro Outlook,
  - `.eml` soubor s přílohou Excel (X-Unsent: 1),
  - Excel o 5 listech a CSV,
  - výchozí příjemce milan.hanus@yanfeng.com.
- **Pravidla:**
  - 12h model: norma Ø 42 h na osobu za týden; stavy V MODELU ≤ 48 h, PŘESČAS 48–60 h, OVĚŘIT > 60 h.
  - Porovnání týdnů jen mezi dvěma úplnými týdny (7 dní v datech). Prah LEPŠÍ / HORŠÍ = ±5 %.
  - U všech metrik je nižší hodnota lepší.

## Soubory (balíček `SupportQ_firebase.zip`)
```
public/index.html        ← celá aplikace (HTML + CSS + JS v jednom souboru, ~138 kB)
database.rules.json      ← bezpečnostní pravidla RTDB
firebase.json            ← hosting (public/) + database rules
.firebaserc              ← default project: support-q-working-hours
```
Externí knihovny se načítají za běhu:
- Firebase compat SDK 10.12.2 z `https://www.gstatic.com/firebasejs/10.12.2/` (app, auth, database),
- SheetJS 0.18.5 z cdnjs (jen pro import a export xlsx).

## Firebase konfigurace (už je vložená v `public/index.html`, konstanta `FIREBASE_CONFIG`)
```js
apiKey: "AIzaSyD8IkCALQhub8CLYngEOI9p1sbX6H2PmTY",   // přepsáno ze screenshotu – OVĚŘ proti konzoli (O/0, I/l)
authDomain: "support-q-working-hours.firebaseapp.com",
databaseURL: "https://support-q-working-hours-default-rtdb.europe-west1.firebasedatabase.app",
projectId: "support-q-working-hours",
appId: "1:825200075055:web:855c1c9f7675fef75fc1da",
messagingSenderId: "825200075055",
storageBucket: "support-q-working-hours.firebasestorage.app"
```
Když je `apiKey` prázdný, aplikace běží v lokálním režimu (localStorage). Tuto verzi dostane Support Q jako samostatný soubor HTML, pokud se Firebase nepoužije.

## Stav k 16. 11. 2026
- [x] Firebase projekt založen, Web app zaregistrována, Hosting site propojen
- [x] Authentication → Email/Password zapnuto
- [x] Realtime Database založena v **Belgium (europe-west1)**
- [x] Pravidla z `database.rules.json` vložena v konzoli (ověřit, že je kliknuto **Publish**)
- [ ] Node.js LTS + `npm install -g firebase-tools` + `firebase login`
- [ ] `firebase deploy` ze složky balíčku → https://support-q-working-hours.web.app
- [ ] První přihlášení Milana → registrace → tlačítko **Stát se správcem**
- [ ] Ověřit `apiKey` (při chybě „Neplatný apiKey“ zkopírovat přesnou hodnotu z Project settings → Your apps → Config)
- [ ] Import prvních dat: CSV za CW45 (soubor `prehled-hodin-za-poskytnute-sluzby-01.11.2026-08.11.2026.csv`) a headcount CW45 = 80 / 3 / 1 / 1
- [ ] Pozvat Support Q (role Editor) a kolegy (role Čtenář)

Záložní varianta bez CLI: Netlify Drop (přetáhnout složku `public`). Pak je ale nutné přidat doménu do Firebase → Authentication → Settings → Authorized domains.

## Datový model RTDB
```
/roles/{uid}      = "admin" | "editor" | "viewer"
/users/{uid}      = { email, lastLogin }                 (zapisuje uživatel sám)
/requests/{uid}   = { email, at }                        (žádost o přístup; admin schvaluje v Metodice)
/sq/meta          = { projects, orders[], imports[], cov[], hc{week:{op,ko,pm,ad}}, hcp{week:{projekt:lidí}}, cur:"EUR", pmEmail, demo? }
/sq/months/{YYYY-MM}/rows/{YYYY-MM-DD|PROJEKT} = [denní_h, noční_h]
```
- Týdny jsou ISO: `2026-W45`.
- Ukládání: `sq/meta` přes `set`, měsíce přes `update`.
- Živé změny: posluchač `on('value')` na `/sq`. Když uživatel právě edituje, aktualizace se odloží.

## Pravidla (shrnutí)
- Do `/sq` smí číst každý, kdo má roli. Zapisovat smí jen admin a editor.
- Roli smí měnit jen admin. Jedinou výjimkou je první uživatel, který si smí nastavit `admin`, dokud uzel `/roles` neexistuje.
- `/users` a `/requests`: uživatel zapisuje svůj záznam, admin čte vše.

## Pravidla pro další úpravy (standard Milanových aplikací)
- Single-file HTML bez buildu; knihovny jen z CDN, lazy-load.
- **Firebase RTDB, ne Firestore**, region europe-west1.
- Import: sloupce hledat **podle názvu hlavičky**, nikdy podle pevného indexu.
- Čeština s plnou diakritikou v UI; identifikátory v kódu anglicky.
- Primární barva navy `#003B7A`; stavy zelená / oranžová / červená = good / warn / crit.
- Žádná vymyšlená data (výjimka: ukázková data, výrazně označená a smazatelná jedním tlačítkem).
- Milan chce stručnou komunikaci, česky, 2–3 možnosti s trade-offy a maximálně jednu doplňující otázku najednou.

## Otevřené body / nápady
- Soulad s IT: firemní data jsou na Google účtu mimo Yanfeng IT, potřeba souhlas.
- Možná vlastní doména a automatický týdenní e-mail (Cloud Functions vyžadují tarif Blaze, zatím ne).
- Sazby EUR/h po projektech a kategoriích (OP 16,5 · Senior 18,5 · Admin 20 · Koord 23 · PM 26 · Resident 44 · Rework Gaden 33) zatím nejsou v objednávkách zadané.
