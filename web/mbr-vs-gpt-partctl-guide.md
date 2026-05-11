# MBR és GPT partíciós tábla — összefoglaló és `partctl.sh` menüútmutató

> **Cél:** Rövid műszaki háttér az **MBR (msdos)** és a **GPT** táblákhoz, **tanácsokkal**, és minden fő pontnál: **hogyan valósítható meg a Partctl-lel** (`bash partctl.sh`).  
> **Figyelem:** particiós tábla és partíciók módosítása **adatvesztést** okozhat. Teszteld **nem** a futó rendszer lemezén; előtte **mentés** és **leválasztás** (unmount), ahol kell.

---

## Közös előkészület — `partctl.sh`

```bash
bash partctl.sh
```

![](/partctl/img/terminal_1.jpg)

A program a **`python3 -m partctl_ncurses_app`** modult indítja (`PYTHONPATH` + `--lang-dir`).

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

![](/partctl/img/lemez_kivalasztasa_2.jpg)

**Navigáció:** `Fel` / `Le` (vagy `k` / `j`), **Enter**; vagy a sor elején látható **`N.`** szám begépelése, majd **Enter**. **Vissza:** súgó szerint **Backspace** / **`q`**.

**Particio kezeles** almenü: a tételek **ábécérendbe** vannak rendezve — a pontos **sorszámot** mindig a **képernyőn** ellenőrizd; a lenti útmutatóban a **menüpont címkéjét** (angol + magyar) használjuk.

**Lemez kezeles** almenü: **rögzített** sorrend (angol címkék):

| # | Menüpont |
|---|----------|
| 1–9 | EXT4 / repair / label / mount / unmount / NTFS műveletek |
| **10** | **Disk cleanup (Wipe)** |
| 11 | Back |

Először mindig: **főmenü → `1`** — **Lemez kivalasztasa** — a listában válaszd ki a **`sdc`** (vagy cél) sort (**sorszám + Enter** vagy kurzor + Enter).

---

## 1. MBR (Master Boot Record / `msdos`) — **Windows XP** példa (`/dev/sdb`)

![](/partctl/img/xp_logo.png)

### 1.1 Rövid összefoglaló

- Az **MBR** séma a lemez **elején** tárolja a boot rekordot és a **partíciós bejegyzéseket** (klasszikusan **4 „slot”**: elsődleges partíciók + opcionálisan egy **kiterjesztett** konténer, abban **logikai** partíciók).
- **Partíció-ID:** egy bájtos **hex kód** (pl. `0x83` Linux, `0x07` Microsoft basic data).
- **Korlátok:** a klasszikus MBR + BIOS környezetben gyakori a **~2 TiB** körüli határ (LBA48 / címzés); több partícióhoz a **kiterjesztett + logikai** séma kell.
- **Boot:** hagyományos **BIOS / CSM** boot gyakran MBR-lel párosul (modern **tiszta UEFI + nagy lemez** esetén inkább **GPT** javasolt).
- **Windows XP:** hivatalosan **MBR + BIOS** boot a jellemző (nem GPT/UEFI „modern” út); a rendszerpartíció tipikusan **NTFS** (`0x07`) vagy régebbi telepítéseknél **FAT32** (`0x0B` / `0x0C`) — a Partctl ugyanúgy **primary** partíciókat és **hex típuskódot** kezel.

### 1.2 Tippek, trükkök, tanácsok (MBR + Windows XP + Partctl)

1. **Ne a futó rendszer lemezén** kísérletezz — külön **teszt HDD/SSD** vagy **USB** (pl. **`sdb`**) ideális; a tényleges XP telepítéshez kell **BIOS boot** és a **Windows XP telepítő** (CD/USB) — a Partctl **csak a lemez előkészítését** végzi.
2. **„Tiszta lap”:** **Lemez kezeles → Disk cleanup (Wipe)** (`főmenü 4` → `10`), **teljes lemez** cél, szükség szerint a **partíciós tábla törlése** opció bejelölve — utána **Particios tabla letrehozasa** MBR-rel.
3. **Két primary elég gyakori XP-hez:** egy **„C:”** rendszer (NTFS) + egy **„D:”** adat (NTFS), mindkettő **primary** — így **nem** kell extended/logikai, amíg nem kell 4-nél több kötet.
4. **Négy elsődleges betelt?** A Partctl **Create partition** logikája MBR-n: az első **három** új partíció tipikusan **primary**; ha már **3 nem kiterjesztett primary** van és nincs extended, a **negyedik** lépésnél a program **kiterjesztett** partíciót hozhat létre, majd **logikai**kat (részletek: `partition_create_ops.py`). Ha elakad: **Lemez attekintes** + **Particio torlese** / újratervezés.
5. **Típus beállítása:** MBR-n a **Partition type code (hex, MBR)** / **MBR particio tipuskod** varázsló — XP „adat” partíciókhoz a listában a **Microsoft basic data (NTFS/exFAT)** jellegű **`07`** szokott illeni; GPT GUID varázsló **nem** MBR-re való (a program figyelmeztet).
6. **Particionev:** a Partctl **Particionev megvaltoztatasa** funkciója **csak GPT** táblán támogatott — MBR-n ne ezt várd a „címke” megoldására.
7. **Ellenőrzés:** **Lemez attekintes** — a particiós tábla sorban **`dos`** / **`msdos`** jelenik meg.

### 1.3 Partctl — példa menülépések (MBR — **Windows XP**, pl. `/dev/sdb` tesztlemez)

| Lépés | Menüút (rövid) | Mit csinálsz |
|-------|----------------|-------------|
| 1 | **Főmenü → `1`** Select Disk | Kiválasztod **`sdb`** (példa: másodlagos lemez / USB, **nem** az aktív rendszerlemez). |
| 2 | **Főmenü → `4` → `10`** Disk cleanup (Wipe) | Cél: **whole disk**; opcionálisan **partition table wipe**; megerősítés. *(Kihagyható, ha már üres a lemez.)* |
| 3 | **Főmenü → `3`** → **Create partition table** / **Particios tabla letrehozasa** | Tábla típusánál: **`2` — MBR (msdos)** (a listában: **1** = GPT, **2** = MBR). Megerősítés, szükség esetén leválasztás. |
| 4 | **Főmenü → `3`** → **Create partition** / **Particio letrehozasa** | **Első** partíció (későbbi **„C:”** / rendszer): kezdő a javasolt érték, vég pl. **`+40GiB`** — klasszikus XP rendszerkönyvtár-méret (igény szerint nagyobb). |
| 5 | Ugyanitt **Create partition** (ismét) | **Második** partíció (későbbi **„D:”** / adat): kezdő a javasolt érték, vég pl. **`100%`** vagy **`+80GiB`** — a maradék szabad hely kitöltése. |
| 6 | *(Opcionális további partíciók)* | Ha **3.** vagy **4.** primary / extended / logikai kell: kövesd a Partctl **felugró** kérdéseit (extended bővítés stb.). Egyszerű **C+D** két primary esetén ez a lépés kimarad. |
| 7 | **Főmenü → `3`** → **Partition type code (hex, MBR)** | **`sdb1`** és **`sdb2`:** válaszd a **`07`** (Microsoft basic data / NTFS) típust a listából — illik az XP **NTFS** kötetekhez. |
| 8 | **Főmenü → `3`** → **Partition format** / **Particio formazas** | Mindkét partíción **ntfs** (ha a Partctl listájában elérhető) — ez **Linuxról** előkészíti a köteteket; a **Windows XP telepítő** ettől függetlenül saját formázást is kínálhat. |
| 9 | **Főmenü → `2`** Disk Overview | Ellenőrzés: tábla **msdos**, **`sdb1`** / **`sdb2`** méretek, sorrend. |

### 1.4 Példa — tipikus **Windows XP (MBR)** elrendezés (illusztráció)

| Eszköz | Szerep (XP) | MBR hex (tipikus) | Megjegyzés |
|--------|-------------|-------------------|------------|
| `sdb1` | **„C:”** — rendszer + Program Files | **`07`** (NTFS) | Primary; XP telepítő célpontja |
| `sdb2` | **„D:”** — adat, játékok, mentések | **`07`** (NTFS) | Primary; maradék lemezterület |

*(A konkrét **GiB** értékek a `parted` szabad sávjától függnek — a **Particio parameterek** panel mutatja. FAT32-es XP-hez más méret / `0B` típus lehet szükséges — ritkább.)*

---

## 2. GPT (GUID Partition Table) — **Windows 11** példa (`/dev/sda`)

![](/partctl/img/win11_logo.png)

### 2.1 Rövid összefoglaló

- A **GPT** a lemez **elején és végén** is tárol **fejlécet és partíciós bejegyzéseket**; sok partíció (tipikusan 128 bejegyzés), egyedi **GUID** típusok.
- **UEFI** boot és **nagy (>2 TiB) lemezek** esetén ez a **szabványos** választás (**Windows 11** hivatalosan is **GPT + UEFI** irány — megegyezik a [`win11-gpt-uefi-particio-whitepaper.md`](win11-gpt-uefi-particio-whitepaper.md) **`/dev/sda`** példájával: ESP + MSR + rendszer + WinRE).
- **Partíció-ID:** **GUID** (pl. EFI System, Microsoft basic data, WinRE).
- **Partctl:** **GPT backup / restore / verify**, **GPT attributes**, **Partition type code (GUID, GPT)** — ezek **csak GPT** táblán működnek.

### 2.2 Tippek, trükkök, tanácsok (GPT + Windows 11 + Partctl)

1. **Első partíció kezdete:** fogadd el a program által javasolt **igazított** kezdőértéket (gyakran **`2048s`** ≈ 1 MiB) — jó gyakorlat SSD / igazítás szempontjából.
2. **Tábla létrehozás:** a **Create partition table** varázsló belsőleg **tisztító** lépéseket is futtathat (`sgdisk` / `wipefs`) a **`mklabel gpt`** előtt — mindig olvasd el a **figyelmeztetést**.
3. **Típus + név:** **Partition type code (GUID, GPT)** a Microsoft / Linux GUID-okhoz; **Particionev megvaltoztatasa** **GPT-n** támogatott.
4. **WinRE / speciális bitek:** **GPT attributes** — `sgdisk` kell; íráshoz tipikusan **root**.
5. **Ellenőrzés:** **Verify GPT partition table**; majd **Disk Overview** + partíción **Enter** részletek (GPT attributum sor, ha `sgdisk` elérhető).

### 2.3 Partctl — példa menülépések (**Windows 11**, **`/dev/sda`** — a whitepaperrel megegyezően)

| Lépés | Menüút (rövid) | Mit csinálsz |
|-------|----------------|-------------|
| 1 | **Főmenü → `1`** | Kiválasztod a **`sda`** céllemezt. |
| 2 | **Főmenü → `4` → `10`** (opcionális) | Teljes lemez wipe + opcionális tábla törlés — „nulláról” GPT-hez. |
| 3 | **Főmenü → `3`** → **Create partition table** | Tábla típus: **`1` — GPT**. Megerősítés. |
| 4 | **Főmenü → `3`** → **Create partition** (ismételj) | Sorban pl. **`+512MiB`** (ESP), **`+16MiB`** (MSR), **`+120GiB`** (rendszer), **`+1024MiB`** (WinRE) — a szabad sáv szerint. |
| 5 | **Főmenü → `3`** → **Partition type code (GUID, GPT)** | `sda1` → **EFI System**; `sda2` → **Microsoft reserved**; `sda3` → **Microsoft basic data**; `sda4` → **Windows Recovery Environment (WinRE)**. |
| 6 | **Partition format** | ESP: **vfat**; rendszer + WinRE: **ntfs** (MSR: tipikusan nem formázod). |
| 7 | (Opcionális) **GPT attributes** | WinRE partíción a szükséges bitek. |
| 8 | **Verify GPT partition table** + **Disk Overview** | Végleges ellenőrzés. |

*(Részletes példa-leképezés: lásd még [`win11-gpt-uefi-particio-whitepaper.md`](win11-gpt-uefi-particio-whitepaper.md).)*

### 2.4 Példa — tipikus **Windows 11** GPT sor (`/dev/sda`, illusztráció)

| Partíció | Méret (példa) | GUID szerep |
|----------|----------------|-------------|
| `sda1` | 512 MiB | EFI System |
| `sda2` | 16 MiB | Microsoft reserved |
| `sda3` | 120 GiB | Microsoft basic data (rendszer) |
| `sda4` | 1 GiB | WinRE |

---

## 3. MBR vs GPT — összehasonlító táblázat és felhasználás

| Szempont | MBR (`msdos`) | GPT |
|-----------|----------------|-----|
| **Boot környezet** | BIOS / CSM gyakori | **UEFI** (modern PC, Windows 11 ajánlott irány) |
| **Lemez / partíció méret** | Gyakorlati korlátok **~2 TiB** körül (klasszikus MBR) | **Nagy lemezek**, több TiB |
| **Partíciók száma** | Max **4 primary**; továbbiak **extended + logikai** (Partctl: automatikus extended lépés) | Sok partíció (tipikusan **128** bejegyzés) |
| **Típus azonosító** | **Hex** (1 bájt) | **GUID** |
| **Partctl: tábla létrehozás** | **Create partition table** → **`2` MBR (msdos)** | **Create partition table** → **`1` GPT** |
| **Partctl: típus beállítás** | **Partition type code (hex, MBR)** | **Partition type code (GUID, GPT)** |
| **Partctl: particionev** | **Nem támogatott** (GPT-only üzenet) | **Change partition name** / **Particionev megvaltoztatasa** |
| **Partctl: GPT backup / restore / verify** | Nem alkalmazható | **Create / Restore / Verify GPT** menük |
| **Partctl: GPT attributes** | Nem alkalmazható | **GPT attributes** (WinRE bitek stb.) |
| **Tipikus felhasználás** | **Windows XP** / retro PC, **BIOS** boot, kis lemez, legacy; egyszerű Linux MBR layout | **Windows 11**, UEFI PC, nagy HDD/SSD, Apple-intel kompatibilitás, szerverek |

### Felhasználási terület (mikor melyik?)

| Helyzet | Javasolt tábla |
|---------|----------------|
| **Új Windows 11 telepítő / UEFI gép** | **GPT** (lásd §2, `sda` példa) |
| **2 TiB feletti** lemez vagy nagy egy partíció | **GPT** |
| **Windows XP / régi gépek, csak BIOS boot**, kb. **2 TiB alatti** lemez | **MBR** (lásd §1, `sdb` példa) |
| **Partctl tanulás / teszt** | **USB** + először **MBR (XP)** vagy **GPT (Win11)** külön próbalemez |

---

## 4. Gyors hivatkozás — Partctl művelet vs tábla típusa

| Művelet (angol menü) | MBR | GPT |
|----------------------|-----|-----|
| Create partition table | `msdos` | `gpt` |
| Create partition | Igen (primary / extended / logical logika) | Igen |
| Partition format | Igen | Igen |
| Partition type code (hex, MBR) | Igen | Nem |
| Partition type code (GUID, GPT) | Nem | Igen |
| Change partition name | Nem | Igen |
| GPT backup / restore / verify | Nem | Igen |
| GPT attributes | Nem | Igen |

---

## 5. Verzió

| Mező | Érték |
|------|--------|
| Dokumentum | MBR vs GPT + `partctl.sh` menüútmutató (MBR példa: **Windows XP**; GPT példa: **Windows 11 / `sda`**) |
| Forráskód hivatkozások | `partctl_ncurses_app/ui/split_menus.py`, `partition_table_views.py` (`draw_partition_table_type_menu`), `partition_create_ops.py` |

```markdown
[MBR vs GPT — Partctl útmutató](docs/mbr-gpt-partctl-guide.md)
```
# MBR és GPT partíciós tábla — összefoglaló és `partctl.sh` menüútmutató

> **Cél:** Rövid műszaki háttér az **MBR (msdos)** és a **GPT** táblákhoz, **tanácsokkal**, és minden fő pontnál: **hogyan valósítható meg a Partctl-lel** (`bash partctl.sh`).  
> **Figyelem:** particiós tábla és partíciók módosítása **adatvesztést** okozhat. Teszteld **nem** a futó rendszer lemezén; előtte **mentés** és **leválasztás** (unmount), ahol kell.

---

## Közös előkészület — `partctl.sh`

```bash
bash partctl.sh
```

A program a **`python3 -m partctl_ncurses_app`** modult indítja (`PYTHONPATH` + `--lang-dir`).

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

**Navigáció:** `Fel` / `Le` (vagy `k` / `j`), **Enter**; vagy a sor elején látható **`N.`** szám begépelése, majd **Enter**. **Vissza:** súgó szerint **Backspace** / **`q`**.

**Particio kezeles** almenü: a tételek **ábécérendbe** vannak rendezve — a pontos **sorszámot** mindig a **képernyőn** ellenőrizd; a lenti útmutatóban a **menüpont címkéjét** (angol + magyar) használjuk.

**Lemez kezeles** almenü: **rögzített** sorrend (angol címkék):

| # | Menüpont |
|---|----------|
| 1–9 | EXT4 / repair / label / mount / unmount / NTFS műveletek |
| **10** | **Disk cleanup (Wipe)** |
| 11 | Back |

---

## 1. MBR (Master Boot Record / `msdos`) partíciós tábla

### 1.1 Rövid összefoglaló

- Az **MBR** séma a lemez **elején** tárolja a boot rekordot és a **partíciós bejegyzéseket** (klasszikusan **4 „slot”**: elsődleges partíciók + opcionálisan egy **kiterjesztett** konténer, abban **logikai** partíciók).
- **Partíció-ID:** egy bájtos **hex kód** (pl. `0x83` Linux, `0x07` Microsoft basic data).
- **Korlátok:** a klasszikus MBR + BIOS környezetben gyakori a **~2 TiB** körüli határ (LBA48 / címzés); több partícióhoz a **kiterjesztett + logikai** séma kell.
- **Boot:** hagyományos **BIOS / CSM** boot gyakran MBR-lel párosul (modern **tiszta UEFI + nagy lemez** esetén inkább **GPT** javasolt).
- **Windows XP:** hivatalosan **MBR + BIOS** boot a jellemző (nem GPT/UEFI „modern” út); a rendszerpartíció tipikusan **NTFS** (`0x07`) vagy régebbi telepítéseknél **FAT32** (`0x0B` / `0x0C`) — a Partctl ugyanúgy **primary** partíciókat és **hex típuskódot** kezel.

### 1.2 Tippek, trükkök, tanácsok (MBR + Windows XP + Partctl)

1. **Ne a futó rendszer lemezén** kísérletezz — külön **teszt HDD/SSD** vagy **USB** (pl. **`sdb`**) ideális; a tényleges XP telepítéshez kell **BIOS boot** és a **Windows XP telepítő** (CD/USB) — a Partctl **csak a lemez előkészítését** végzi.
2. **„Tiszta lap”:** **Lemez kezeles → Disk cleanup (Wipe)** (`főmenü 4` → `10`), **teljes lemez** cél, szükség szerint a **partíciós tábla törlése** opció bejelölve — utána **Particios tabla letrehozasa** MBR-rel.
3. **Két primary elég gyakori XP-hez:** egy **„C:”** rendszer (NTFS) + egy **„D:”** adat (NTFS), mindkettő **primary** — így **nem** kell extended/logikai, amíg nem kell 4-nél több kötet.
4. **Négy elsődleges betelt?** A Partctl **Create partition** logikája MBR-n: az első **három** új partíció tipikusan **primary**; ha már **3 nem kiterjesztett primary** van és nincs extended, a **negyedik** lépésnél a program **kiterjesztett** partíciót hozhat létre, majd **logikai**kat (részletek: `partition_create_ops.py`). Ha elakad: **Lemez attekintes** + **Particio torlese** / újratervezés.
5. **Típus beállítása:** MBR-n a **Partition type code (hex, MBR)** / **MBR particio tipuskod** varázsló — XP „adat” partíciókhoz a listában a **Microsoft basic data (NTFS/exFAT)** jellegű **`07`** szokott illeni; GPT GUID varázsló **nem** MBR-re való (a program figyelmeztet).
6. **Particionev:** a Partctl **Particionev megvaltoztatasa** funkciója **csak GPT** táblán támogatott — MBR-n ne ezt várd a „címke” megoldására.
7. **Ellenőrzés:** **Lemez attekintes** — a particiós tábla sorban **`dos`** / **`msdos`** jelenik meg.

### 1.3 Partctl — példa menülépések (MBR — **Windows XP**, pl. `/dev/sdb` tesztlemez)

| Lépés | Menüút (rövid) | Mit csinálsz |
|-------|----------------|-------------|
| 1 | **Főmenü → `1`** Select Disk | Kiválasztod **`sdb`** (példa: másodlagos lemez / USB, **nem** az aktív rendszerlemez). |
| 2 | **Főmenü → `4` → `10`** Disk cleanup (Wipe) | Cél: **whole disk**; opcionálisan **partition table wipe**; megerősítés. *(Kihagyható, ha már üres a lemez.)* |
| 3 | **Főmenü → `3`** → **Create partition table** / **Particios tabla letrehozasa** | Tábla típusánál: **`2` — MBR (msdos)** (a listában: **1** = GPT, **2** = MBR). Megerősítés, szükség esetén leválasztás. |
| 4 | **Főmenü → `3`** → **Create partition** / **Particio letrehozasa** | **Első** partíció (későbbi **„C:”** / rendszer): kezdő a javasolt érték, vég pl. **`+40GiB`** — klasszikus XP rendszerkönyvtár-méret (igény szerint nagyobb). |
| 5 | Ugyanitt **Create partition** (ismét) | **Második** partíció (későbbi **„D:”** / adat): kezdő a javasolt érték, vég pl. **`100%`** vagy **`+80GiB`** — a maradék szabad hely kitöltése. |
| 6 | *(Opcionális további partíciók)* | Ha **3.** vagy **4.** primary / extended / logikai kell: kövesd a Partctl **felugró** kérdéseit (extended bővítés stb.). Egyszerű **C+D** két primary esetén ez a lépés kimarad. |
| 7 | **Főmenü → `3`** → **Partition type code (hex, MBR)** | **`sdb1`** és **`sdb2`:** válaszd a **`07`** (Microsoft basic data / NTFS) típust a listából — illik az XP **NTFS** kötetekhez. |
| 8 | **Főmenü → `3`** → **Partition format** / **Particio formazas** | Mindkét partíción **ntfs** (ha a Partctl listájában elérhető) — ez **Linuxról** előkészíti a köteteket; a **Windows XP telepítő** ettől függetlenül saját formázást is kínálhat. |
| 9 | **Főmenü → `2`** Disk Overview | Ellenőrzés: tábla **msdos**, **`sdb1`** / **`sdb2`** méretek, sorrend. |

### 1.4 Példa — tipikus **Windows XP (MBR)** elrendezés (illusztráció)

| Eszköz | Szerep (XP) | MBR hex (tipikus) | Megjegyzés |
|--------|-------------|-------------------|------------|
| `sdb1` | **„C:”** — rendszer + Program Files | **`07`** (NTFS) | Primary; XP telepítő célpontja |
| `sdb2` | **„D:”** — adat, játékok, mentések | **`07`** (NTFS) | Primary; maradék lemezterület |

*(A konkrét **GiB** értékek a `parted` szabad sávjától függnek — a **Particio parameterek** panel mutatja. FAT32-es XP-hez más méret / `0B` típus lehet szükséges — ritkább.)*

---

## 2. GPT (GUID Partition Table) — **Windows 11** példa (`/dev/sda`)

### 2.1 Rövid összefoglaló

- A **GPT** a lemez **elején és végén** is tárol **fejlécet és partíciós bejegyzéseket**; sok partíció (tipikusan 128 bejegyzés), egyedi **GUID** típusok.
- **UEFI** boot és **nagy (>2 TiB) lemezek** esetén ez a **szabványos** választás (**Windows 11** hivatalosan is **GPT + UEFI** irány — megegyezik a [`win11-gpt-uefi-particio-whitepaper.md`](win11-gpt-uefi-particio-whitepaper.md) **`/dev/sda`** példájával: ESP + MSR + rendszer + WinRE).
- **Partíció-ID:** **GUID** (pl. EFI System, Microsoft basic data, WinRE).
- **Partctl:** **GPT backup / restore / verify**, **GPT attributes**, **Partition type code (GUID, GPT)** — ezek **csak GPT** táblán működnek.

### 2.2 Tippek, trükkök, tanácsok (GPT + Windows 11 + Partctl)

1. **Első partíció kezdete:** fogadd el a program által javasolt **igazított** kezdőértéket (gyakran **`2048s`** ≈ 1 MiB) — jó gyakorlat SSD / igazítás szempontjából.
2. **Tábla létrehozás:** a **Create partition table** varázsló belsőleg **tisztító** lépéseket is futtathat (`sgdisk` / `wipefs`) a **`mklabel gpt`** előtt — mindig olvasd el a **figyelmeztetést**.
3. **Típus + név:** **Partition type code (GUID, GPT)** a Microsoft / Linux GUID-okhoz; **Particionev megvaltoztatasa** **GPT-n** támogatott.
4. **WinRE / speciális bitek:** **GPT attributes** — `sgdisk` kell; íráshoz tipikusan **root**.
5. **Ellenőrzés:** **Verify GPT partition table**; majd **Disk Overview** + partíción **Enter** részletek (GPT attributum sor, ha `sgdisk` elérhető).

### 2.3 Partctl — példa menülépések (**Windows 11**, **`/dev/sda`** — a whitepaperrel megegyezően)

| Lépés | Menüút (rövid) | Mit csinálsz |
|-------|----------------|-------------|
| 1 | **Főmenü → `1`** | Kiválasztod a **`sda`** céllemezt. |
| 2 | **Főmenü → `4` → `10`** (opcionális) | Teljes lemez wipe + opcionális tábla törlés — „nulláról” GPT-hez. |
| 3 | **Főmenü → `3`** → **Create partition table** | Tábla típus: **`1` — GPT**. Megerősítés. |
| 4 | **Főmenü → `3`** → **Create partition** (ismételj) | Sorban pl. **`+512MiB`** (ESP), **`+16MiB`** (MSR), **`+120GiB`** (rendszer), **`+1024MiB`** (WinRE) — a szabad sáv szerint. |
| 5 | **Főmenü → `3`** → **Partition type code (GUID, GPT)** | `sda1` → **EFI System**; `sda2` → **Microsoft reserved**; `sda3` → **Microsoft basic data**; `sda4` → **Windows Recovery Environment (WinRE)**. |
| 6 | **Partition format** | ESP: **vfat**; rendszer + WinRE: **ntfs** (MSR: tipikusan nem formázod). |
| 7 | (Opcionális) **GPT attributes** | WinRE partíción a szükséges bitek. |
| 8 | **Verify GPT partition table** + **Disk Overview** | Végleges ellenőrzés. |

*(Részletes példa-leképezés: lásd még [`win11-gpt-uefi-particio-whitepaper.md`](win11-gpt-uefi-particio-whitepaper.md).)*

### 2.4 Példa — tipikus **Windows 11** GPT sor (`/dev/sda`, illusztráció)

| Partíció | Méret (példa) | GUID szerep |
|----------|----------------|-------------|
| `sda1` | 512 MiB | EFI System |
| `sda2` | 16 MiB | Microsoft reserved |
| `sda3` | 120 GiB | Microsoft basic data (rendszer) |
| `sda4` | 1 GiB | WinRE |

---

## 3. MBR vs GPT — összehasonlító táblázat és felhasználás

| Szempont | MBR (`msdos`) | GPT |
|-----------|----------------|-----|
| **Boot környezet** | BIOS / CSM gyakori | **UEFI** (modern PC, Windows 11 ajánlott irány) |
| **Lemez / partíció méret** | Gyakorlati korlátok **~2 TiB** körül (klasszikus MBR) | **Nagy lemezek**, több TiB |
| **Partíciók száma** | Max **4 primary**; továbbiak **extended + logikai** (Partctl: automatikus extended lépés) | Sok partíció (tipikusan **128** bejegyzés) |
| **Típus azonosító** | **Hex** (1 bájt) | **GUID** |
| **Partctl: tábla létrehozás** | **Create partition table** → **`2` MBR (msdos)** | **Create partition table** → **`1` GPT** |
| **Partctl: típus beállítás** | **Partition type code (hex, MBR)** | **Partition type code (GUID, GPT)** |
| **Partctl: particionev** | **Nem támogatott** (GPT-only üzenet) | **Change partition name** / **Particionev megvaltoztatasa** |
| **Partctl: GPT backup / restore / verify** | Nem alkalmazható | **Create / Restore / Verify GPT** menük |
| **Partctl: GPT attributes** | Nem alkalmazható | **GPT attributes** (WinRE bitek stb.) |
| **Tipikus felhasználás** | **Windows XP** / retro PC, **BIOS** boot, kis lemez, legacy; egyszerű Linux MBR layout | **Windows 11**, UEFI PC, nagy HDD/SSD, Apple-intel kompatibilitás, szerverek |

### Felhasználási terület (mikor melyik?)

| Helyzet | Javasolt tábla |
|---------|----------------|
| **Új Windows 11 telepítő / UEFI gép** | **GPT** (lásd §2, `sda` példa) |
| **2 TiB feletti** lemez vagy nagy egy partíció | **GPT** |
| **Windows XP / régi gépek, csak BIOS boot**, kb. **2 TiB alatti** lemez | **MBR** (lásd §1, `sdb` példa) |
| **Partctl tanulás / teszt** | **USB** + először **MBR (XP)** vagy **GPT (Win11)** külön próbalemez |

---

## 4. Gyors hivatkozás — Partctl művelet vs tábla típusa

| Művelet (angol menü) | MBR | GPT |
|----------------------|-----|-----|
| Create partition table | `msdos` | `gpt` |
| Create partition | Igen (primary / extended / logical logika) | Igen |
| Partition format | Igen | Igen |
| Partition type code (hex, MBR) | Igen | Nem |
| Partition type code (GUID, GPT) | Nem | Igen |
| Change partition name | Nem | Igen |
| GPT backup / restore / verify | Nem | Igen |
| GPT attributes | Nem | Igen |

---

## 5. Verzió

| Mező | Érték |
|------|--------|
| Dokumentum | MBR vs GPT + `partctl.sh` menüútmutató (MBR példa: **Windows XP**; GPT példa: **Windows 11 / `sda`**) |
| Forráskód hivatkozások | `partctl_ncurses_app/ui/split_menus.py`, `partition_table_views.py` (`draw_partition_table_type_menu`), `partition_create_ops.py` |

```markdown
[MBR vs GPT — Partctl útmutató](docs/mbr-gpt-partctl-guide.md)
```
