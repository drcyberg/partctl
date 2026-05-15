# LVM működési elv — összefoglaló és `partctl.sh` menüútmutató

> **Cél:** Rövid, jól értelmezhető háttér a **Linux LVM** (Logical Volume Manager) rétegeiről, a **thick** és **thin** különbségéről, és arról, hogyan hozhatók létre / kezelhetők a kötetek a **Partctl** menüiből (`bash partctl.sh`).  
> **Figyelem:** LVM és particiós műveletek **adatvesztést** okozhatnak. Csak **mentett**, **nem** futó rendszerlemezen, **leválasztott** (unmount) célokon dolgozz; éles környezetben mindig **biztonsági mentés**.

---

## Ábra — LVM működési elv (sötét téma)

<p align="center">
  <img src="/img/lvm_mukodesi_elv_1.png" alt="LVM működési elv — parancsok és rétegek" width="92%" />
</p>

*Felül:* **Fizikai kötet → Kötetcsoport → Logikai kötet → Fájlrendszer**. Bal oldalon a három alapparancs (`pvcreate`, `vgcreate`, `lvcreate`), jobb oldalon a tárolási rétegek.

<p align="center">
  <img src="/img/lvm_thick_1.png" alt="LVM működési elv — henger diagram (thick)" width="92%" />
</p>

*Klasszikus (thick) LVM:* több **PV** egy **VG**-be, onnan **LV**-k, majd **fájlrendszer**.

<p align="center">
  <img src="/img/lvm_thin_1.png" alt="LVM-thin működési elv" width="92%" />
</p>

*Thin provisioning:* a **VG** felett **thin pool**, abból **thin LV** (virtuális méret), majd fájlrendszer.

---

## 1. Mi az LVM? — négy réteg, egy mondatban

Az LVM a **Linux partíciók fölött** egy **logikai** réteget épít:

| Réteg | Rövidítés | Mit jelent? | Tipikus parancs |
|-------|-----------|-------------|-----------------|
| **Fizikai kötet** | **PV** | Egy partició (vagy lemez), amit az LVM „befogad” | `pvcreate` |
| **Kötetcsoport** | **VG** | A PV-k összevonva: egy nagy „tárolómedence” | `vgcreate` |
| **Logikai kötet** | **LV** | A VG-ből kivágott, használható blokk-eszköz (`/dev/vg/lv`) | `lvcreate` |
| **Fájlrendszer** | — | ext4, NTFS, xfs stb. az LV **fölött** | `mkfs.*`, Partctl **Particio formazas** |

**Fontos:** Az LVM **nem** helyettesíti a particiós táblát (GPT/MBR). Kell egy **Linux partíció** (vagy egész lemez PV-ként), **utána** jön az LVM.

---

## 2. Thick vs thin — melyik mikor?

### 2.1 LVM-thick (klasszikus)

- A logikai kötet mérete **előre lefoglalt** a VG-ből (`lvcreate -l 100%FREE` jelleg).
- Amit a `df` / `lsblk` mutat, az **közel ugyanannyi** hely, amennyit az LV „lefoglalt”.
- **Egyszerűbb** gondolkodás, kevesebb fogalom — **kezdőknek és általános adat-/rendszerköteteknek** ajánlott.

### 2.2 LVM-thin (thin provisioning)

- Először egy **thin pool** jön létre a VG-ben (`lvcreate --type thin-pool`).
- A **thin LV** (adat-kötet) **virtuális méretű** lehet; a **tényleges** hely a poolból foglalódik **igény szerint**.
- Több thin LV **megoszthatja** ugyanazt a poolt (over-provisioning lehetséges — vigyázz a túlfoglalással).
- A Partctl **egy partició = egy thin tároló** modellt követ: automatikusan létrehozza a **poolt** és a **data** thin LV-t is.

| | **Thick** | **Thin** |
|---|-----------|----------|
| Lépések (Partctl) | PV → VG → **LV** | PV → VG → **thin pool** → **thin LV** |
| Méret | VG szabad helyének ~100%-a | Pool = VG szabad; thin LV virtuális méret (pl. 100% a poolé) |
| Használat | Általános kötet | VDI, konténerek, sok kis „virtuális lemez” egy partición |

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

---

## 5. Automatikus elnevezések (Partctl)

A program **egységes, felismerhető** LVM-neveket javasol / használ (a konkrét partició neve alapján, pl. `sdc3`):

| Elem | Példa név | Megjegyzés |
|------|-----------|------------|
| **VG** | `sdc3_partctl_vg` | Egy partició → egy VG (automatikus varázslók) |
| **Thick LV** | `thick` | VG-n belül: `/dev/sdc3_partctl_vg/thick` |
| **Thin pool** | `sdc3_partctl_thinpool` | Thin pool LV |
| **Thin data LV** | `sdc3_partctl_thinpool_data` | Használható kötet (virtuális partíció) |

*A thick varázsló más mintát is használhat (`partctl_sdc3_thick_vg` + `thick`); a **thin** út a fenti `*_partctl_vg` / `*_partctl_thinpool` konvenciót követi.*

---

## 6. LVM-thick kötet létrehozása (lépésről lépésre)

**Cél:** egy partició (pl. `sdc3`) → PV → VG → egy thick LV → később formázás.

| Lépés | Menüút | Mit csinálsz |
|-------|--------|--------------|
| 1 | **Főmenü → `1`** | Lemez kiválasztása (pl. `sdc`). |
| 2 | *(Ha nincs partició)* | **Particio kezeles → Particio letrehozasa** — egy teljes partició a szabad sávra (Linux típus). |
| 3 | **Főmenü → `3` → LVM muveletek → `2`** | **LVM-thick kotet letrehozasa**. |
| 4 | Varázsló | Válaszd ki a **cél particiót** (pl. `sdc3`). |
| 5 | Megerősítés | Ellenőrizd: PV, VG név, LV név (`thick`) — **sárga** megerősítő panel. |
| 6 | **Folyamat (Proc)** | Kék panel, **2 fázis:** (1) PV + VG, (2) LV létrehozás. Várj a végéig. |
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

## 7. LVM-thin kötet létrehozása (thin pool + data / „virtuális partíció”)

**Cél:** egy partició → PV → VG → **thin pool** → **thin LV** (data) — a Partctl **egyetlen varázslóban**, **három Proc-fázissal**.

| Lépés | Menüút | Mit csinálsz |
|-------|--------|--------------|
| 1–2 | Ugyanaz, mint thick-nél | Lemez + **szabad partició** (`sdc3`). |
| 3 | **LVM muveletek → `3`** | **LVM-thin kotet letrehozasa**. |
| 4 | Varázsló | Cél partició; megerősítés: PV, VG, **thin pool**, **thin LV** (virtuális 100%). |
| 5 | **Folyamat (Proc)** | **3 fázis:** (1) PV + VG, (2) thin pool, (3) thin LV (data). **Egy** folyamat, **egy** záró Info panel. |
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
| 4 | **Proc** — pool létrejön |
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

---

## 11. Tippek és gyakori hibák

1. **„Device busy”** — csatolt kötet vagy aktív mapper; használd az **Ideiglenes lecsatolas** menüt, vagy a Wipe **LVM deaktiválás** ajánlatát.
2. **„Already a PV”** — a partició már LVM tag; másik particiót válassz, vagy töröld a régi VG-t / Wipe-olj.
3. **Thin túlfoglalás** — több thin LV virtuális mérete **összesen** meghaladhatja a pool fizikai méretét; csak akkor írj, ha érted a kockázatot.
4. **Mapper útvonal** — a rendszer `/dev/mapper/vg-lv` formát is használhat; a Partctl áttekintésben mindkettő megjelenhet.
5. **Napló** — a **Proc** panel mellett a `log/partctl-*.log` fájlban a parancsok és kimenetek visszakereshetők.
6. **Egy Info a végén** — a thick és thin automatikus varázslók **szándékosan** csak **egy** zöld összefoglalót mutatnak siker esetén; a részletes lépések a **Proc** panelen és a naplóban látszanak.

---

## 12. Összefoglaló — melyik menü mire való?

```text
Új thick tároló (egy partició):     Particio kezeles → LVM muveletek → LVM-thick kotet letrehozasa
Új thin tároló (pool + data):       Particio kezeles → LVM muveletek → LVM-thin kotet letrehozasa
Formázás:                          Particio kezeles → Particio formazas
Részletek / ellenőrzés:            Lemez attekintes → Enter (LVM sor)
Méret állítás:                     LVM muveletek → LV kotet kezeles
Teljes takarítás:                  Lemez kezeles → Lemez tisztitas (Wipe)
```

---

## Kapcsolódó anyagok

- [MBR és GPT partíciós tábla](mbr-vs-gpt-partctl-guide.md) — partició és tábla **LVM előtt**
- [Partíció igazítás](particio-igazitas-partctl-guide.md) — particiók **LVM előtti** geometriája
- [Partctl felhasználói kézikönyv](README.md) — telepítés, napló, általános menük

*Utolsó frissítés jelleg: Partctl V1.0.0 — LVM thick/thin automatikus varázslók, Proc panel (többfázisú), egységes záró Info.*
