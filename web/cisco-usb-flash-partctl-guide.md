# Cisco kompatibilis USB flash és USB 3.0 SSD előkészítése — `partctl.sh` menüútmutató

![](/partctl/img/Cisco_logo_blue_2016.svg.png)

> **Cél:** Áttekinteni, **milyen fájlrendszert** és **milyen méretű** adathordozót **vár** a **Cisco** (Catalyst 9300 / 9200 / 3850 / 3650 IOS és IOS XE) az **IOS image** másolásához, **konfiguráció** mentéséhez vagy a beépített **USB 3.0 SSD** (pl. **SSD-240G**) használatához. Utána két **gyakorlati példa** a **Partctl** (`bash partctl.sh`) **menüpontjain** keresztül.

**Figyelem:** partíciós tábla, partíciók, **wipe** és **formázás** **adatvesztést** okoz a kiválasztott adathordozón. Mindig **mentett**, **leválasztott** (unmount) USB-vel dolgozz, **ne** a futó rendszerlemezen kísérletezz, és a **céllemezt** (pl. `/dev/sda`) a Partctl **Lemez attekintes** képernyőjén ellenőrizd a művelet előtt.

---

## 0. Gyors áttekintés — mit várunk az adathordozótól?

| Cisco eszköz / közeg | Partíciók száma | Ajánlott fájlrendszer | Tipikus méret | Partctl format opció (`Partition format`) |
|----------------------|-----------------|------------------------|----------------|--------------------------------------------|
| **USB flash drive** (Catalyst 9300 / 9200 / 3850 / 3650 IOS upgrade, konfiguráció) | **1 db** (egyetlen kötet) | **FAT32** (legacy: FAT16) | **16–32 GB** ajánlott (a forrás szerint stabil felismerés) | **`vfat`** |
| **Cisco USB 3.0 SSD** (pl. **SSD-240G**, Catalyst 9300) | **1 db** (egyetlen kötet) | **EXT4** (alternatíva: EXT3 / EXT2) | **240 GB** (hivatalosan támogatott modell) | **`ext4`** |
| **Nem támogatott** (Cisco USB 3.0 SSD-nél) | — | NTFS, exFAT, VFAT | — | Ne használd Cisco SSD-hez |

**Partíció-limitáció (fontos):** A Catalyst 9200 és a többi IOS / IOS XE switch **nem** kezeli megbízhatóan a **több partíciós** USB pendrive-ot — még akkor sem, ha minden partíció **FAT32**. A CLI általában **egyetlen** mount pontot ad (`usbflash0:`), ami az **első, felismerhető** kötetre mutat. **Ne** hozz létre `sda2`, `sda3` … partíciókat Cisco-célra; a nagyobb fizikai pendrive **maradék területét** hagyd **allokálatlanul**, ne második partícióval.

A FAT32 partíciós tábla típusa Linux-oldalon tipikusan **MBR (msdos)** — a régi USB-pendrive-ok jellemzően ezt használják. A Cisco USB 3.0 SSD (EXT4) tipikusan **GPT** táblán él (modern lemezgeometria, **>2 TiB** ugyan nem releváns 240 GB-nál, de a GPT a Linux-világban általános választás); itt is **egy** adatpartíció az ajánlott előkészítés.

---

## 1. Háttér — miért lényeges a fájlrendszer választás?

A Cisco IOS és IOS XE saját, **beágyazott** fájlrendszer-felismerő réteget használ. A `show file systems` és a `format usbflash1: …` parancsok mögött **nem** egy teljes Linux kernel áll — ezért a felismerés a **FAT32** (USB flash) és az **EXT4** (USB 3.0 SSD) kombinációkra van **finomhangolva**, és tipikusan **egy partícióra** (`usbflash0:`, `usbflash1:`). Egy „Linuxon működik” jelzésű adathordozó **nem** garancia arra, hogy a switch CLI is felismeri — különösen, ha **több partíció** van rajta (a switch a második FAT32 kötetet gyakran **egyáltalán nem** mountolja).

**Tipikus következmények rossz formázás esetén:**

1. **IOS image másolása sikertelen** — `%Error opening usbflash0:...` / `device not found` típusú üzenet.
2. **`show file systems`** nem listázza az USB-t, vagy `unknown` típussal.
3. **`dir usbflash0:`** üres listát ad pedig a kötet tele van — a switch nem tudja értelmezni a fájlrendszert (vagy **nem** azt a partíciót mountolja, amelyiken a fájlok vannak).
4. **Több FAT32 partíció** ugyanazon a pendrive-on — Linuxon `sda1` és `sda2` is látszik, de IOS alatt gyakran **csak az első** (vagy **egyik sem**) érhető el megbízhatóan; a `copy … usbflash0:` útvonal **egy** kötetre épül.
5. Bizonyos eszközök esetén a pendrive **flash drive detect** vagy **boot** folyamat véletlenszerűen sikerül vagy nem sikerül, különösen: **>32 GB** FAT32 pendrive-nál vagy ha **exFAT** / **NTFS** fájlrendszer van rajta.
6. **Cisco USB 3.0 SSD security lock** — a hardver lezárt állapotba kerül.

A `Partctl` ezeknek a **forráshibáit** a **partíció + fájlrendszer** szinten kezeli: tiszta lemez, jó tábla, **igazított** partíció, **Cisco-kompatibilis** formázás.

---

## 2. Cisco-támogatott fájlrendszerek és méretek (részletes)

### 2.0 Partíciók száma — **egyetlen kötet** (limitáció)

| Közeg | Hány partíció? | Miért? |
|-------|----------------|--------|
| **Külső USB flash** (9200 / 9300 / …) | **Pontosan 1** (pl. `sda1`) | Az IOS / IOS XE beágyazott fájlrendszer-rétege **nem** PC-s USB-partíció kezelő: a `usbflash0:` név **egy** FAT32 kötetre utal. A 2., 3., 4. partíció (még FAT32 is) **nem** használható megbízhatóan IOS image / config másolásra. |
| **Cisco USB 3.0 SSD** (SSD-240G) | **Pontosan 1** (pl. `sdc1`) | A firmware **EXT4**-re van hangolva **egy** adatpartícióval; több partíció Linux előkészítésből zavarhatja az IOS `format usbflash1:` folyamatot. |
| **Nagyobb pendrive** (>32 GB) | **1 partíció, korlátozott méret** | Pl. **~30 GiB** egyetlen `sda1`, a lemez többi része **unallocated** — **ne** második partícióval „kitölteni”. |

**Összehasonlítás:** Linux / Windows szabadon használ több partíciót egy USB-n; **Cisco switch nem**. A Partctl előkészítésnél is **egy** `Create partition` lépés + **egy** `Partition format` a Cisco-célú pendrive-ra.

### 2.1 USB flash drive (Catalyst 9300 / 9200 / 3850 / 3650)

| Tulajdonság | Érték | Megjegyzés |
|-------------|--------|-------------|
| **Partíciók száma** | **1 db** | **Kötelező gyakorlat:** egyetlen primary (`sda1`), FAT32. Több partíció **nem** támogatott IOS oldalon. |
| **Fő fájlrendszer** | **FAT32** | Az **IOS** és **IOS XE** ezt várja IOS image és konfiguráció transferhez. |
| **Legacy** | **FAT16** | Régebbi switch modelleknél. |
| **Nem ajánlott** | exFAT, NTFS | Az IOS image upgrade során **nem megbízhatóan** felismert. |
| **Nem ajánlott** | 2+ partíció (még FAT32 is) | A második kötet **nem** jelenik meg `usbflash1:` néven; a `copy` / `dir` parancsok **egy** kötetre számítanak. |
| **Ajánlott méret** | **16–32 GB** | A forrás szerint nagyobb pendrive intermittensen nem-felismert lehet. |
| **Partíciós tábla** | **MBR (msdos)** | Tipikus pendrive-okra ez van gyárilag; a Partctl is ezt javasolja `vfat` mellé. |
| **Cluster size** | **Default** (vagy 4096 B) | A `mkfs.vfat` alapértelmezett alloc unit elég. |
| **Partctl format** | **`vfat`** | A `Partition format` listából — **csak** az egyetlen adatpartícióra. |

### 2.2 Cisco USB 3.0 SSD (pl. SSD-240G — Catalyst 9300)

| Tulajdonság | Érték | Megjegyzés |
|-------------|--------|-------------|
| **Partíciók száma** | **1 db** | Egyetlen GPT-partíció (`sdc1`) + EXT4 — a stack CLI `usbflash1-2:` formátuma **tag/slot** jelölés, **nem** második partíció ugyanazon a sticken. |
| **Fő fájlrendszer** | **EXT4** | A Catalyst 9300 hivatalosan ezt várja a beépített USB SSD-n. |
| **Alternatíva** | EXT3, EXT2 | Támogatott, de legtöbbször a `format … ext4` IOS-parancs a default. |
| **Nem támogatott** | NTFS, exFAT, **VFAT** | A Cisco USB 3.0 SSD-n **nem** működnek. |
| **Méret** | **240 GB** | A hivatalos **SSD-240G** modul. |
| **Particiós tábla** | **GPT** | Modern lemezgeometria, Linux-oldali előkészítéshez logikus választás. |
| **Partctl format** | **`ext4`** | A `Partition format` listából. |

### 2.3 Nem támogatott kombinációk (gyakori csapdák)

| Eset | Probléma |
|------|----------|
| **Több partíció** ugyanazon USB flash-en (pl. `sda1` FAT32 + `sda2` FAT32) | IOS **nem** kezeli PC-szerűen: nincs megbízható `usbflash0:` / `usbflash1:` páros pendrive-partíciókhoz. A fájlok a „rossz” partíción maradhatnak, a switch **üres** kötetet lát. |
| **>32 GB FAT32 pendrive** Cisco IOS image-hez | Az IOS néhány verziója nem-felismertként kezeli vagy random hibákat ad. |
| **exFAT pendrive** Cisco IOS image-hez | IOS upgrade során **nem megbízható**, gyakran „unrecognized media”. |
| **NTFS pendrive** Cisco IOS image-hez | Hasonló helyzet — nem cél fájlrendszer az IOS-nek. |
| **NTFS / VFAT** a beépített **Cisco USB 3.0 SSD-n** | Cisco firmware **nem** kezeli; csak EXT-családból válassz. |
| **GPT táblás pendrive** Catalyst 9300 IOS upgrade-hez | Sok IOS verzió MBR típusú pendrive-ra van felkészítve — ha furcsa eredmény, váltsd `msdos`-ra. |
| **Nagy pendrive „kitöltése”** több partícióval | Helyette: **egy** ~30 GiB FAT32 partíció, maradék **unallocated** (lásd §5, lépés 4). |

---

## 3. Tipikus hibák és kiküszöbölésük (Cisco + Linux + Partctl szemszögből)

| Tünet (Cisco CLI) | Lehetséges ok | Javítás Partctl-lel |
|--------------------|----------------|----------------------|
| `Error opening usbflash0:` | Rossz fájlrendszer (exFAT / NTFS / üres) | **`3` Particio kezeles → Particio formazas → `vfat`** (FAT32) — pendrive-ra |
| `show file systems` nem mutatja az USB-t | Hibás MBR/GPT, fájlrendszer-aláírás roncsolt | **`4 → 10` Disk cleanup (Wipe)** + új **`Particios tabla letrehozasa`** (MBR) + új `vfat` |
| `dir usbflash0:` üres lista | Particio jó, de FAT verzió nem stimmel (FAT16 ↔ FAT32 a `mkfs` hívásnál) | Újra-formázás `vfat`-ra (`mkfs.vfat -F 32`-vel, ezt a Partctl gondozza) |
| `unknown filesystem` IOS-ben | Jó tábla, de NTFS / exFAT van rajta | **`Particio formazas → vfat`** vagy **`ext4`** (SSD-nél) |
| Cisco USB 3.0 SSD `not formatted` IOS-ben | Linuxról `ntfs` / `exfat` van rátéve | **`Particio formazas → ext4`** — tiszta EXT4 |
| `format usbflash1: ext4` IOS-ben sikertelen | A partícióban LVM, RAID superblock, GPT maradvány zavarja | **`4 → 10` Disk cleanup (Wipe)** **„signatures + partition table”** opcióval, majd újrahúzás Partctl-ből |
| Random felismerési hibák, nagy pendrive | **>32 GB** méret | Tegyél rá **egy partíciót csak 32 GB méretben** (a maradék hagyd allokálatlanul); `vfat` formázás |
| Linuxon `sda1` + `sda2` látszik, IOS-en üres / hibás `usbflash0:` | **Több partíció** FAT32-en | Wipe + **MBR** + **csak egy** primary + `vfat` — a 2. partíciót **töröld** vagy ne hozd létre |

**Tipp:** a hibákat **mindig** a Cisco CLI felületén nézd át: `show file systems`, `show media`, és csak utána a `format … :`.

---

## 4. Partctl előkészület (rövid emlékeztető)

```bash
bash partctl.sh
```

![](/partctl/img/terminal_1.jpg)

### Főmenü (rögzített sorszámok — minden nyelven ugyanaz)

| # | Angol | Magyar felületen |
|---|--------|---------------------|
| **1** | Select Disk | Lemez kivalasztasa |
| **2** | Disk Overview | Lemez attekintes |
| **3** | Partition management | Particio kezeles |
| **4** | Disk management | Lemez kezeles |
| **5** | Setup | Beallitasok |
| **6** | About | Rolunk |
| **7** | Exit | Kilepes |

![](/partctl/img/lemez_kivalasztasa_2.jpg)

**Navigáció:** `Fel` / `Le` (vagy `k` / `j`), **Enter**; vagy a sor elején látható **`N.`** szám begépelése, majd **Enter**. **Vissza:** **Backspace** / **`q`**.

A **Lemez kezeles** almenü **rögzített** sorrendben:

| # | Angol menüpont |
|---|----------------|
| 1–9 | EXT4 / repair / label / mount / unmount / NTFS műveletek |
| **10** | **Disk cleanup (Wipe)** |
| 11 | Back |

![](/partctl/img/wipe_1.jpg)

A **Particio kezeles** lista **ábécérendben** van — a konkrét sorszámot mindig a futó programban ellenőrizd, az alábbi útmutató a **menüpont címkéjét** használja.

---

## 5. Példa A — **16 GB USB flash drive** → **FAT32** (Catalyst 9200 / 9300 / 3850 / 3650)

**Cél:** Cisco IOS image (pl. `cat9k_iosxe.17.09.04.SPA.bin`) másolásához és konfiguráció backuphoz használható USB flash drive (`/dev/sda`, ~14,5 GiB szabad sáv).

**Partíció-szabály:** A példa **egyetlen** partíciót hoz létre (`sda1`). **Ne** futtasd újra a **Create partition** menüpontot második kötetre — a Catalyst 9200 (és általában az IOS / IOS XE) **nem** használja a `sda2` … partíciókat. Nagyobb fizikai pendrive esetén a **Vég:** `+30GiB` (vagy hasonló) **egy** partícióra vonatkozik; a lemez hátralévő sávját **ne** particionáld tovább.

| Lépés | Menüút (rövid) | Mit csinálsz |
|-------|----------------|--------------|
| 1 | **Főmenü → `1`** Select Disk / Lemez kivalasztasa | Kiválasztod a **`sda`** USB pendrive-ot (a `Lemez attekintes`-ben ellenőrizd, hogy valóban a céllemez — **ne** a rendszerlemez!). |
| 2 | **Főmenü → `4` → `10`** Disk cleanup (Wipe) | Cél: **whole disk (`/dev/sda`)**; jelöld be a **partíciós tábla törlés** opciót (Space) is — „tiszta lap”. Megerősítés. |
| 3 | **Főmenü → `3`** → **Create partition table** / **Particios tabla letrehozasa** | Tábla típus: **`2` — MBR (msdos)** (a klasszikus pendrive séma, Cisco IOS-barát). Megerősítés. |
| 4 | **Főmenü → `3`** → **Create partition** / **Particio letrehozasa** | **Egyszer** futtasd — eredmény: **`sda1`**. A **„Kezdet”** mezőnél **Enter** az alapértelmezett **`2048s`** értékre (1 MiB-igazítás). A **„Vég”** mezőnél: **`100%`** a teljes pendrive-ra (≤ 32 GB), vagy pl. **`+30GiB`** ha nagyobb a pendrive és csak 30 GB-ot szeretnél Cisco számára (**a maradékot allokálatlanul**, nem `sda2`-vel). |
| 5 | **Főmenü → `3`** → **Partition format** / **Particio formazas** | Cél: `sda1`; típus: **`vfat`** a listából. A Partctl a `mkfs.vfat -F 32 /dev/sda1` típusú parancsot futtatja (FAT32). |
| 6 | **Főmenü → `2`** Disk Overview / Lemez attekintes | Ellenőrzés: tábla **`dos` / `msdos`**, `sda1` méret, `FSTYPE` oszlop **`vfat`**. |
| 7 | *(Opcionális)* **Főmenü → `4`** Lemez kezeles → **Filesystem label** / **Fajlrendszer cimke** | Adj a kötetnek a Cisco-számára beszédes nevet, pl. **`CISCO_USB`**. |

**Eredmény (`/dev/sda` szempontból):**

| Eszköz | Tábla | Partíció | FS | Méret (példa) | Megjegyzés |
|--------|-------|----------|----|---------------|-------------|
| `sda` | `msdos` | — | — | 14,5 GiB | Pendrive |
| `sda1` | — | primary | **`vfat`** | ~14,5 GiB | **Egyetlen** Cisco IOS / konfiguráció kötet (`usbflash0:`) |
| *(nincs `sda2`)* | — | — | — | — | Második partíció **szándékosan nincs** — IOS nem támogatja megbízhatóan |

| Lemez áttekintés | Particio reszletei |
| --- | --- |
| ![cisco_fat32_1](/partctl/img/cisco_fat32_1.jpg "Cisco FAT32 #1") | ![cisco_fat32_2](/partctl/img/cisco_fat32_2.jpg "Cisco FAT32 #2") |

**Csatlakoztatás után a switch oldali ellenőrzés** (lásd §7):

```text
Switch# show file systems
Switch# dir usbflash0:
```

Ha a `usbflash0:` látszik és a `dir` üres tartalmat ad — **kész** a kötet, jöhet az IOS upgrade.

---

## 6. Példa B — **240 GB USB 3.0 SSD (Cisco NVME SSD 240GB)** → **EXT4** (Catalyst 9300)

**Cél:** A Catalyst 9300 beépített USB 3.0 SSD modul (`SSD-240G`) előkészítése IOS oldali `format usbflash1: ext4` előtt — Linuxról **tiszta lap**, **GPT tábla**, **egyetlen EXT4** kötet. Példa eszköznév: `/dev/sdc`, ~223,6 GiB.

**Partíció-szabály:** Hasonlóan a pendrive-hoz: **egy** adatpartíció (`sdc1`). A stack parancs `format usbflash1-2: ext4` a **2. switch tag** slotját jelöli, **nem** a lemez második partícióját.

| Lépés | Menüút (rövid) | Mit csinálsz |
|-------|----------------|--------------|
| 1 | **Főmenü → `1`** Select Disk | Kiválasztod a **`sdc`** SSD-t (a `Lemez attekintes`-ben ellenőrizd a modell/méret párost). |
| 2 | **Főmenü → `4` → `10`** Disk cleanup (Wipe) | Cél: **whole disk (`/dev/sdc`)**; jelöld be a **partíciós tábla törlés** opciót — a régi GPT/LUKS/LVM aláírások biztos eltűnnek, így a Cisco felismerés nem akad meg. |
| 3 | **Főmenü → `3`** → **Create partition table** / **Particios tabla letrehozasa** | Tábla típus: **`1` — GPT**. Megerősítés. |
| 4 | **Főmenü → `3`** → **Create partition** / **Particio letrehozasa** | **Kezdet:** **Enter** a javasolt **`2048s`** értékre (1 MiB-igazítás). **Vég:** **`100%`** — a Partctl mostantól a vég oldalon is **1 MiB**-os védőkeretet hagy és **2048-szektoros** boundary-ra igazít (lásd [`particio-igazitas-partctl-guide.md`](particio-igazitas-partctl-guide.md) §3), így a `sgdisk -v` is csendes lesz ("GPT tabla ellenorzes" menü pont). |
| 5 | **Főmenü → `3`** → **Partition format** / **Particio formazas** | Cél: `sdc1`; típus: **`ext4`** a listából. A Partctl `mkfs.ext4` hívást futtat — a modul felugró kérdéseit (pl. 64bit feature) fogadd el az alapértelmezetten. |
| 6 | **Főmenü → `3`** → **Verify GPT partition table** | Ellenőrzés: **GPT OK**, nincs „doesn't end on a 2048-sector boundary” figyelmeztetés (ha mégis, az igazítás újratervezése szükséges — lásd `particio-igazitas-partctl-guide.md` §3). |
| 7 | **Főmenü → `2`** Disk Overview / Lemez attekintes | Tábla **`gpt`**, `sdc1` méret, `FSTYPE` oszlop **`ext4`**. |
| 8 | *(Opcionális)* **Filesystem label** | Adj címkét, pl. **`CISCO_SSD`** — Linux-oldalon segít a kötet azonosításában. |

**Eredmény (`/dev/sdc` szempontból):**

| Eszköz | Tábla | Partíció | FS | Méret (példa) | Megjegyzés |
|--------|-------|----------|----|---------------|-------------|
| `sdc` | `gpt` | — | — | 223,6 GiB | Cisco SSD-240G |
| `sdc1` | — | (#1) | **`ext4`** | ~223,6 GiB | Cisco USB 3.0 SSD kötet |

| Lemez áttekintés | Particio reszletei |
| --- | --- |
| ![cisco_ext4_1](/partctl/img/cisco_ext4_1.jpg "Cisco EXT4 #1") | ![cisco_ext4_2](/partctl/img/cisco_ext4_2.jpg "Cisco EXT4 #2") |

**Switch oldal — a Catalyst 9300 saját CLI-jén** (ha a Cisco firmware újra-formázást is végez, ezt fogadd el):

```text
Device# format usbflash1: ext4
```

Stack tag esetén a member ID-vel együtt:

```text
Device# format usbflash1-2: ext4
```

---

## 7. Ellenőrzés Cisco CLI-ből

A Partctl-előkészítés után a switch oldalon az alábbi parancsok adnak gyors visszajelzést:

```text
Switch# show file systems            ! listázza a felismert fájlrendszereket
Switch# dir usbflash0:                ! pendrive tartalom (FAT32)
Switch# dir usbflash1:                ! Cisco USB 3.0 SSD (EXT4)
Switch# show media                    ! média / mount állapot
Switch# copy usbflash0:cat9k_iosxe.17.09.04.SPA.bin flash:
```

**SSD-specifikus parancsok** (security state, unlock, unmount):

```text
Device# show hw-module usbflash1 security status
Device# hw-module switch 1 usbflash1 security unlock password
Device# hw-module switch 1 usbflash1 unmount
```

Ha a `show file systems` **nem** mutatja az USB-t, a leggyakoribb okok és lépések:

1. **Pendrive újrahúzása** — fizikai csatlakoztatás-újrapróbálás (`show usb device`).
2. **Visszamenni Linuxra**, **Partctl `Disk cleanup (Wipe)`** + új **MBR/GPT** + új formázás (§5 / §6).
3. **Másik pendrive** — pendrive-csere a hiba beazonosításához.
4. **IOS verzió ellenőrzés** — egyes régebbi IOS image-ek szűkebb listát fogadnak el.

---

## 8. Best practices (gyakorlati tanácsok)

- **Mentés mindig** a pendrive / SSD adatairól a formázás **előtt** — a Partctl `Disk cleanup (Wipe)` **nem** visszavonható.
- **Egy partíció** Cisco-célra — **soha** ne készíts `sda2` / `sda3` … kötetet ugyanarra a pendrive-ra, még FAT32-vel sem; az IOS **egy** `usbflash0:` kötetet vár.
- **FAT32** USB flash-hez Catalyst 9300 / 9200 / 3850 / 3650 esetén; **EXT4** a beépített Cisco USB 3.0 SSD-hez (`SSD-240G`).
- **Pendrive méret:** **16–32 GB** stabilabb felismerés. Ha nagyobb a pendrive, **egy** partíció ~30 GB-ig (**Vég:** `+30GiB`) — a maradék **unallocated**, nem második partíció.
- **Cisco USB 3.0 SSD unmount** a switch-en **mindig** kihúzás előtt (`hw-module … unmount`) — különben a Partctl-en hibás aláírást találhatsz.
- **Egyértelmű címke** (`Filesystem label`) — `CISCO_USB`, `CISCO_SSD`, vagy site-specifikus prefix (pl. `DC1_C9300_USB`).
- **Egy adott Partctl session — egy adott céllemez:** a futtatás során a `Lemez attekintes` panelen mindig ellenőrizd, hogy a kiválasztott `sdX` valóban a Cisco-céllemez.

---

## 9. Verzió

| Mező | Érték |
|------|--------|
| Dokumentum | Cisco USB flash + USB 3.0 SSD előkészítés `partctl.sh`-val (FAT32 pendrive és EXT4 SSD példa) |
| Cisco modellek | Catalyst 9300 / 9200 / 3850 / 3650, Cisco USB 3.0 SSD (SSD-240G) |


```markdown
https://github.com/drcyberg/partctl/blob/main/example/cisco-usb-flash-partctl-guide.md
```

### Fő oldal (Partctl)

- [Partctl](https://drcyberg.github.io/partctl/web/partctl)

### Köszönöm ha támogatsz

- ***Buy me a coffee***: [LINK](https://buymeacoffee.com/drcyberg)
- ***Paypal (QR Code)***: [LINK](https://github.com/drcyberg/partctl/blob/main/img/qrcode.png)
- ***Paypal (URL)***: [LINK](https://paypal.me/Kunee82)

*Utolsó frissítés jelleg: Partctl V1.0.0 viselkedés — **Cisco USB flash: 1 db partíció (FAT32, MBR)**; a **Particio kezeles** lista ábécérendje miatt a konkrét **sorszámok** mindig a futó programban ellenőrizendők.*
