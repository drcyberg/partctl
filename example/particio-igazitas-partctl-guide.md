# Partíció-igazítás, fájlrendszer és teljesítmény — összefoglaló és `partctl.sh` menüútmutató

> **Cél:** Összefoglalni, **mit jelent** a partíció **optimális / nem optimális** létrehozása (pl. **1 MiB** határok, **2048s** / **4096s** kezdő LBA-k, lemez eleje–vége szabad sáv), és **milyen hatása** lehet eznek a **fájlrendszerre** és a **gyakorlati teljesítményre** — **szekvenciális MB/s**, **IOPS** és **késleltetés** szempontjából, **hozzávetőleges előtte/utána** nagyságrendekkel (lásd **§0** és a **§2.3** táblázatok). **HDD**, **SSD**, **RAID** környezetben, **ext4**, **NTFS** és hasonló rendszerek mellett.  
> **Eszköz:** minden lépés a **Partctl** (`bash partctl.sh`) **menüpontjain** keresztül.

**Figyelem:** particiós tábla, partíciók, wipe és formázás **adatvesztést** okozhat. Csak **mentett**, **nem** futó rendszerlemezen, **leválasztott** (unmount) kötetekkel kísérletezz; éles környezetben mindig **biztonsági mentés**.

---

## 0. Gyors áttekintés — teljesítmény **optimalizálás előtt / után** (hozzávetőleges)

![](/img/Hard-Drive-speed.png)

Az alábbi táblázat **nem** konkrét lemezmodell-mérések másolata, hanem **irodalmi / gyakorlati nagyságrendek**: a **„jó igazítás után”** a Partctl alapértelmezett **1 MiB**-hez igazított kezdő LBA + **`parted -a optimal`** vonalat, a **„rossz igazítás előtt”** pedig tipikus **eltolt** (nem 4 KiB / 1 MiB-hoz illeszkedő) partíciókezdő offsetekhez kötött **szintetikus** és **valós** mintákat jelent.

| Terhelés / mutató | Optimalizálás **előtt** (rossz igazítás, tipikus) | Optimalizálás **után** (jó igazítás) | Várható változás (hozzávetőleges) |
|-------------------|--------------------------------------------------|---------------------------------------|-----------------------------------|
| **Szekvenciális olvasás** (MB/s, nagy fájl) | Gyakran a lemez/busz **csúcsának közelében** | Ugyanígy | **≈ 0–3%** vagy **mérési zaj** — sok consumer SSD/HDD-n **nem** kimutatható MB/s különbség. |
| **Szekvenciális írás** (MB/s, nagy fájl, SSD) | Néha **1–5%-kal** alacsonyabb a csúcshoz képest | Stabilabb csúcs | **≈ 0–5%** javulás **lehetséges**, de gyakori a **nulla** eltérés is. |
| **Szekvenciális** (MB/s, HDD) | Fej + sáv dominál; offset másodlagos | Optimalizált offset | **≈ 0–5%** alatti, gyakran **kimérhetetlen** eltérés a szekvenciális MB/s-ben. |
| **Véletlen 4 KiB írás** (IOPS vagy ms késleltetés) | **512e** SSD-n többlet **RMW** → IOPS csökkenhet, késleltetés nőhet | Kevesebb vezérlő-oldali többlet | **IOPS:** tipikusan **≈ 5–25%** jobb a jó igazításnál **szintetikus** 4K random íráson; extrém, régi/rossz párosításnál **akár ~30–40%** is előfordulhat. **MB/s** itt kevésbé értelmes mutató. |
| **RAID + kis blokk** (stripe vs offset) | „Split write” több lemezre | Egy lemezre eső 4K-sáv | **IOPS / késleltetés:** **≈ 10–40%** romlás **rossz** párosításnál nem ritka szintetikus teszteken; extrém geometrián **nagyobb** is lehet. |

**Példa MB/s skálán (csak illusztráció):** ha egy SSD szekvenciális olvasási csúcsa pl. **~500–550 MB/s**, a rossz → jó igazítás váltás a szekvenciális olvasásban gyakran **nem** ad **10–20 MB/s**-nál nagyobb eltérést — inkább **±0–15 MB/s** zaj és **<3%** tartomány. Ahol **érzékelhető** javulás van, ott jellemzően a **kis írások** és az **IOPS**, nem egyetlen nagy **MB/s** szám.

**Hogyan ellenőrizd a saját lemezeden:** ugyanaz a teszt **ugyanazzal** a kötettel, csak partíciós geometria változtatás előtt/után (vagy másolat lemezen) — pl. **`fio`** szekvenciális és **4k randwrite** profilokkal; **fontos:** abszolút MB/s a **modell**, **firmware**, **hőmérséklet** és **tömörítettség** miatt csak önmagadhoz hasonlítható.

---

## 1. Fogalmak — mi az az „igazítás”?

| Fogalom | Rövid magyarázat |
|--------|-------------------|
| **Logikai szektor (LBA)** | A partíciós tábla és az operációs rendszer **szektoronként** számol; tipikus méret ma gyakran **512 bájt** (régebbi és sok **512e** SSD/HDD is így jelent meg). |
| **Fizikai szektor (4 KiB AF)** | Sok modern lemez **4096 bájtos** fizikai blokkon dolgozik („Advanced Format”). Ha a **logikai** 512 B, egy fizikai blokk **8** logikai szektornak felel. |
| **1 MiB igazítás** | **1 MiB = 1024 × 1024 bájt**. **512 B** logikai szektor mellett ez **2048** szektornak felel meg → ezért látjuk gyakran a **`2048s`** kezdő LBA-t jó gyakorlatként. **4096 B** logikai szektor mellett ugyanaz a határ **256** szektor (**256s**). |
| **`parted -a optimal`** | A GNU Parted **igazítási módja**; a Partctl a partíció létrehozásakor ezt használja, hogy a megadott tartomány a lemez és a tábla szabályai szerint **használható** legyen. |

**Fontos:** A **„2048s vagy 4096s”** nem két egymást kizáró „jó” választás ugyanazon lemezen — a **jó** határ a **logikai szektor méret** és a **1 MiB / 4 KiB** igény **közös** eredménye. Ugyanazon lemezen a cél: a partíció **kezdete** és a rajta lévő **fájlrendszer metaadatai** (pl. ext4 szuperblokk, NTFS clusters) **ne essenek fél fizikai blokkokra** hosszú távon.

---

## 2. Ha nem optimálisan hozzuk létre a partíciót (és a fájlrendszert) — mit eredményez?

### 2.1 Negatív hatások (valós, de nem mindig „mérhető nagy MB/s csökkenés”)

1. **SSD (NAND, 512e / 4Kn)**  
   - **Rossz igazítás** esetén egy **4 KiB** (vagy nagyobb) írási egység **két** fizikai programozási egységet is érinthet → többlet **olvasás–módosítás–írás (RMW)** a vezérlőben.  
   - **Következmény:** főleg **kis, véletlenszerű írásoknál** nőhet a késleltetés és a **TBW** (kopás) terhelése; **nagy sorozatos olvasásnál** (szekvenciális MB/s) sok consumer SSD-n a különbség **kicsi vagy elveszik** a másik nyakszűkületek mögött.

2. **HDD**  
   - A **fejmozgás** és a **sáv / szektor** elrendezés miatt a **rossz** (vagy túl finomra nem igazított) elhelyezés **elméletben** ronthatja a hatékonyságot; a gyakorlatban a **fragmentáció** és a **mechanikai** jellegű késleltetés gyakran **dominál** az igazítási hiba felett.  
   - **Nagy sorozatos** folyamoknál a **MB/s** gyakran inkább a **sáv külső/belső** pozíciójától és a forgalom mintájától függ.

3. **RAID / mdadm / hardver RAID**  
   - Ha a **stripe méret** (chunk) és a **partíció / fájlrendszer** kezdő offset **nincs összhangban**, egy alkalmazás szintű **4 KiB / nagyobb** blokk **több lemezes I/O**-ra eshet szét („split write”).  
   - Ez **nem** mindig látszik egyetlen „MB/s” számban: **IOPS** és **késleltetés** romlhat, különösen **kis blokkoknál**.

4. **ext4**  
   - Alapértelmezett **blokk** gyakran **4 KiB**; a partíció elején lévő **szuperblokk + journal** elhelyezkedése **igazított** lemezhez van optimalizálva. Rossz igazítás **extrém** esetben **ritka** edge case-eket adhat; tipikus asztali használatban a **fő rizikó** inkább a **SSD RMW** és a **RAID stripe** együttese.

5. **NTFS**  
   - A **cluster méret** (telepítő / `mkfs.ntfs` beállítás) és a **partíció offset** együtt dönti el, hogy a fájlrendszer belső struktúrái **4 KiB** (vagy nagyobb) határokhoz illeszkednek-e. Rossz párosítás **főleg írásnál** és **kis fájloknál** fájhat.

### 2.2 Pozitív hatások, ha követjük a szabályokat

- **Kisebb** vezérlő- és kernel-oldali **többletmunka** (kevesebb RMW, kiszámíthatóbb I/O).  
- **RAID** alatt **jobb** stripe-egyezés → stabilabb **IOPS** / késleltetés.  
- **Hosszú távú** előny: kevésbé „csúsznak el” a metaadatok a fizikai blokkokhoz képest — különösen **SSD + RAID + adatbázis / VM** esetén érezhető.  
- **Vég-oldali 1 MiB rés (V1.0.0)** — a Partctl alapértelmezésben a **lemez végén is** ~1 MiB szabad helyet hagy a partíció után. Ennek **közvetlen** értéke: **GPT tartalék (backup) fejléc** biztos helye, **LUKS / cryptsetup** fejlécek és **mdadm superblock** stabil pozíciója, valamint kisebb eséllyel ütközik **`sgdisk -v`** „doesn't end on a 32-sector boundary” típusú figyelmeztetésekkel. **MB/s-ban** nem ad mérhető nyereséget — **megbízhatósági** és **eszközkompatibilitási** előny.

### 2.3 Terhelés és egyéb mutatók — mit várjunk reálisan?

| Terhelés típusa | Mit mérünk | Igazítás hatása (tipikus nagyságrend) | Előtte / utána (röviden) |
|-----------------|------------|--------------------------------------|--------------------------|
| **Szekvenciális olvasás** nagy fájlok | **MB/s** közel maximális | Gyakran **minimális** eltérés (más limit: SATA/NVMe, külső busz, CPU). | **≈ 0–3%** vagy zaj — lásd §0. |
| **Szekvenciális írás** | **MB/s** | SSD-n **kicsi** eltérés lehetséges; HDD-n inkább **pozíció** és **sáv** dominál. | **≈ 0–5%** javulás **lehetséges**, gyakran **0**. |
| **Véletlen 4K írás** | IOPS / ms késleltetés | Itt **inkább** látszik az igazítás hiánya (nem feltétlenül „MB/s” mutatóban). | **IOPS:** tipikusan **≈ 5–25%** jobb jó igazításnál; extrémnél **~30–40%**. |
| **RAID + kis blokkok** | IOPS, késleltetés | **Stripe + offset** együtt kritikus; egyetlen **MB/s** szám félrevezető lehet. | **≈ 10–40%** IOPS/késleltetés romlás rossz párosításnál nem ritka. |

**Összegzés:** A **„nem optimális partíció = fix X MB/s kevesebb”** általánosítás **ritkán** igaz egyetlen **X**-szel — **százalékos** nagyságrendek **hozzávetőleges** irányt adnak. A **valós kár** inkább: **többlet írási terhelés**, **ingadozó késleltetés**, **RAID alatti szétcsúszott I/O** — ezeket **benchmark** (pl. `fio`) és **diszk monitor** segítségével érdemes a **saját** lemezen ellenőrizni.

---

## 3. Hogyan segít ebben a Partctl (`partctl.sh`)?

A Partctl a **partíció létrehozás** során:

- **Alapértelmezett kezdő LBA:** a szabad sáv elejét **felkerekíti** a következő **1 MiB** határra a logikai szektor méret alapján (példa: 512 B-nél **2048s** lépésköz).  
- **Alapértelmezett vég LBA:** ha a kiválasztott szabad sáv abszolút vége **`N`** szektor (parted `Ns`), a javasolt alapértelmezés **nem** `Ns`. A logika két lépésből áll: (1) **tail guard**: levonunk egy **1 MiB**-nek megfelelő szektorszámot — **`mib_step`** = `ceil(1 MiB / logikai szektor)` (512 B-nél **2048**, 4096 B-nél **256**); (2) **vég-igazítás**: a végszektort **lefelé `mib_step`-re** (azaz **1 MiB-os határra**) igazítjuk, hogy a `(end + 1)` osztható legyen `mib_step`-pel. Így a `sgdisk -v` „Partition doesn't end on a 2048-sector boundary” figyelmeztetés is elkerülhető, és a következő partíció pontosan a következő 1 MiB-on indulhat. A gyakorlatban a vég `N − 1..2 MiB` környékén esik — pl. **`34s..30842846s`** (14,7 GiB stick) szabad sávon a javaslat **`end = 30838783s`** (tail ≈ 2 MiB), **`2048s..488397167s`** (233 GiB SSD) sávon **`end = 488394751s`** (tail ≈ 1,18 MiB). A **„Vég”** mezőt **kézzel bármikor felülírhatod** (pl. tényleges `Ns` maximumra, ha tudatosan minden szektort fel akarsz használni — ekkor viszont `sgdisk -v` panaszt adhat).
- **`parted`** hívás: **`parted -s -a optimal … mkpart …`** — az **optimal** igazítás a Parted része.  
- A felületen a **„Kezdet”** mezőhöz tartozó szöveg jelzi: **LBA `…s` formátum**, és hogy az alapértelmezés **1 MiB-hoz igazított** a szabad tartományon belül (lásd `hu.json`: `partition_create_param_start_hint`). A **„Vég”** mező súgója (`partition_create_param_end_hint`) szintén utal a **~1 MiB-os vég-réskeretre** a javaslati értéknél.

**Mit jelent a gyakorlatban?** Ha a varázsló a kezdetnél **`2048s`**-t, a végnél pl. **`488394751s`**-t kínál fel egy ~233 GiB-os szabad sávon, akkor a partíció a lemez legutolsó **~1–2 MiB-ját** szabadon hagyja, és a vég pontosan **1 MiB-os határra esik** (`(end+1) mod 2048 = 0` 512 B-nél). Ez tudatos tervezés — **ne** írd át nullára a véget abszolút `free_end`-re, hacsak nem konkrét okod van rá (pl. nem-GPT, nem-LUKS, és minden bájt számít, vállalva a `sgdisk -v` warningot).

**Kiegészítő funkció:** **„Particio igazitas”** / **Partition alignment** — teljes lemezes **újraigazítás** (belsőleg **`sfdisk`**), **csak** akkor engedélyezett, ha **nincs** csatolt kötet, **nincs** LVM jelleg a listában, és a partíciókon **nincs** felismert fájlrendszer (tiszta, üres particiók). Ez **nem** helyettesíti az új partíció **tervezett** létrehozását — de **előkészített** lemezen segíthet.

---

## 4. Közös előkészület — indítás és főmenü

```bash
bash partctl.sh
```

![](/img/terminal_1.jpg)

(A program a **`python3 -m partctl_ncurses_app`** modult indítja (`PYTHONPATH` + `--lang-dir`).)

### Főmenü (rögzített sorszámok — minden nyelven ugyanaz)

| # | Angol | Magyar (`hu.json`) |
|---|--------|---------------------|
| **1** | Select Disk | Lemez kivalasztasa |
| **2** | Disk Overview | Lemez attekintes |
| **3** | Partition management | Particio kezeles |
| **4** | Disk management | Lemez kezeles |
| **5** | Setup | Beallitasok |
| **6** | About | Rolunk |
| **7** | Exit | Kilepes |

![](/img/lemez_kivalasztasa_2.jpg)

**Navigáció:** `Fel` / `Le` (vagy `k` / `j`), **Enter**; vagy a sor elején látható **`N.`** szám begépelése, majd **Enter**. **Vissza:** súgó szerint **Backspace** / **`q`**.

**Particio kezeles** almenü: a tételek **ábécérendbe** vannak rendezve — a pontos **sorszámot** mindig a **képernyőn** ellenőrizd. Az alábbi útmutatóban a **menüpont címkéjét** (angol + magyar) használjuk.

Először mindig: **főmenü → `1`** — **Lemez kivalasztasa** — a listában válaszd ki a **`sdb`** (vagy cél) sort (**sorszám + Enter** vagy kurzor + Enter).

---

## 5. **Új partíció** „jó gyakorlat szerint” (alapértelmezett igazítás elfogadása)

**Cél:** Az első partíció kezdete **1 MiB** határon legyen; a **Particio parameterek** panelen **ne** írjunk be szándékosan „furcsa” kezdő szektort (pl. **1s**, **63s** klasszikus BIOS-offset), hacsak nem tudjuk pontosan, mit csinálunk.

| Lépés | Menüút (rövid) | Mit csinálsz |
|-------|----------------|--------------|
| 1 | **Főmenü → `1`** Select Disk / Lemez kivalasztasa | Kiválasztod a **cél** lemezt (pl. teszt **`sdb`** — **ne** az élő rendszerlemez). |
| 2 | *(Opcionális)* **Főmenü → `4` → `10`** Disk cleanup (Wipe) | „Tiszta lap”: teljes lemez + szükség szerint **partíciós tábla törlés** — részletek: [`win11-gpt-uefi-particio-whitepaper.md`](win11-gpt-uefi-particio-whitepaper.md) §3. |
| 3 | **Főmenü → `3`** → **Create partition table** / **Particios tabla letrehozasa** | **GPT** vagy **MBR** igény szerint; megerősítés, figyelmeztetések elolvasása. |
| 4 | **Főmenü → `3`** → **Create partition** / **Particio letrehozasa** | A **„Kezdet”** mezőnél **Enter** az **alapértelmezett** (1 MiB-hoz igazított) értékre; **„Vég”**-nél pl. **`+100GiB`**, **`100%`**, vagy abszolút **`…s`** — a súgó szerint. |
| 5 | **Főmenü → `3`** → **Partition format** / **Particio formazas** | **ext4** / **ntfs** / stb. a cél szerint — a fájlrendszer **most** jön létre **már igazított** partíción. |
| 6 | **Főmenü → `2`** Disk Overview / Lemez attekintes | Ellenőrzés: partíció **kezdete** (szektor), méret, tábla típusa. |

| Kezdet | Vég |
|--------|-----|
| ![particio_letrehozasa_2](/img/particio_letrehozasa_2.jpg "Partício létrehozása #2") | ![particio_letrehozasa_3](/img/particio_letrehozasa_3.jpg "Partício létrehozása #3") |

- **Tanulság:** Ha a **4.** lépésben **módosítod** a kezdő szektort **szándékosan** (pl. kihagysz **2048** helyett csak **32** szektort), a Parted **`optimal`** módja megpróbálja a **biztonságos** tartományba terelni — de a **végső** geometria mindig a **megadott** és a **lemez** korlátok együttes eredménye.
- **Jó gyakorlat:** bízd a **javasolt kezdő és vég** értéket a programra.

---

## 6. **Ellenőrzés** és **„rossz” geometria** felismerése

| Lépés | Menüút | Mit nézel |
|-------|--------|-----------|
| 1 | **Főmenü → `1`** | Céllemez kiválasztva. |
| 2 | **Főmenü → `2`** Lemez attekintes | Partíciós sorok: **kezdő szektor**, **vége**, típus. Ha a **kezdő** nem **2048** többszöröse 512 B-nél (vagy nem illik a **1 MiB** szabályhoz a lemez logikai szektorához), érdemes megfontolni az **újraparticionálást** (adatvesztéssel jár). |
| 3 | *(Linux CLI, a Partctl mellett)* | `parted -m /dev/sdX unit s print` — részletes **szektor** nézet; `lsblk -o NAME,SIZE,TYPE,FSTYPE,PHY-SEC,LOG-SEC` — **logikai / fizikai** szektor jelzés (ha elérhető). |

![gpt_tabla_ellenorzes_2](/img/gpt_tabla_ellenorzes_2.jpg "GPT tábla ellenőrzés #2")

---

## 7. **Particio igazitas** / **Partition alignment**

**Előfeltételek (a program is blokkolja, ha nem teljesülnek):**

- **`sfdisk`** elérhető a rendszeren.  
- **Nincs** csatolt fájlrendszer a lemez partícióin; **nincs** LVM jelleg a listában.  
- A partíciókon **nincs** felismert fájlrendszer (üres particiók) — a varázsló **biztonsági** okból nem fut le „éles” FS mellett.

| Lépés | Menüút | Megjegyzés |
|-------|--------|------------|
| 1 | **Főmenü → `1`** | Céllemez. |
| 2 | **Főmenü → `3`** → **Particio igazitas** / **Partition alignment** | A belső cél: **4 KiB** (alapértelmezett konfiguráció) határhoz igazítás tipikus **512 B** logikai szektor mellett. |
| 3 | Kövesd a **megerősítő** párbeszédeket | **Adatvesztés-mentes** újraigazítás **nem** garantálható minden geometrián — mindig olvasd el a figyelmeztetést. |
| 4 | **Főmenü → `2`** | Eredmény ellenőrzése. |

| Partíció igazítás | Igazítási előnézet | Igazított |
|-------------------|--------------------|-----------|
| ![particio_igazitas_1](/img/particio_igazitas_1.jpg "Partíció igazítás #1") | ![particio_igazitas_1](/img/particio_igazitas_2.jpg "Partíció igazítás #2") | ![gpt_tabla_ellenorzes_3](/img/gpt_tabla_ellenorzes_3.jpg "GPT tábla ellenőrzés #3") |

Ha már **formázott** partícióid vannak és csak „javítani” szeretnél: ez a varázsló **nem** erre való — ilyenkor **mentés**, **újraparticionálás**, majd **5.** szerinti **formázás** a helyes út.

---

Partíció-igazítás, fájlrendszer és teljesítmény — háttér és `partctl.sh` menüútmutató
```markdown
https://github.com/drcyberg/partctl/blob/main/example/particio-igazitas-partctl-guide.md
```

### Fő oldal (Partctl)

- [Partctl](https://drcyberg.github.io/partctl/web/partctl)

### Köszönöm ha támogatsz

- ***Buy me a coffee***: [LINK](https://buymeacoffee.com/drcyberg)
- ***Paypal***: [LINK](https://github.com/drcyberg/partctl/blob/main/img/qrcode.png)

---

*Utolsó frissítés jelleg: Partctl V1.0.0 viselkedés (`partctl.sh` → `partctl_ncurses_app`) — a **Particio kezeles** lista ábécérendje miatt a konkrét **sorszámok** mindig a futó programban ellenőrizendők.*
