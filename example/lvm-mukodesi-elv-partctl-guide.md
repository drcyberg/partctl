# LVM működési elv — összefoglaló és `partctl.sh` menüútmutató

> **Cél:** Rövid, jól értelmezhető háttér a **Linux LVM** (Logical Volume Manager) rétegeiről, a **thick** és **thin** különbségéről, és arról, hogyan hozhatók létre / kezelhetők a kötetek a **Partctl** menüiből (`bash partctl.sh`).  

> **Figyelem:** LVM és particiós műveletek **adatvesztést** okozhatnak. Csak **mentett**, **nem** futó rendszerlemezen, **leválasztott** (unmount) célokon dolgozz; éles környezetben mindig **biztonsági mentés**.

---

<p align="center">
  <img src="/img/lvm_mukodesi_elv_1.png" alt="LVM működési elv — parancsok és rétegek" width="92%" />
</p>

---

## 1. Mi az LVM? — négy réteg, egy mondatban

Az **LVM (Logical Volume Manager)** a Linux rendszerek hatékony, rugalmas lemezkezelő eszköze, amely absztrakciós réteget képez a fizikai merevlemezek és a fájlrendszer között. Lehetővé teszi a partíciók dinamikus átméretezését, több lemez összevonását, és snapshotok készítését a rendszer leállítása nélkül.
Tehát az LVM a **Linux partíciók fölött** egy **logikai** réteget épít. A következők szerint épül fel:

| Réteg | Rövidítés | Mit jelent? | Tipikus parancs |
|-------|-----------|-------------|-----------------|
| **Fizikai kötet** | **PV (Physical Volume)** | Egy partició (vagy lemez), amit az LVM „befogad” | `pvcreate` |
| **Kötetcsoport** | **VG (Volume Group)** | A PV-k összevonva: egy nagy „tárolómedence” | `vgcreate` |
| **Logikai kötet** | **LV (Logical Volume)** | A VG-ből kivágott, használható blokk-eszköz (`/dev/vg/lv`) | `lvcreate` |
| **Fájlrendszer** | — | ext4, NTFS, xfs stb. az LV **fölött** | `mkfs.*`, Partctl **Particio formazas** |

**Fontos:** Az LVM **nem** helyettesíti a particiós táblát (GPT/MBR). Kell egy **Linux partíció** (vagy egész lemez PV-ként), **utána** jön az LVM.

---

## 2. Thick vs thin — melyik mikor?

### 2.1 Mikor érdemes **thick**-et használni?

| Környezet / cél | Miért thick? |
|-----------------|--------------|
| **Szerver — rendszer- és adatkötet** (`/`, `/var`, `/home`, adatbázis-kötet) | Kiszámítható hely; a `df` és az LVM **egyezik**; kevesebb „pool tele” meglepetés. |
| **Egy nagy, stabil kötet** egy partición | Egyszerű bővítés: `lvextend` + fájlrendszer-növelés (Partctl: **LV méret növelése**). |
| **Hagyományos LVM snapshot** (mentés előtti pillanatfelvétel) | A klasszikus **COW snapshot** (`lvcreate -s`) **thick** eredeti LV-re épül — szervereken gyakori minta. |
| **Kevés kötet, hosszú élettartam** | Kevesebb metaadat, kevesebb thin-pool karbantartás. |
| **Partctl gyakorló / első LVM** | A **LVM-thick kotet letrehozasa** menüpont a legegyszerűbb teljes út. |

**Szerver példa (thick + snapshot gondolat):** adatbázis vagy fájl-szerver kötet (`/dev/vg0/data`). Mentés előtt **olvasható pillanatkép** készítése úgy, hogy az eredeti LV-ről **snapshot LV** jön létre a VG szabad területéből, a snapshot ideiglenesen mountolható vagy `dd` / `rsync` forrás lehet — az eredeti kötet közben írható maradhat (COW: a változások a snapshot területére kerülnek).

### 2.2 Mikor érdemes **thin**-t használni?

| Környezet / cél | Miért thin? |
|-----------------|-------------|
| **Virtuális gépek (KVM/QEMU, néhány hypervisor)** | Sok **független „lemez”** (VDI) egy fizikai partición; a hely **csak akkor** foglalódik, amikor a VM ír. |
| **Konténerek, fejlesztői környezetek** | Gyorsan sok kis kötet; gyakori klónozás / törlés. |
| **Tárhely kiszolgáló jelleg** (sok logikai unit egy medencében) | A pool egy helyen koncentrálja a fizikai helyet; a thin LV-k **virtuálisan nagyok** lehetnek. |
| **Thin snapshot** (pool szinten) | Modern LVM: **thin snapshot** a thin poolhoz kötődik — sok VM pillanatfelvétele **költséghatékonyabban**, mint ugyanannyi teljes thick másolat. |

**Mikor *ne* thin legyen az első választás:** kritikus **egyetlen** szerverkötet (pl. gyökér vagy adatbázis), ha a csapat nem akar **pool telítettség** monitorozást; ha régi eszközök / backup szoftver nem ismeri a thin LV-t; ha **nincs** terv a pool és a metaadat terület figyelésére.

### 2.3 Snapshot — szerver környezetben (összefoglaló)

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
| 1 | **LV kotet kezeles** | Meglévő LV: átnevezés, törlés, **méret növelés/csökkentés**, thin pool + data LV kézi létrehozása meglévő VG-n |
| 2 | **LVM-thick kotet letrehozasa** | Automatikus thick: PV + VG + LV egy varázslóban |
| 3 | **LVM-thin kotet letrehozasa** | Automatikus thin: PV + VG + thin pool + thin LV (data) |
| 4 | **PV kotet kezeles** | PV lista, létrehozás, törlés |
| 5 | **VG kotet kezeles** | VG lista, létrehozás, bővítés, aktiválás, törlés |
| 6 | **Vissza** | Vissza a partició menübe |

![](/img/lvm_muvelet_1.jpg)

---

## 5. Automatikus elnevezések (Partctl)

A program **egységes, felismerhető** LVM-neveket javasol / használ (a konkrét partició neve alapján, pl. `sdc3`):

| Elem | Példa név | Megjegyzés |
|------|-----------|------------|
| **VG** | `sdc3_partctl_vg` | Egy partició → egy VG (automatikus varázslók) |
| **Thick LV** | `thick` | VG-n belül: `/dev/sdc3_partctl_vg/thick` |
| **Thin pool** | `sdc3_partctl_thinpool` | Thin pool LV |
| **Thin data LV** | `sdc3_partctl_thinpool_data` | Használható kötet (virtuális partíció) |

- A **thin** út a fenti `*_partctl_vg` / `*_partctl_thinpool` konvenciót követi.
- A **thick** varázsló más mintát is használhat (`partctl_sdc3_thick_vg` + `thick`)

---

## 6. LVM-thick kötet automatikus létrehozása

<p align="center">
  <img src="/img/lvm_thick_1.png" alt="LVM működési elv — henger diagram (thick)" width="92%" />
</p>

**Cél:** egy partició (pl. `sdc3`) → PV → VG → egy thick LV → később formázás.

| Lépés | Menüút | Mit csinálsz |
|-------|--------|--------------|
| 1 | **Főmenü → `1`** | Lemez kiválasztása (pl. `sdc`). |
| 2 | *(Ha nincs partició)* | **Particio kezeles → Particio letrehozasa** — egy teljes partició a szabad sávra (Linux típus). |
| 3 | **Főmenü → `3` → LVM muveletek → `2`** | **LVM-thick kotet letrehozasa**. |
| 4 | Varázsló | Válaszd ki a **cél particiót** (pl. `sdc3`). |
| 5 | Megerősítés | Ellenőrizd: PV, VG név, LV név (`thick`) — **sárga** megerősítő panel. |
| 6 | **Folyamat** | Kék panel, **2 fázis:** (1) PV + VG, (2) LV létrehozás. Várj a végéig. |
| 7 | **Info** | **Egy** zöld összefoglaló: siker + teljes parancslánc. |
| 8 | **Főmenü → `3` → Particio formazas** | Válaszd az LV-t (pl. `/dev/sdc3_partctl_vg/thick` vagy mapper útvonal), **ext4** / **ntfs** stb. |
| 9 | *(Opcionális)* **Főmenü → `4` → Ideiglenes csatolás** | Csak teszthez; éles szerveren állandó `fstab` külön téma. |

**Háttérben (thick), tipikus parancsok:**

```text
pvcreate -ff -y /dev/sdc3
vgcreate sdc3_partctl_vg /dev/sdc3
lvcreate -y -W y -l 100%FREE -n thick sdc3_partctl_vg
```

---

## 7. LVM-thin kötet automatikus létrehozása (thin pool + data / „virtuális partíció”)

<p align="center">
  <img src="/img/lvm_thin_1.png" alt="LVM-thin működési elv" width="92%" />
</p>

**Cél:** egy partició → PV → VG → **thin pool** → **thin LV** (data) — a Partctl **egyetlen varázslóban**.

| Lépés | Menüút | Mit csinálsz |
|-------|--------|--------------|
| 1–2 | Ugyanaz, mint thick-nél | Lemez + **szabad partició** (`sdc3`). |
| 3 | **LVM muveletek → `3`** | **LVM-thin kotet letrehozasa**. |
| 4 | Varázsló | Cél partició; megerősítés: PV, VG, **thin pool**, **thin LV** (virtuális 100%). |
| 5 | **Folyamat** | **3 fázis:** (1) PV + VG, (2) thin pool, (3) thin LV (data). **Egy** folyamat, **egy** záró Info panel. |
| 6 | **Particio formazas** | Cél: a **data** thin LV (pl. `.../sdc3_partctl_thinpool_data`). |
| 7 | **Lemez attekintes** | Ellenőrizd: pool + data LV, méret, mapper útvonalak. |

**Háttérben (thin), tipikus parancsok:**

```text
pvcreate -ff -y /dev/sdc3
vgcreate sdc3_partctl_vg /dev/sdc3
lvcreate -y --type thin-pool -l 100%FREE -n sdc3_partctl_thinpool sdc3_partctl_vg
lvcreate -y -V <pool_meret>B -T /dev/sdc3_partctl_vg/sdc3_partctl_thinpool -n sdc3_partctl_thinpool_data
```

A **virtuális méret** (`-V`) a Partctl a pool aktuális méretéből számolja (100% modell).

---

## 8. Meglévő VG-n: thin pool + data kézi létrehozása

Ha már van **VG** (pl. több PV-vel bővítve), de thin pool kell:

| Lépés | Menüút |
|-------|--------|
| 1 | **LVM muveletek → `1`** **LV kotet kezeles** |
| 2 | Válaszd a **VG**-t |
| 3 | **LVM thin** típus → thin **pool** neve → megerősítés |
| 4 | **Folyamat** — pool létrejön |
| 5 | A program **automatikusan** létrehozza a **data** thin LV-t is (pool méret alapján) |
| 6 | **Egy** Info panel a végén |

---

## 9. PV / VG / LV kezelés (rövid táblázat)

| Feladat | Menü |
|---------|------|
| Új PV egy particióról | **PV kotet kezeles** → létrehozás |
| VG összeállítása PV-kből | **VG kotet kezeles** → létrehozás |
| LV törlése | **LV kotet kezeles** → törlés |
| LV méret növelése / csökkentése | **LV kotet kezeles** → extend / reduce varázslók |
| VG aktiválás / deaktiválás | **VG kotet kezeles** |

**Lemez attekintes:** LVM soroknál **Enter** — részletes `pvdisplay` / `vgdisplay` / `lvdisplay` jellegű összefoglaló.

---

## 10. Formázás, csatolás, takarítás

| Feladat | Menüút |
|---------|--------|
| Fájlrendszer az LV-n | **Particio kezeles → Particio formazas** |
| Fájlrendszer javítás | **Particio kezeles → Fajlrendszer javitas** |
| Ideiglenes csatolás | **Lemez kezeles → Ideiglenes csatolas** |
| Ideiglenes lecsatolás | **Lemez kezeles → Ideiglenes lecsatolas** |
| LVM + aláírások törlése | **Lemez kezeles → Lemez tisztitas (Wipe)** — opcionális LVM lépések (LV/VG/PV) |

A **Wipe** LVM opciói a **partíciós tábla megőrzése** mellett is futtathatók; teljes lemez törlésnél lásd a [`mbr-vs-gpt-partctl-guide.md`](mbr-vs-gpt-partctl-guide.md) Wipe szakaszát.

```markdown
https://github.com/drcyberg/partctl/blob/main/example/lvm-mukodesi-elv-partctl-guide.md
```

### Fő oldal (Partctl)

- [Partctl](https://drcyberg.github.io/partctl/web/partctl)

### Köszönöm ha támogatsz

- ***Buy me a coffee***: [LINK](https://buymeacoffee.com/drcyberg)
- ***Paypal***: [LINK](https://github.com/drcyberg/partctl/blob/main/img/qrcode.png)

*Utolsó frissítés jelleg: Partctl V1.0.0 viselkedés — a **Particio kezeles** lista ábécérendje miatt a konkrét **sorszámok** mindig a futó programban ellenőrizendők.*
