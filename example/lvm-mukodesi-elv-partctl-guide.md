# LVM működési elv — összefoglaló és `partctl.sh` menüútmutató

> **Cél:** Rövid, jól értelmezhető háttér a **Linux LVM** (Logical Volume Manager) rétegeiről, a **thick** és **thin** különbségéről, és arról, hogyan hozhatók létre / kezelhetők a kötetek a **Partctl** menüiből (`bash partctl.sh`).  

> **Figyelem:** LVM és particiós műveletek **adatvesztést** okozhatnak. Csak **mentett**, **nem** futó rendszerlemezen, **leválasztott** (unmount) célokon dolgozz; éles környezetben mindig **biztonsági mentés**.

---

<p align="center">
  <img src="/img/lvm_mukodesi_elv_1.png" alt="LVM működési elv — parancsok és rétegek" width="92%" />
</p>

---

## 1. Mi az LVM?

Az **LVM (Logical Volume Manager)** a Linux rendszerek hatékony, rugalmas lemezkezelő eszköze, amely absztrakciós réteget képez a fizikai merevlemezek és a fájlrendszer között. Lehetővé teszi a partíciók dinamikus átméretezését, több lemez összevonását, és snapshotok készítését a rendszer leállítása nélkül.
Tehát az LVM a **Linux partíciók fölött** egy **logikai** réteget épít. A következők szerint épül fel:

| Réteg | Rövidítés | Mit jelent? | Tipikus parancs |
|-------|-----------|-------------|-----------------|
| **Fizikai kötet** | **PV (Physical Volume)** | Egy partició (vagy lemez), amit az LVM „befogad” | `pvcreate` |
| **Kötetcsoport** | **VG (Volume Group)** | A PV-k összevonva: egy nagy „tárolómedence” | `vgcreate` |
| **Logikai kötet** | **LV (Logical Volume)** | A VG-ből kivágott, használható blokk-eszköz (`/dev/vg/lv`) | `lvcreate` |
| **Fájlrendszer** | **FS (File System)** | ext4, NTFS, xfs stb. az LV **fölött** | `mkfs.*`, Partctl **Particio formazas** |

**Fontos:** Az LVM **nem** helyettesíti a particiós táblát (GPT/MBR). Kell egy **Linux partíció** (vagy egész lemez PV-ként), **utána** jön az LVM.

---

## 2. Thick vs thin — melyik mikor?

### 2.1 Mikor érdemes **thick**-et használni?

A **Thick kötetek** esetében az LVM a logikai kötet létrehozásakor **azonnal** lefoglalja és dedikálja a kért fizikai tárhelyet (merevlemez-kapacitást). Ez azt jelenti, hogy a lemezcímzés közvetlen, a virtuális méret megegyezik a ténylegesen elfoglalt fizikai mérettel.

| Környezet / cél | Miért thick? |
|-----------------|--------------|
| **Szerver — rendszer- és adatkötet** (`/`, `/var`, `/home`, adatbázis-kötet) | Kiszámítható hely; a `df` és az LVM **egyezik**; kevesebb „pool tele” meglepetés. |
| **Egy nagy, stabil kötet** egy partición | Egyszerű bővítés: `lvextend` + fájlrendszer-növelés (Partctl: **LV méret növelése**). |
| **Hagyományos LVM snapshot** (mentés előtti pillanatfelvétel) | A klasszikus **COW snapshot** (`lvcreate -s`) **thick** eredeti LV-re épül — szervereken gyakori minta. |
| **Kevés kötet, hosszú élettartam** | Kevesebb metaadat, kevesebb thin-pool karbantartás. |
| **Partctl gyakorló / első LVM** | A **LVM-thick kotet letrehozasa** menüpont a legegyszerűbb teljes út. |

**Szerver példa (thick + snapshot gondolat):** adatbázis vagy fájl-szerver kötet (`/dev/vg0/data`). Mentés előtt **olvasható pillanatkép** készítése úgy, hogy az eredeti LV-ről **snapshot LV** jön létre a VG szabad területéből, a snapshot ideiglenesen mountolható vagy `dd` / `rsync` forrás lehet — az eredeti kötet közben írható maradhat (COW: a változások a snapshot területére kerülnek).

### 2.2 Mikor érdemes **thin**-t használni?

Az **LVM Thin** (vagy LVM Thin Provisioning) egy fejlett tárolókapacitás-kezelési technológia, amely lehetővé teszi, hogy a logikai kötetek (Logical Volumes) virtuális mérete nagyobb legyen, mint a fizikai háttértároló tényleges kapacitása. A tárhely kiosztása nem előre történik, hanem **dinamikusan**, igény szerint.

| Környezet / cél | Miért thin? |
|-----------------|-------------|
| **Virtuális gépek (KVM/QEMU, néhány hypervisor)** | Sok **független „lemez”** (VDI) egy fizikai partición; a hely **csak akkor** foglalódik, amikor a VM ír. |
| **Konténerek, fejlesztői környezetek** | Gyorsan sok kis kötet; gyakori klónozás / törlés. |
| **Tárhely kiszolgáló jelleg** (sok logikai unit egy medencében) | A pool egy helyen koncentrálja a fizikai helyet; a thin LV-k **virtuálisan nagyok** lehetnek. |
| **Thin snapshot** (pool szinten) | Modern LVM: **thin snapshot** a thin poolhoz kötődik — sok VM pillanatfelvétele **költséghatékonyabban**, mint ugyanannyi teljes thick másolat. |

**Mikor *ne* thin legyen az első választás:** kritikus **egyetlen** szerverkötet (pl. gyökér vagy adatbázis), ha a csapat nem akar **pool telítettség** monitorozást; ha régi eszközök / backup szoftver nem ismeri a thin LV-t; ha **nincs** terv a pool és a metaadat terület figyelésére.

### 2.3 Snapshot — szerver környezetben

Az LVM thin pillanatkép (snapshot) egy adott logikai kötet pillanatnyi állapotának azonnali, **tárhely hatékony másolata**, amely a thin provisioning technológiát használja. Ideális virtuális gépek (pl. Proxmox, KVM), tárolók és rendszerek biztonsági mentéséhez, mivel nem foglal feleslegesen előre lemezterületet.

| Típus | Mire való? | Hol „ül”? | Tipikus szerver használat |
|-------|------------|-----------|---------------------------|
| **Thick snapshot (COW)** | Egy **thick LV** írásvédett másolata pillanatnyilag | Külön **snapshot LV** a **VG**-ben (helyet foglal!) | Éjszakai backup, konzisztens fájlrendszer-kép **csatolás nélküli** másolathoz, frissítés előtti visszaállítási pont |
| **Thin snapshot** | Thin LV **pillanata** a **thin pool**-on belül | A pool **szabad** és **meta** kapacitásából | Sok VM / sok kötet, gyakori pillanatfelvételek, klónok |

**Fontos különbségek:**

- **Thick snapshot:** a VG-ben kell **elég szabad extent** a snapshot LV-nek. Ha a VG tele van, **nem** jön létre snapshot.
- **Thin snapshot:** a **pool telítettségét** (`Data%` / `lvs`) kell figyelni.

**Partctl kapcsolat:** thick / thin **létrehozás**, bővítés, törlés, formázás menükben elérhető.

### 2.4 Korlátozások és kockázatok

#### Thick — tipikus limitációk

| Limitáció | Magyarázat |
|-----------|------------|
| **Előre lefoglalt hely** | Nem lehet „túl nagyot mutatni” a kötetnek anélkül, hogy a VG-ben tényleg ott lenne a hely. |
| **VG szabad hely** | Új LV vagy **snapshot** csak akkor, ha maradt extent a csoportban. |
| **Méret csökkentés** | Nehezebb: fájlrendszer zsugorítás + `lvreduce` (adatvesztés-kockázat, offline igény). |
| **Sok kis kötet** | Minden LV külön foglal — sok esetben **pazarlóbb**, mint egy megosztott thin pool. |
| **Áthelyezés / mirroring** | `pvmove`, `lvconvert` működik, de tervezést igényel (Partctl: főleg PV/VG/LV menük). |

#### Thin — tipikus limitációk

| Limitáció | Magyarázat |
|-----------|------------|
| **Pool telítettség** | Ha a **thin pool** (data + meta) megtelik, **minden** thin LV azonnal érintett lehet — akár írási hiba, akár sérült metaadat. |
| **Over-provisioning** | A thin LV-k **virtuális** összmérete **nagyobb** lehet, mint a pool fizikai mérete → későbbi „hirtelen tele” veszély. |
| **Metaadat terület** | A poolnak külön **meta** LV-re is szüksége van; kicsi / tele meta → thin műveletek hibáznak. |
| **Bonyolultabb üzemeltetés** | `lvs`, `lvs -o+lv_pool_lv_size,lv_metadata_size,data_percent` jellegű figyelés szükséges. |
| **Nem minden eszköz / régi kernel** | Ritka ma, de egyes régi backup vagy speciális boot környezetek thin-t nem támogatnak jól. |
| **Pool zsugorítás** | Gyakorlatilag **nem** triviális; új pool + migráció tervezett feladat. |

### 2.5 Összehasonlító táblázat

| | **Thick** | **Thin** |
|---|-----------|----------|
| Lépések (Partctl) | PV → VG → **LV** | PV → VG → **thin pool** → **thin LV** |
| Méret | VG szabad helyének ~100%-a | Pool = VG szabad; thin LV virtuális méret (pl. 100% a poolé) |
| Helyfoglalás | Azonnal a VG-ből | Igény szerint a poolból |
| Snapshot (szerver) | **COW snapshot** thick LV-n | **Thin snapshot** a poolon |
| Tipikus használat | Szerver adat, DB, `/var`, backup előtti kép | VM, sok LV, fejlesztői / VDI jelleg |
| Fő kockázat | VG tele → nincs új LV / snapshot | Pool tele → **összes** thin LV veszélyben |

---

## 3. Előfeltételek

1. **Root / sudo** — az LVM parancsok rendszergazdai jogot igényelnek (a `partctl.sh` is).
2. **Csomagok:** `lvm2` (és a Partctl által használt `pvcreate`, `vgcreate`, `lvcreate`, `lvs`, …). Telepítés: `bash setup.sh` → **Ellenőrzés / Telepítés**.
3. **Partíció:** a cél lemezen legyen **legalább egy szabad partició** (vagy üres lemez GPT/MBR táblával), pl. Linux típus (`8300` / `83`).
4. **Nincs csatolva:** a cél partició / LV **ne legyen mountolva**; a Partctl kérésre segít leválasztani.
5. **Nincs dupla PV:** ugyanaz a partició **ne legyen már** PV — ellenkező esetben előbb **Wipe** vagy **PV törlés**.

---

## 4. Közös előkészület — `partctl.sh`

```bash
bash partctl.sh
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

**Navigáció:** `Fel` / `Le` (vagy `k` / `j`), **Enter**; vagy a sor elején látható szám + **Enter**. **Vissza:** **Backspace** / **`q`**.

**Minden LVM-varázsló előtt:**

1. **Főmenü → `1`** — válaszd ki a céllemezt (pl. **`sdc`**).
2. Ellenőrzés: **Főmenü → `2`** — **Lemez attekintes**; LVM soroknál **Enter** → jobb oldali **LVM reszletek** panel (ha van metaadat).

### Particio kezeles → LVM muveletek

A **Particio kezeles** lista **ábécérendben** van — a **LVM muveletek** sorszáma **változhat**. Mindig a **képernyőn** látható számot / címkét használd.

**Útvonal:**

```text
Főmenü → 3 (Particio kezeles) → LVM muveletek
```

### LVM muveletek almenü (rögzített sorrend)

| # | Menüpont (magyar) | Szerep |
|---|-------------------|--------|
| 1 | **LV kotet kezeles** | LV lista; **kézi** thick / thin LV; átnevezés, törlés, méret növelés/csökkentés |
| 2 | **LVM-thick kotet letrehozasa** | **Automatikus** thick: PV + VG + LV egy varázslóban |
| 3 | **LVM-thin kotet letrehozasa** | **Automatikus** thin: PV + VG + thin pool + thin LV (data) |
| 4 | **PV kotet kezeles** | **Kézi** út 1. lépése: PV lista, `pvcreate`, `pvremove` |
| 5 | **VG kotet kezeles** | **Kézi** út 2. lépése: VG lista, `vgcreate`, bővítés, aktiválás, törlés |
| 6 | **Vissza** | Vissza a partició menübe |

![](/img/lvm_muvelet_1.jpg)

---

## 5. Automatikus elnevezések (Partctl)

A program **egységes, felismerhető** LVM-neveket javasol / használ (a konkrét partició neve alapján, pl. `sdc1`):

| Elem | Példa név | Megjegyzés |
|------|-----------|------------|
| **VG** | `sdc1_partctl_vg` | Egy partició → egy VG (automatikus varázslók) |
| **Thick LV** | `thick` | VG-n belül: `/dev/sdc1_partctl_vg/thick` |
| **Thin pool** | `sdc1_partctl_thinpool` | Thin pool LV |
| **Thin data LV** | `sdc1_partctl_thinpool_data` | Használható kötet (virtuális partíció) |

- A **thin** út a fenti `*_partctl_vg` / `*_partctl_thinpool` konvenciót követi.
- A **thick** varázsló más mintát is használhat (`partctl_sdc1_thick_vg` + `thick`)

---

## 6. LVM-thick kötet létrehozása

<p align="center">
  <img src="/img/lvm_thick_1.png" alt="LVM működési elv — henger diagram (thick)" width="92%" />
</p>

> **Cél:** egy partició (pl. `sdc1`) → PV → VG → egy thick LV → később formázás.

| Módszer | Mikor érdemes? | Menü |
|---------|----------------|------|
| **Automatikus** (§6.1) | Egy partició, mindent egyszerre | **LVM-thick kotet letrehozasa** (`2`) |
| **Kézi** (§6.2) | Saját VG/LV név, lépésenkénti ellenőrzés, több PV később | **PV** (`4`) → **VG** (`5`) → **LV** (`1`) |

### 6.1 Automatikus létrehozás (varázsló)

| Lépés | Menüút | Mit csinálsz |
|-------|--------|--------------|
| 1 | **Főmenü → `1`** | Lemez kiválasztása (pl. `sdc`). |
| 2 | *(Ha nincs partició)* | **Particio kezeles → Particio letrehozasa** — egy teljes vagy tetszőleges partició a szabad sávra (Linux típus). |
| 3 | **Főmenü → `3` → LVM muveletek → `2`** | **LVM-thick kotet letrehozasa**. |
| 4 | Varázsló panel | Válaszd ki a **cél particiót** (pl. `sdc1`). |
| 5 | Megerősítés panel | Ellenőrizd: PV, VG név, LV név (`thick`) — **sárga** megerősítő panel. |
| 6 | **Folyamat** | Kék színű panel, **2 fázis:** (1) PV + VG, (2) LV létrehozás. Várj a végéig. |
| 7 | **Info** | zöld színű összefoglaló panel: siker + teljes parancslánc. |
| 8 | **Főmenü → `3` → Particio formazas** | Válaszd az LV-t (pl. `/dev/sdc1_partctl_vg/thick` vagy mapper útvonal), **ext4** / **ntfs** stb. |
| 9 | *(Opcionális)* **Főmenü → `4` → Ideiglenes csatolás** | Csak teszthez; éles szerveren állandó `fstab` külön téma. |

| Lemez áttekintés | Partíció részletei |
| --- | --- |
| ![lvm-thick_2](/img/lvm-thick_2.jpg "LVM-thick #2") | ![lvm-thick_3](/img/lvm-thick_3.jpg "LVM-thick #3") |

**Háttérben (automatikus thick), tipikus parancsok:**

```text
pvcreate -ff -y /dev/sdc1
vgcreate sdc1_partctl_vg /dev/sdc1
lvcreate -y -W y -l 100%FREE -n thick sdc1_partctl_vg
```

### 6.2 Kézi létrehozás (PV → VG → LV menük)

Ugyanaz a végeredmény, de **három külön menüben** állítod össze a rétegeket. A Partctl minden lépésnél **sárga** megerősítő panelt mutat, majd kék **Folyamat** panelt futtat.

**Előfeltétel:** kiválasztott lemez (`Főmenü → 1`), **leválasztott** cél partició (pl. `sdc1`), nincs rajta fájlrendszer / nincs csatolva.

| Lépés | Menüút | Mit csinálsz |
|-------|--------|--------------|
| 1 | **Főmenü → `1`** | Lemez: pl. `sdc`. |
| 2 | *(Ha kell)* **Particio kezeles → Particio letrehozasa** | Egy Linux partició (pl. `sdc1`) a szabad sávra. |
| 3 | **LVM muveletek → `4` → `2`** | **PV kotet kezeles** → **PV letrehozas (pvcreate)**. |
| 4 | PV varázsló | Táblázatból válaszd a particiót (`/dev/sdc1`). **Enter** → **Letrehozzam most a PV-t?** → Igen. |
| 5 | **Folyamat + Info** | `pvcreate -ff -y /dev/sdc1` — várj a zöld színű Info panelig. |
| 6 | **LVM muveletek → `5` → `2`** | **VG kotet kezeles** → **VG letrehozas** (ha a menüben más a szöveg: VG létrehozása). |
| 7 | VG varázsló | Válaszd a **szabad PV**-t (`/dev/sdc1`, nincs VG-hez rendelve). |
| 8 | VG név | Alapértelmezett: `sdc1_partctl_vg` — átírható. **Enter** → **sárga** megerősítés (`vgcreate …`). |
| 9 | **Folyamat + Info** panel | `vgcreate sdc1_partctl_vg /dev/sdc1`. |
| 10 | **LVM muveletek → `1` → `2`** | **LV kotet kezeles** → **LV letrehozas (lvcreate)**. |
| 11 | VG választás | Válaszd a friss VG-t (`sdc1_partctl_vg`). |
| 12 | Pool típus | **LVM thick** (ne a thin). |
| 13 | LV név | Alapértelmezett: `sdc1_partctl_lv` — átírható (pl. `thick`). |
| 14 | Megerősítő panel | **100%FREE** — a program a teljes szabad VG-területet használja. **sárga** panel → Igen. |
| 15 | **Folyamat + Info** panel | `lvcreate -y -l 100%FREE -n <lv> sdc1_partctl_vg`. |
| 16 | **Particio formazas** | Cél LV: pl. `/dev/sdc1_partctl_vg/sdc1_partctl_lv` vagy mapper útvonal. |

**Háttérben (kézi thick), tipikus parancsok** — megegyeznek az automatikus első két lépésével; az LV név a 13. lépésben megadott:

```text
pvcreate -ff -y /dev/sdc1
vgcreate sdc1_partctl_vg /dev/sdc1
lvcreate -y -l 100%FREE -n sdc1_partctl_lv sdc1_partctl_vg
```

> **Megjegyzés:** a kézi thick LV létrehozás jelenleg **mindig** `100%FREE` méretet kér; részleges méretet a **LV méret növelése / csökkentése** menük adják később.

---

## 7. LVM-thin kötet létrehozása (thin pool + data / „virtuális partíció”)

<p align="center">
  <img src="/img/lvm_thin_1.png" alt="LVM-thin működési elv" width="92%" />
</p>

> **Cél:** egy partició → PV → VG → **thin pool** → **thin LV** (data) — használható „virtuális partíció”.

### 7.1 Automatikus létrehozás (varázsló)

| Lépés | Menüút | Mit csinálsz |
|-------|--------|--------------|
| 1–2 | Ugyanaz, mint thick-nél | Lemez + **szabad partició** (`sdc1`). |
| 3 | **LVM muveletek → `3`** | **LVM-thin kotet letrehozasa**. |
| 4 | Varázsló | Cél partició; megerősítés: PV, VG, **thin pool**, **thin LV** (virtuális 100%). |
| 5 | **Folyamat** | **3 fázis:** (1) PV + VG, (2) thin pool, (3) thin LV (data). **Egy** kék színű folyamat panelon megjelenítve, majd **egy** záró zöld színű Info panelon az eredmény megjelenítve. |
| 6 | **Particio formazas** | Cél: a **data** thin LV (pl. `.../sdc1_partctl_thinpool_data`). |
| 7 | **Lemez attekintes** | Ellenőrizd: pool + data LV, méret, mapper útvonalak. |

| Lemez áttekintés | Partíció részletei |
| --- | --- |
| ![lvm-thin_2](/img/lvm-thin_2.jpg "LVM-thin #2") | ![lvm-thin_3](/img/lvm-thin_3.jpg "LVM-thin #3") |

**Háttérben (automatikus thin), tipikus parancsok:**

```text
pvcreate -ff -y /dev/sdc1
vgcreate sdc1_partctl_vg /dev/sdc1
lvcreate -y --type thin-pool -l 100%FREE -n sdc1_partctl_thinpool sdc1_partctl_vg
lvcreate -y -V <pool_meret>B -T /dev/sdc1_partctl_vg/sdc1_partctl_thinpool -n sdc1_partctl_thinpool_data
```

> A **virtuális méret** (`-V`) a Partctl a pool aktuális méretéből számolja (100% modell).

### 7.2 Kézi létrehozás — teljes út (PV → VG → thin pool + data)

A **6.2** lépései **1–9** (partíció, PV, VG) **megegyeznek**; innen folytatod:

| Lépés | Menüút | Mit csinálsz |
|-------|--------|--------------|
| 10 | **LVM muveletek → `1` → `2`** | **LV kotet kezeles** → **LV letrehozas (lvcreate)**. |
| 11 | VG választás | Pl. `sdc1_partctl_vg`. |
| 12 | Pool típus | **LVM thin** (ne a thick). |
| 13 | Thin-pool név | Alapértelmezett: `sdc1_partctl_thinpool` — átírható. |
| 14 | Megerősítő panel | **100%FREE** thin pool — **sárga** panel → Igen. |
| 15 | **Folyamat (2 fázis)** | (1) thin pool létrehozás, (2) **data** thin LV — **egy** kék színű folyamat panelon megjelenítve, majd **egy** záró zöld színű Info panelon az eredmény megjelenítve. |
| 16 | Data LV név | A program automatikusan: `sdc1_partctl_thinpool_data` (a PV partició nevéből). |
| 17 | **Particio formazas** | A **data** LV-re formázol (ne a pool-ra). |
| 18 | **Lemez attekintes** | Két LV: pool + data; a data a használható kötet. |

**Háttérben (kézi thin), tipikus parancsok:**

```text
pvcreate -ff -y /dev/sdc1
vgcreate sdc1_partctl_vg /dev/sdc1
lvcreate -y --type thin-pool -l 100%FREE -n sdc1_partctl_thinpool sdc1_partctl_vg
lvcreate -y -V <pool_meret>B -T /dev/sdc1_partctl_vg/sdc1_partctl_thinpool -n sdc1_partctl_thinpool_data
```

### 7.3 Kézi létrehozás — csak meglévő VG-n (thin pool + data)

Ha a **PV** és **VG** már létezik (pl. `vgextend`-del bővített kötetcsoport), elég az LV menü:

| Lépés | Menüút | Mit csinálsz |
|-------|--------|--------------|
| 1 | **LVM muveletek → `1` → `2`** | **LV kotet kezeles** → **LV letrehozas**. |
| 2 | VG | Válaszd a meglévő VG-t (legyen **szabad PE** a poolhoz). |
| 3 | **LVM thin** | Thin-pool név → **sárga** megerősítés (`100%FREE` a szabad VG-területre). |
| 4 | **Folyamat** panel | **2 fázis:** thin pool, majd data thin LV (virtuális méret = pool mérete). |
| 5 | **Info** | zöld színű összefoglaló panel a teljes parancslánccal. |
| 6 | **Particio formazas** | Cél: a **data** thin LV. |

```markdown
https://github.com/drcyberg/partctl/blob/main/example/lvm-mukodesi-elv-partctl-guide.md
```

### Fő oldal (Partctl)

- [Partctl](https://drcyberg.github.io/partctl/web/partctl)

### Köszönöm ha támogatsz

- ***Buy me a coffee***: [LINK](https://buymeacoffee.com/drcyberg)
- ***Paypal (QR Code)***: [LINK](https://github.com/drcyberg/partctl/blob/main/img/qrcode.png)
- ***Paypal (URL)***: [LINK](https://paypal.me/Kunee82)

*Utolsó frissítés jelleg: Partctl V1.0.0 viselkedés — a **Particio kezeles** lista ábécérendje miatt a konkrét **sorszámok** mindig a futó programban ellenőrizendők.*
