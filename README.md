# dev-tgf-gyakorlat — `aquavagyok`

Fejlesztő felvétel, **gyakorlati fázis**.

Szia **aqua**! 👋 Ez a te személyre szabott feladatod. Ne válts vissza a `main` ágra, minden itt történik.

---

## 0. Miért pont ezt a feladatot kaptad?

A jelentkezési lapod alapján állítottuk össze. Nem véletlenszerű feladat, hanem **pont azokra a pontokra kérdez rá, amiket magadról leírtál**:

| Amit a lapon írtál | Amit a feladat ebből ellenőriz |
| --- | --- |
| „Java – **profi**” | Valódi, több osztályos, párhuzamosságot is kezelő Java kód. A „profi” szint nálunk konkrét elvárásokat jelent, ezeket lentebb kifejtjük. |
| „**Nem szoktam használni verziókezelőket**” | A feladat kötelezően Git-alapú. Ezt most meg kell tanulnod — 1-2 óra alatt megvan, és nálunk ez alapfeltétel. |
| „Korábban volt GitHub profilom… de **elvesztek**” | Nincs mit megnézni tőled, ezért a **munkafolyamatot** akarjuk látni, nem csak a végeredményt. Ezért számítanak a commitok. |
| „**3 éve** foglalkozom szerverfejlesztéssel” | 3 év alatt egy dupe-biztos tranzakciónak és az async I/O-nak rutinnak kell lennie. Ezt méri a feladat magja. |
| Dupe-hibás szituációs kérdés (reprodukálás → log → hotfix → teszt → deploy) | Szép válasz volt. Most **kódban** kell megcsinálnod ugyanezt: a feladat egy klasszikus dupe-felületet ad a kezedbe. |
| TPS drop kérdés („optimalizált alternatívát javaslok”) | A feladatban van egy szándékosan drága művelet. Nem elég elmondani, meg is kell oldanod. |
| „MySQL, SQLite” | Mindkettőt támogatnia kell a pluginnak, ugyanazzal a kóddal. |
| „Spigot, Paper, Purpur — eddig csak ezeket” | **Nem** kérünk NMS-t, BungeeCordot vagy Velocityt. Tiszta Paper API. Nem buktatni akarunk. |
| „Angol: **B2**” | A kód, a Javadoc és a commit üzenetek angolul. |
| „Heti **5-6, max 7 óra**” | A feladat kb. **8-10 órányi** munka, ezért **10 nap** a határidő. Nem sietünk, nem kell éjszakázni. |
| „Van jó minőségű mikrofonom: **Igen**” | A leadás után egy **15 perces hangos védés** lesz Discordon, képernyőmegosztással. |

> Kor: 17 év. Ez nálunk **nem kizáró ok**, és a feladat értékelésénél sem számít. A kód számít.

---

## 1. A feladat: `TgfMarket` — játékosok közötti piac plugin

Készíts egy **Paper** plugint, amivel a játékosok itemeket tudnak eladásra kínálni, mások pedig megvenni azokat.

### Alap környezet

- **Paper 1.21.4**, **Java 21**
- Build: **Gradle** vagy **Maven** (te választasz, de a `./gradlew build` / `mvn package` fusson le tisztán)
- Külső könyvtár engedélyezett (pl. HikariCP), de **minden függőséget indokolj** a `DECISIONS.md`-ben

### Funkciók

**`/market sell <ár>`**
- A kézben tartott item feltöltése eladásra a megadott áron.
- Az item **azonnal kikerül** a játékos inventoryjából.
- Üres kéz, nem pozitív ár, ár-limiten túli érték → érthető hibaüzenet, semmi történés.
- Egy játékos egyszerre max. **8** aktív hirdetést tarthat.

**`/market`**
- GUI (chest inventory), **lapozható**, oldalanként 45 hirdetés.
- Minden itemen lore-ban: eladó neve, ár, feltöltés ideje.
- Bal klikk = vásárlás. **Minden más klikktípus (shift-klikk, középső klikk, number key, drag, dupla klikk) blokkolva.**
- A vásárló GUI-ja **nem** frissülhet úgy, hogy közben más játékos vásárlása miatt elcsúszik és mást vesz meg, mint amire kattintott.

**`/market cancel`**
- A saját hirdetések visszavonása, item vissza az inventoryba.
- Ha tele az inventory: **ne dobd el a földre**, hanem tagadd meg a műveletet érthető üzenettel.

**Egyenleg**
- Saját, egyszerű balance kezelés az adatbázisban (nincs Vault dependency).
- `/market balance`, és egy admin `/market give <player> <összeg>` parancs a teszteléshez.

**Adatbázis**
- A `config.yml`-ből választható: `sqlite` vagy `mysql`. **Ugyanaz a kód mindkettőn menjen.**
- Séma automatikus létrehozása induláskor.
- Szerver újraindítás után minden hirdetés és egyenleg megmarad.

---

## 2. A feladat lényege — erre megy a pontszám

A funkciólista a könnyebbik fele. A valódi kérdés ez a három:

### 🔴 A) Dupe- és pénznyomtatás-mentesség

Ez a feladat magja. Legalább ezekre a helyzetekre **működő védelemmel** kell rendelkezned:

1. **Két játékos** kattint ugyanarra a hirdetésre **ugyanabban a tickben** → pontosan az egyikük kapja meg az itemet, a másik hibaüzenetet kap, és **nem vonódik le a pénze**.
2. Az eladó `/market cancel`-t nyom, miközben valaki épp veszi az itemet → az item **egy példányban** létezik utána, nem nullában és nem kettőben.
3. A vásárló **kilép a szerverről** a tranzakció közben → nincs eltűnő item és nincs eltűnő pénz.
4. A vásárló inventoryja **tele van** → a vásárlás nem történik meg, a pénz nem vonódik le, a hirdetés marad.
5. A GUI-ban a hirdetés **eltávolítása és a pénzlevonás nem szakadhat szét** — ha az egyik meghiúsul, a másiknak sem szabad megtörténnie.

A `DECISIONS.md`-ben pontosan írd le, **melyik mechanizmussal** (DB tranzakció, feltételes `UPDATE ... WHERE`, lock, single-threaded executor, stb.) oldottad meg és **miért azt választottad**.

### 🔴 B) A main thread nem blokkolhat

- **Egyetlen** adatbázis-hívás sem futhat a szerver fő szálán. Se olvasás, se írás.
- A Bukkit API-t (inventory, item adás, üzenetküldés) viszont **csak** a fő szálon hívhatod.
- A kettő közötti átjárást neked kell helyesen megoldanod.
- A GUI megnyitása ne akassza meg a szervert 500 hirdetésnél sem.

Ha bárhol találunk egy `Statement.executeQuery()`-t a main threaden, az a feladat érdemi bukása — ez a leggyakoribb TPS drop ok éles szervereken, és a lapon te magad hoztad fel a témát.

### 🔴 C) SQL injection

Minden lekérdezés `PreparedStatement`, paraméterezve. Egyetlen string-konkatenált SQL sem maradhat a kódban. Egy `Aqua'; DROP TABLE listings;--` nevű játékos ne tudjon kárt okozni.

---

## 3. Git — ez nem opcionális rész

A lapon azt írtad, nem használsz verziókezelőt. **Ez a feladat egyik célja, hogy ezt behozd.** Elvárás:

- Dolgozz **ezen a `aquavagyok` branchen**, ide pusholj.
- **Minimum 8 értelmes commit**, a munka menetét követve.
  - ❌ Egyetlen `"kesz"` / `"final"` commit a végén, amiben ott az egész projekt → **automatikus elutasítás.** Pont azt nem látjuk belőle, amiért kérjük.
  - ✅ Egy commit = egy logikailag összetartozó változás.
- Commit üzenetek **angolul**, jelen idejű felszólító módban:
  - `add sqlite storage backend`
  - `fix race condition on concurrent purchase`
  - `move listing query off the main thread`
- **`.gitignore` kötelező.** Nem kerülhet be a repóba: `build/`, `target/`, `*.jar`, `.idea/`, `.gradle/`, `run/`, `logs/`, `*.db`, `config` valós jelszóval.
- Ha véletlenül bekerül egy jelszó vagy egy `.jar`, azt **külön committal vedd ki**, és említsd meg a `DECISIONS.md`-ben. Ez nem hiba, ez normális tanulási folyamat — az eltitkolása lenne a hiba.

Ha még sosem használtad, ez a három parancs a lényeg: `git add <fájl>`, `git commit -m "üzenet"`, `git push`. Nyugodtan kérdezz rá Discordon, ha elakadsz — a kérdezés itt nem levonás.

---

## 4. Leadandó fájlok

```
├── .gitignore
├── README.md              ← ezt NE írd felül, hagyd meg
├── SETUP.md               ← te írod: hogyan buildeljem és próbáljam ki
├── DECISIONS.md           ← te írod: lásd lentebb
├── build.gradle(.kts) vagy pom.xml
└── src/main/java/...      ← a plugin
    └── resources/
        ├── plugin.yml
        └── config.yml
```

### `SETUP.md`
Rövid, lényegre törő: build parancs, hova kell a jar, mit kell beállítani a configban, milyen parancsokkal tudjuk kipróbálni a fő folyamatot lépésről lépésre.

### `DECISIONS.md`
Ez a legfontosabb szöveges rész. Magyarul is lehet. Tartalmazza:

1. **Hogyan oldottad meg a 2. A) pont 5 dupe-forgatókönyvét?** Forgatókönyvenként 2-3 mondat.
2. **Hogyan biztosítod, hogy nincs DB hívás a main threaden?**
3. **Mit hagytál ki, és miért?** Ha valamire nem futotta az időből, azt írd le őszintén. Egy tudatosan vállalt hiányosság nálunk **többet ér**, mint egy elrejtett bug.
4. **AI-használat.** Ha használtál ChatGPT-t / Copilotot / bármit, írd le, mihez. **Ez nem tiltott és nem levonás** — mi is használunk. A védésen viszont **minden sort el kell tudnod magyarázni**, amit beadtál. Ha ott kiderül, hogy egy általad írt kódot nem értesz, az sokkal rosszabbul néz ki, mint ha itt őszintén jelzed.

---

## 5. Amit NEM kérünk

Hogy tudd, mire ne menjen az időd:

- ❌ NMS / reflection / version-specific hackek
- ❌ BungeeCord / Velocity / cross-server szinkron
- ❌ Vault, PlaceholderAPI vagy bármilyen soft-depend integráció
- ❌ Szép textúrák, custom modellek, Adventure-alapú látványvilág
- ❌ Unit tesztek (jó pont, ha van, de nem elvárás)
- ❌ Aukció, licit, adórendszer, kategóriák, kereső — **ne bővítsd a scope-ot**

A heti 5-7 órád reális, ehhez szabtuk a méretet. Egy **működő, dupe-mentes, szűk** plugin sokkal többet ér nálunk, mint egy funkciógazdag, de lyukas.

---

## 6. Értékelés

| Terület | Súly |
| --- | --- |
| Dupe- és race condition védelem (2/A) | **35%** |
| Async / main thread helyes kezelése (2/B) | **20%** |
| Kódminőség, struktúra, olvashatóság | 15% |
| DB réteg: SQLite+MySQL, PreparedStatement (2/C) | 10% |
| Git-használat, commit-történet | 10% |
| `DECISIONS.md` és `SETUP.md` minősége | 10% |

**Azonnali elutasítás:** másolt plugin-kód forrásmegjelölés nélkül · egyetlen dump-commit · a védésen nem tudod elmagyarázni a saját kódod.

---

## 7. Határidő és leadás

- **Határidő: a branch létrehozásától számított 10 nap.** A pontos dátumot Discordon megkapod.
- Leadás: pusholj erre a branchre, majd írj a Discordon, hogy kész vagy.
- **Védés:** utána egyeztetünk egy ~15 perces hangos hívást képernyőmegosztással. Végigveszünk pár kérdést a kódodból, és élőben megnézünk egy-két dupe-forgatókönyvet.
- Ha csúsznál, **szólj előre**. A csúszás önmagában nem probléma, a néma eltűnés az.

---

## 8. Ha elakadsz

Kérdezz. Komolyan. Egy jó kérdés Discordon **nem levonás** — nálunk a valós munka is így megy. Amiben szívesen segítünk:

- Git-alapok, ha most használod először
- Paper API / Gradle-Maven setup nyűgök
- A feladat szövegének értelmezése

Amiben nem: a konkrét concurrency-megoldás kitalálása helyetted. Az a feladat lényege.

Sok sikert! 💧
