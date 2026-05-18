# GPT partíciós tábla mentés, helyreállítás és ellenőrzés — `partctl.sh` menüútmutató és esettanulmány

**Kinek szól:** Linuxon lemezt és partíciót kezelő rendszergazdáknak, üzemeltetőknek és haladó felhasználóknak, akik **GPT** lemezen szeretnék **biztonsági menteni**, **ellenőrizni** vagy **visszaállítani** a partíciós tábla metaadatait a **Partctl** (`bash partctl.sh`) menüin keresztül — anélkül, hogy kézzel kellene `sgdisk` parancsokat memorizálni.

> **Cél:** Lépésről lépésre bemutatni a **Create GPT backup**, **Restore GPT backup** és **Verify GPT partition table** funkciókat; rövid **esettanulmányokkal** illusztrálni, mikor érdemes használni őket.  
> **Eszköz:** minden művelet a Partctl **Partíció kezelés** almenüjéből indul; a háttérben az **`sgdisk`** fut.

**Figyelem:** A GPT mentés **nem** fájlrendszer-, vagy fájladat mentés. A partíciók **tartalma** (NTFS, ext4, stb.) **nem** kerül lementésre. Kockázatos műveletek (tábla újraírás, wipe, helyreállítás) **adatvesztést** okozhatnak, ha a layout és a tényleges adatok nincsenek összhangban. Mindig legyen **teljes adatmentés**, ha az adatokra szükség van.

---

## 0. Gyors áttekintés — mit csinál a Partctl?

<p align="center">
  <img src="/img/gpt_tabla_mentes_helyreallitas_resized.png" alt="GPT partíciós tábla — mentés, helyreállítás, ellenőrzés (Partctl / sgdisk)" width="100%" />
</p>

| Művelet (angol menü) | Magyar menü | Háttérparancs (tipikus) | Mit ment / állít vissza |
|----------------------|-------------|-------------------------|-------------------------|
| **Create GPT backup** | **GPT tabla mentes** | `sgdisk --backup=… /dev/sdX` | A **GPT fejléc + partíciós bejegyzések** bináris másolata (`.bin`) |
| **Restore GPT backup** | **GPT tabla helyreallitas** | `sgdisk --load-backup=… /dev/sdX` | Felülírja a lemez **jelenlegi GPT metaadatait** a mentésből |
| **Verify GPT partition table** | **GPT tabla ellenorzes** | `sgdisk -v /dev/sdX` | **Ellenőrzi** a táblát (elsődleges/másodlagos fejléc, határok, figyelmeztetések) |

**Mentés fájl helye:** Partctl a saját könyvtárában automatikusan létrejön a **`backup/`** mappa. A fájlnév mintája:

```text
backup/{lemez}-gpt-backup-{YYYYMMDD}-{NNN}.bin
```

Példa: `backup/sda-gpt-backup-20260518-001.bin` — ugyanazon napon a következő mentés `002`, `003`, …

> Megjegyzés: **MBR** (`msdos`) lemezen ezek a menük **nem** érhetők el (hibaüzenet: csak GPT tábla). MBR ellenőrzéshez a Partctl külön menüpontot ad: **Verify MBR partition table** / **MBR tabla ellenorzes**.

---

## 1. Háttér — mi van a `.bin` fájlban?

A **GUID Partition Table (GPT)** két példányban tárolja a metaadatokat a lemezen:

- **Elsődleges GPT fejléc** — a lemez **elején**
- **Másodlagos (backup) GPT fejléc** — a lemez **végén**

Az `sgdisk --backup` ezt a **layout információt** menti: partíciók kezdete/vége, típus-GUID, nevek, attribútumok — **nem** a partíciók belsejében lévő fájlokat.

| Mentés tartalma | Nincs benne |
|-----------------|-------------|
| Partícióhatárok, GUID típusok, GPT nevek | Fájlok, könyvtárak, NTFS/ext4 tartalom |
| Javítási alap **hibás GPT fejléc** esetén | Helyettesítő **teljes lemezklón** |

**Mikor hasznos:**

- **Kockázatos szerkesztés előtt** (új partíció, törlés, átméretezés, táblamódosítás)
- **„Elrontott GPT fejléc”** helyreállítása, ha az adatpartíciók fizikailag épültek
- **Ugyanarra a lemezre** visszaállítani egy **ismert jó** layoutot (pl. tesztkörnyezet ismétlése)

**Mikor nem elég:**

- Fájlok visszaállítása törés után → **fájlrendszer-szintű** mentés kell
- Lemez másik gépre „klónozása” adatokkal együtt → **blokk-szintű** másolás (`dd`, `clonezilla`, stb.)

---

## 2. Előfeltételek

| Követelmény | Megjegyzés |
|-------------|------------|
| **Root / sudo** | `bash partctl.sh` rendszergazdai joggal |
| **`sgdisk`** (csomag: `gdisk`) | A `setup.sh` **Ellenőrzés** menüje jelzi, ha hiányzik |
| **GPT lemez** | **Lemez áttekintés** / `sgdisk -l` — `Partition table: gpt` |
| **Céllemez azonosítása** | Pl. USB: `lsblk`, modell, méret — **soha** ne téveszd össze a rendszerlemezzel |
| **Írási jog a `backup/` mappába** | A Partctl a **aktuális munkakönyvtárból** indul (ahonnan `bash partctl.sh`-t futtatod) |

**Ajánlott:** kockázatos művelet előtt **csatold le** (unmount) az érintett partíciókat. A **helyreállítás** varázsló a lemezt **detach** ellenőrzéssel indítja (`ensure_target_detached`).

---

## 3. Közös előkészület — indítás és navigáció

```bash
sudo bash partctl.sh
```

![](/img/terminal_1.jpg)

### Főmenü (rögzített sorszámok)

| # | Angol | Magyar |
|---|--------|--------|
| **1** | Select Disk | Lemez kivalasztasa |
| **2** | Disk Overview | Lemez attekintes |
| **3** | Partition management | Particio kezeles |
| **4** | Disk management | Lemez kezeles |
| **5** | Setup | Beallitasok |
| **6** | About | Rolunk |
| **7** | Exit | Kilepes |

![](/img/lemez_kivalasztasa_2.jpg)

**Lépések minden esettanulmányhoz:**

1. **Főmenü → `1`** — válaszd ki a **céllemezt** (pl. `sda` — USB pendrive).
2. **Főmenü → `3`** — **Partíció kezelés** / **Partition management**.

### Partíció kezelés almenü — fontos

Az almenü tételei **betűrendben** jelennek meg (nyelvfüggő sorrend). A **számozás ezért változhat** — mindig a **szöveges menünevet** keresd, ne fix számot.

| Angol menüpont | Magyar menüpont |
|----------------|-----------------|
| Create GPT backup | GPT tabla mentes |
| Restore GPT backup | GPT tabla helyreallitas |
| Verify GPT partition table | GPT tabla ellenorzes |

**Navigáció:** Fel/Le, PgUp/PgDn, Home/End, szám + Enter, **Backspace/Q** = vissza.

---

## 4. Lépésről lépésre — GPT partíciós tábla mentés (Create GPT backup)

### 4.1 Menüút

| Lépés | Hol | Tevékenység |
|-------|-----|-------------|
| 1 | Főmenü → **`1`** | Lemez kiválasztása (pl. `sda`) |
| 2 | Főmenü → **`3`** | Partíció kezelés |
| 3 | Almenü | **GPT tabla mentes** / **Create GPT backup** |
| 4 | Megerősítő panel | **Igen** — a panel mutatja a lemezt és a cél `.bin` útvonalat |
| 5 | Folyamat panel | `sgdisk --backup=…` fut (GPT művelet folyamatban) |
| 6 | Info panel | Siker: elmentett fájl teljes útvonala |

| Megerősítés | Információ |
| --- | --- |
| ![gpt_tabla_mentes_1](/img/gpt_tabla_mentes_1.jpg "GPT partíciós tábla mentés #1") | ![gpt_tabla_mentes_2](/img/gpt_tabla_mentes_2.jpg "GPT partíciós tábla mentés #2") |

### 4.2 Mit látsz a megerősítésnél?

A program **automatikusan** generálja az útvonalat — nem kell kézzel beírni:

```text
Elkeszitsem most a GPT biztonsagi mentest?
Lemez: /dev/sda
Fajl: …/backup/sda-gpt-backup-20260518-001.bin
```

### 4.3 Ellenőrzés mentés után (ajánlott)

| Lépés | Művelet |
|-------|---------|
| A | `ls -l backup/*.bin` — a fájl mérete **nem nulla** |
| B | Partctl: **GPT tabla ellenorzes** (lásd §6) — `sgdisk -v` rc=0 |
| C | **Lemez áttekintés** — partíciók sorrendje és mérete egyezik a várt layouttal |

### 4.4 Egyenértékű parancssor (referencia)

```bash
sudo sgdisk --backup=backup/sda-gpt-backup-20260518-001.bin /dev/sda
```

---

## 5. Lépésről lépésre — GPT partíciós tábla helyreállítás (Restore GPT backup)

### 5.1 Mikor használd?

- A GPT fejléc **sérült**, de a partíciók **adata** még a helyükön van
- Vissza szeretnéd állítani egy **korábban mentett** layoutot
- Tesztkörnyezetben **ismételhető** állapotot akarsz

### 5.2 Menüút

| Lépés | Hol | Tevékenység |
|-------|-----|-------------|
| 1 | Főmenü → **`1`** | Ugyanaz a lemez (`sda`) |
| 2 | Főmenü → **`3`** | Partíció kezelés |
| 3 | Almenü | **GPT tabla helyreallitas** / **Restore GPT backup** |
| 4 | Lista | A `backup/` mappában lévő **`.bin`** fájlok — válassz egyet (szám + Enter) |
| 5 | Figyelmeztető panel | **Igen** — felülírja a jelenlegi GPT metaadatokat |
| 6 | Detach ellenőrzés | Ha valami még csatolva van, a program figyelmeztet / blokkol |
| 7 | Folyamat | `sgdisk --load-backup=…` + `partprobe` / `udevadm settle` |
| 8 | Info | Siker üzenet |

| Kiválasztás | Megerősítés | Információ |
| --- | --- | --- |
| ![gpt_tabla_helyreallitas_1](/img/gpt_tabla_helyreallitas_1.jpg "GPT partíciós tábla helyreállítása #1") | ![gpt_tabla_helyreallitas_2](/img/gpt_tabla_helyreallitas_2.jpg "GPT partíciós tábla helyreállítása #2") | ![gpt_tabla_helyreallitas_3](/img/gpt_tabla_helyreallitas_3.jpg "GPT partíciós tábla helyreállítása #3") |

**Ha a lista üres:** `(no .bin files found in this directory)` — előbb készíts mentést (§4), vagy másold a `.bin` fájlt a Partctl indítási könyvtár **`backup/`** alá.

### 5.3 Figyelmeztetés szövege (jelentése)

```text
Figyelem: ez felulirja a jelenlegi GPT metaadatokat.
```

Ez **nem** formázza újra a partíciókat, de ha a **mentett layout** eltér a jelenlegi adattól (más határok, törölt partíció), az operációs rendszer **rossz szektorokat** olvashat — **adatvesztés** vagy **sérült fájlrendszer** lehetséges.

### 5.4 Egyenértékű parancssor

```bash
sudo umount /dev/sda* 2>/dev/null || true
sudo sgdisk --load-backup=backup/sda-gpt-backup-20260518-001.bin /dev/sda
sudo partprobe /dev/sda
```

---

## 6. Lépésről lépésre — GPT partíciós tábla ellenőrzés (Verify GPT partition table)

### 6.1 Mit csinál?

Az `sgdisk -v` **nem módosít** semmit — **ellenőrzi**:

- Elsődleges és másodlagos GPT fejléc egyezése
- Partícióhatárok (pl. **2048 szektor** / 1 MiB igazítás figyelmeztetések)
- Átfedő vagy hiányzó bejegyzések

Használd **mentés előtt és után**, valamint **helyreállítás után**.

### 6.2 Menüút

| Lépés | Hol | Tevékenység |
|-------|-----|-------------|
| 1 | Főmenü → **`1`** | Lemez kiválasztása |
| 2 | Főmenü → **`3`** | Partíció kezelés |
| 3 | Almenü | **GPT tabla ellenorzes** / **Verify GPT partition table** |
| 4 | Megerősítés | **Igen** |
| 5 | Eredmény | Info panel: „ellenőrzés kész” + `sgdisk` kimenet |

| Megerősítés | Információ |
| --- | --- |
| ![gpt_tabla_ellenorzes_4](/img/gpt_tabla_ellenorzes_4.jpg "GPT partíciós tábla ellenőrzés #1") | ![gpt_tabla_ellenorzes_5](/img/gpt_tabla_ellenorzes_5.jpg "GPT partíciós tábla ellenőrzés #2") |

### 6.3 Tipikus kimenetek

| Eredmény | Jelentés |
|----------|----------|
| **Completed successfully** | A tábla **konzisztens** (a figyelmeztetések lehetnek tájékoztatók) |
| **Problem with GPT data structures** | Sérült vagy hiányzó másodlagos fejléc — mentés / helyreállítás vagy `sgdisk` javítás szükséges |
| **Doesn't end on a 2048-sector boundary** | Igazítási figyelmeztetés — lásd [partíció-igazítás útmutató](particio-igazitas-partctl-guide.md) |

### 6.4 Egyenértékű parancssor

```bash
sudo sgdisk -v /dev/sda
```

---

## 7. Esettanulmány A — GPT partíciós tábla mentés elkészítése egy kockázatos művelet előtt (USB GPT, több partíció)

**Helyzet:** `/dev/sda` — 32 GB USB, GPT, négy partíció (ESP + adat + WinRE + szabad). Átméretezni vagy törölni fogsz egy partíciót.

| # | Művelet | Cél |
|---|---------|-----|
| 1 | **Lemez áttekintés** | Aktuális layout feljegyzése (képernyőfotó / jegyzet) |
| 2 | **GPT tabla mentes** | `backup/sda-gpt-backup-20260518-001.bin` |
| 3 | **GPT tabla ellenorzes** | Baseline: sikeres `-v` |
| 4 | *(felhasználói szerkesztés)* | Pl. partíció törlés / átméretezés |
| 5 | **GPT tabla ellenorzes** | Új állapot ellenőrzése |
| 6 | Ha elrontottad a táblát, de az adat még él | **GPT tabla helyreallitas** a `001.bin`-ből, majd újra `-v` |

**Tanulság:** A mentés **gyors** és **csak metaadat** — egy rossz `sgdisk` / parted lépés után gyakran ez menti meg a layoutot.

---

## 8. Esettanulmány B — „Invalid GPT” / másodlagos fejléc hiba helyreállítása

**Helyzet:** A lemez **elején** sérült a GPT (pl. véletlen `dd`, rossz klónozás), de a partíciók **adat** még olvasható volt korábban. Van egy **friss** `backup/sda-gpt-backup-20260518-001.bin`.

| # | Művelet |
|---|---------|
| 1 | **Unmount** minden `sda` partíció (`umount`) |
| 2 | Partctl: **GPT tabla helyreallitas** → válaszd a legutóbbi jó `.bin`-t |
| 3 | **GPT tabla ellenorzes** — várható: sikeres ellenőrzés |
| 4 | **Lemez áttekintés** — partíciók látszanak-e |
| 5 | Próba **csatolás** / `fsck` — csak ha a layout **egyezik** a tényleges fájlrendszerrel |

**Ha a helyreállítás után a fájlrendszer hibás:** a layout nem egyezett az adattal, vagy az adatszektorok is sérültek — ez **nem** oldható meg csak GPT restore-lal.

---

## 9. Esettanulmány C — Ismételt tesztkörnyezet (azonos layout visszaállítása)

**Helyzet:** Fejlesztői gépen többször ugyanarra az USB lemezre telepítesz tesztrendszert; a **partíciós váz** (méret, GUID típusok) mindig ugyanaz legyen.

| Fázis | Művelet |
|-------|---------|
| **Egyszer** | Jó layout kialakítása → **GPT tabla mentes** → `golden.bin` másolat archívumba |
| **Minden teszt előtt** | Wipe / tábla törlés (ha kell) → új GPT → **Restore** `golden.bin` → **Verify** |
| **Ellenőrzés** | **Lemez áttekintés** + opcionális **GPT particio tipuskod** / formázás |

> **Megjegyzés:** Ha a lemez **fizikai méret** változik (nagyobb USB), a régi mentés **nem** biztos, hogy közvetlenül betölthető — ellenőrizd `sgdisk -v` kimenetét.

---

## 10. Hibaelhárítás

| Tünet | Lehetséges ok | Teendő |
|-------|----------------|--------|
| „Csak GPT particios tablan tamogatott” | MBR lemez | **Create partition table** → GPT, vagy maradj MBR-nél (nincs GPT backup menü) |
| „Hianyzik: sgdisk” | Nincs `gdisk` csomag | `sudo bash setup.sh` → Telepítés |
| Üres `.bin` lista helyreállításkor | Nincs fájl a `backup/` mappában | Mentés (§4) vagy `.bin` másolása a projekt `backup/` alá |
| Restore után rossz / üres partíció | Rossz `.bin` vagy másik lemez | Ellenőrizd a lemez nevét (`lsblk`) és a mentés fájlnevét (`sda-…`) |
| Verify figyelmeztetés 2048 boundary | Igazítás | [Partíció igazítás](particio-igazitas-partctl-guide.md) |
| „Device busy” / detach hiba | Csatolt partíció | **Lemez kezelés** → ideiglenes lecsatolás, vagy `umount` |

**Napló:** `log/partctl-YYYYMMDD-HHMMSS.log` — a futó `sgdisk` parancs és a visszatérési kód is benne van.

```markdown
https://github.com/drcyberg/partctl/blob/main/example/gpt-tabla-mentes-helyreallitas-partctl-guide.md
```

### Fő oldal (Partctl)

- [Partctl](https://drcyberg.github.io/partctl/web/partctl)

### Köszönöm ha támogatsz

- ***Buy me a coffee***: [LINK](https://buymeacoffee.com/drcyberg)
- ***Paypal (QR Code)***: [LINK](https://github.com/drcyberg/partctl/blob/main/img/qrcode.png)
- ***Paypal (URL)***: [LINK](https://paypal.me/Kunee82)

*Utolsó frissítés jelleg: Partctl V1.0.0 viselkedés — a **Particio kezeles** lista ábécérendje miatt a konkrét **sorszámok** mindig a futó programban ellenőrizendők.*
