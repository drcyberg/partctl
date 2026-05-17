# Partíció létrehozás, törlés és célzott wipe — `partctl.sh` menüútmutató

> **Cél:** Szakszerűen bemutatni, hogyan hozol létre és távolítasz el partíciókat a **Partctl** (`bash partctl.sh`) felületén úgy, hogy **csökkenjen** a „látszólagos” vagy hibás partíciós állapot esélye (régi aláírások, eltérő kernel-tábla, félig törött szerkezet), és hogyan **kíméled** a flash/SSD kopását: **nem mindig** kell a teljes lemezt nullázni — sok esetben elég **egy partíció** wipe-ja vagy törlése.

**Figyelem:** partíciós tábla, partíciók, **wipe** és **formázás** **adatvesztést** okoz. Mindig **mentett**, **leválasztott** (unmount) adathordozóval dolgozz, **ne** a futó rendszerlemezen, és a **céllemezt** (pl. `/dev/sdc`) a **Lemez áttekintés** képernyőn ellenőrizd minden lépés előtt.

---

## 0. Gyors áttekintés — mit csinál a Partctl ebben a témában?

![](/img/particio-wipe-resized.png)

| Probléma / igény | Partctl megoldás (menü) | Mi történik a háttérben (röviden) |
|------------------|-------------------------|-----------------------------------|
| Régi FS/LVM „szellem” jelzés, de a tábla még „látszik” | **Lemez kezelés → Disk cleanup (Wipe)** | `wipefs -a` a cél eszközön; opcionálisan jelölők, tipuskód, nullázás |
| Teljesen üres lemez, új tábla | Wipe + **partíciós tábla törlés** → **Partíciós tábla létrehozása** | `sgdisk --zap-all` / `wipefs` (tábla varázsló), majd `parted mklabel` |
| Új partíció, jó igazítással | **Partíció létrehozása** | Alap **`2048s`** kezdő, `parted -a optimal`, utána `partprobe` |
| Egy zóna adatainak törlése, a többi megmarad | Wipe **konkrét partícióra** (nem teljes lemez) | `wipefs` csak `/dev/sdc2`-n; opcionális `dd` nullázás **csak** arra a sávra |
| Partíció eltávolítása, a többi marad | **Partíció törlése** | `parted rm`; kernel frissítés (`partprobe`) |
| Tábla megmarad, minden aláírás megy (sok partíció) | Teljes lemez wipe, **tábla törlés nélkül** | Partctl **partíciónként** futtat `wipefs`-t (nem a lemez csomóponton) |

---

## 1. Háttér — mi az a „látszólagos” partíciós tábla (Ghost FS)?

Gyakran **nem** arról van szó, hogy a GPT/MBR „kitalál” partíciókat, hanem az alábbiak keverednek:

1. **Régi fájlrendszer- vagy LVM-aláírások** — a `lsblk` / csatoló réteg még „lát” `ext4`, `ntfs`, `LVM2_member` jelzést, holott a partíciós bejegyzés már törölve vagy más.
2. **GPT másodlagos fejléc / maradék GUID** — korábbi táblából maradt metaadat.
3. **Kernel cache** — a tábla már módosult, de a régi `/dev/sdc2` node még látszik, amíg nincs `partprobe`, azaz az eszköz újra becsatlakoztatva.
4. **Részleges művelet** — pl. csak `dd` nullázás **tábla törlés nélkül**, vagy fordítva: tábla törölve, de az aláírások megmaradtak.

A Partctl ezt több ponton kezeli:

- 1.) **Wipe** előtt leválasztás (`ensure_target_detached`).
- 2.) **Partíció törlés** előtt: `refresh_disk_block_layer` (friss kernel-tábla).
- 3.) **Új particiós tábla** előtt: `sgdisk --zap-all` + `wipefs -a` a lemezen (ha elérhető).
- 4.) **Teljes lemez wipe**, ha **nincs** bejelölve a „partíciós tábla törlése”: a program **nem** a `/dev/sdc` csomóponton futtatja a `wipefs`-t (az törölné a táblát), hanem **minden meglévő partíció eszközén** külön (`wipefs -a /dev/sdc1`, `sdc2`, …).

---

## 2. Wipe típusok — mit válassz?

**Útvonal minden wipe-hoz:** **főmenü → `4` Lemez kezelés → `10` Disk cleanup (Wipe)**.

### 2.1 Cél kiválasztása

A varázsló első lépésében választhatsz:

| Cél | Mikor |
|-----|--------|
| **teljes lemez** (`/dev/sdc`) | Nulláról, vagy minden partíció egyszerre |
| **egy partíció** (`/dev/sdc2`) | Csak egy zóna tisztítése — **kevesebb írás**, a többi érintetlen |

![](/img/wipe_2.jpg)

### 2.2 Jelölőnégyzetek (Space) — összefoglaló

| Opció | Teljes lemez | Egy partíció | Hatás |
|-------|:------------:|:------------:|-------|
| **`wipefs -a`** (fix, mindig) | ✓ | ✓ | Fájlrendszer / LVM **aláírások** törlése |
| **Partíciós tábla törlése (GPT/MBR)** | ✓ | — | A lemez **struktúrája** is megy (üres lemez) |
| **GPT/MBR jelölők** | ✓ | ✓ | `parted set … off` |
| **Partíció típuskód** | ✓ | ✓ | GUID / hex visszaállítás |
| **Partíció nullázás (`dd`)** | — | ✓ | **Csak** a kiválasztott partíció sávját írja tele nullával |
| **Teljes lemez nullázás (`dd`)** | ✓ (opc.) | — | **Nagyon lassú**, maximális írási terhelés |
| **LVM (LV/VG/PV)** | ✓ | ✓ | Kapcsolódó LVM bejegyzések törlése wipe előtt |

![](/img/wipe_3.jpg)

**Kopás / élettartam (flash, SSD):**

| Művelet | Írási terhelés | Tipikus használat |
|---------|----------------|-------------------|
| `wipefs` egy partíción | **Alacsony** | Formázás előtti tisztítás, „szellem” FS jel eltüntetése |
| Partíció törlése (`parted rm`) | **Minimális** | Struktúra módosítás, adat a sávban **maradhat** (nem biztonságos törlés!) |
| Partíció `dd` nullázás | **Közepes** (csak a partíció mérete) | Adat megsemmisítése **egy zónában** |
| Teljes lemez `dd` nullázás | **Maximális** | Biztonságos megsemmisítés vagy teljes újrakezdés |

> **Fontos:** a `wipefs` **nem** biztonságos adattörlés — csak az aláírásokat távolítja el. Ha valódi adatmegsemmisítés kell, használd a **nullázást**, vagy titkosítást (LUKS) kulcs eldobásával.

---

## 3. Indítás és közös vezérlés

```bash
bash partctl.sh
```

![](/img/terminal_1.jpg)

**Rögzített főmenü-sorszámok:**

| # | Angol | Magyar felületen |
|---|--------|---------------------|
| **1** | Select Disk | Lemez kivalasztasa |
| **2** | Disk Overview | Lemez attekintes |
| **3** | Partition management | Particio kezeles |
| **4** | Disk management | Lemez kezeles |
| **5** | Setup | Beallitasok |
| **6** | About | Rolunk |
| **7** | Exit | Kilepes |

**Navigáció:** `Fel` / `Le` (vagy `k` / `j`), **Enter**; vagy a sor elején látható **`N.`** szám + **Enter**. **Vissza:** **Backspace** / **`q`**.

A **Particio kezeles** lista **ábécérendben** van — a konkrét sorszámot mindig a **futó programban** ellenőrizd; az alábbi útmutató a **menüpont címkéjét** használja.

---

## 4. Példa A — **32 GB pendrive**, egy FAT32 kötet (klasszikus)

**Cél:** Egyetlen, jól felismerhető FAT32 partíció (pl. általános adathordozó, kisebb fájlokkal). Példa lemez: **`/dev/sdc`**, ~29 GiB.

| Lépés | Menüút | Mit csinálsz |
|-------|--------|--------------|
| 1 | **Főmenü → `1`** Lemez kivalasztasa | Kiválasztod **`sdc`**-t — a **Lemez attekintes**-ben ellenőrizd, hogy **nem** a rendszerlemez. |
| 2 | **Főmenü → `4` → `10`** Disk cleanup (Wipe) | Cél: **teljes lemez**; **[✓] Partíciós tábla törlése** — „tiszta lap”. |
| 3 | **Főmenü → `3`** → **Particios tabla letrehozasa** | **`2` — MBR (msdos)** (pendrive, max. kompatibilitás). Megerősítés. |
| 4 | **Főmenü → `3`** → **Particio letrehozasa** | **Kezdet:** **Enter** → alap **`2048s`**. **Vég:** **`100%`**. |
| 5 | **Főmenü → `3`** → **Particio formazas** | Cél: `sdc1`; típus: **`vfat`** (FAT32). |
| 6 | **Főmenü → `2`** Lemez attekintes | Tábla `msdos`, `sdc1`, `FSTYPE` = `vfat`. |

**Eredmény:**

| Eszköz | Tábla | Partíció | FS | Megjegyzés |
|--------|-------|----------|-----|------------|
| `sdc` | `msdos` | — | — | Pendrive |
| `sdc1` | — | primary | `vfat` | Egy kötet, max. **~4 GiB/fájl** (FAT32 limit) |

| Lemez áttekintés | Partíció részletei |
| --- | --- |
| ![usb_partition_1](/img/usb_partition_1.jpg "Lemez áttekintés #1") | ![usb_partition_2](/img/usb_partition_2.jpg "Partíció részletei #1") |

---

## 5. Példa B — **két vagy több FAT32** partíció beállítása

**Cél:** Ugyanazon a pendrive-on **7 db külön kötet** létrehozni, hogy **nagy fájl** terjedelmek is elférjen (max. 7 x 4 GiB partíciók), anélkül hogy exFAT/NTFS kellene. Az ilyen típusú USB pendrivokat Cisco és más eszközök esetében érdemes használni, például: különböző **firmware feltöltésekhez**.
Példa: **`/dev/sdc`**, ~32 GiB.

> Mehjegyzés: Tévhit, hogy a FAT32 **4 GiB korlátja fájlméretre** vonatkozik, **nem** a partíció méretére. Egy 32 GB-os FAT32 partíción is csak ~4 GiB-nál kisebb egyetlen fájl mehet.

| Lépés | Menüút | Mit csinálsz |
|-------|--------|--------------|
| 1 | Lemez kiválasztása** | `/dev/sdc` |
| 2 | Teljes lemez wipe** és partíciós tábla törlése | `teljes lemez (/dev/sdc)` |
| 3 | Particios tabla letrehozasa | MBR (msdos) |
| 4a | Particio letrehozasa | Kezdet: `2048s`. Vég: `+4GiB` |
| 4b | Particio letrehozasa | Kezdet: `8390656s`. Vég: `+4GiB` |
| 4c | Particio letrehozasa | Kezdet: `16779264s`. Vég: `+4GiB` |
| 4d | Particio letrehozasa | Kezdet: `25167872s`. Vég: `+4GiB` |
| 4e | Particio letrehozasa | Kezdet: `33556480s`. Vég: `+4GiB` |
| 4f | Particio letrehozasa | Kezdet: `41945088s`. Vég: `+4GiB` |
| 4g | Particio letrehozasa | Kezdet: `50333696s`. Vég: `+4GiB` |
| 5a | Particio formazas | `sdc1` → `vfat` |
| 5b | Particio formazas | `sdc2` → `vfat` |
| 5c | Particio formazas | `sdc3` → `vfat` |
| 5d | Particio formazas | `sdc4` → **üres** |
| 5e | Particio formazas | `sdc5` → `vfat` |
| 5f | Particio formazas | `sdc6` → `vfat` |
| 5g | Particio formazas | `sdc7` → `vfat` |
| 5h | Particio formazas | `sdc8` → `vfat` |
| 6 | Lemez attekintes | 8 sor: `sdc1` - `sdc8`, ami mindegyik `vfat` fájlrendszer, kivéve az `sdc4` partíció, , ami egy **Extended (LBA)** particio tipuskod jelöléssel van ellátva ismerve az MBR (dos) partíció limitációját.

**Eredmény:**

| Eszköz | Méret (példa) | FS | Használat |
|--------|---------------|-----|-----------|
| `/dev/sdc1` | 4.0 GiB | `vfat` | pl. `cisco-fw-1.bin` (≤4 GiB) |
| `/dev/sdc2` | 4.0 GiB | `vfat` | pl. `cisco-fw-2.bin` (≤4 GiB) |
| `/dev/sdc3` | 4.0 GiB | `vfat` | pl. `cisco-fw-3.bin` (≤4 GiB) |
| `/dev/sdc4` | 1.0 KiB | `üres` | Extended (LBA) |
| `/dev/sdc5` | 4.0 GiB | `vfat` | pl. `cisco-fw-4.bin` (≤4 GiB) |
| `/dev/sdc6` | 4.0 GiB | `vfat` | pl. `cisco-fw-5.bin` (≤4 GiB) |
| `/dev/sdc7` | 4.0 GiB | `vfat` | pl. `cisco-fw-6.bin` (≤4 GiB) |
| `/dev/sdc8` | 4.0 GiB | `vfat` | pl. `cisco-fw-7.bin` (≤4 GiB) 

| Lemez áttekintés | Fájlrendszer címke |
| --- | --- |
| ![usb_multiple_partition_1](/img/usb_multiple_partition_1.jpg "Lemez áttekintés #1") | ![cisco_fat32_2](/img/usb_multiple_partition_2.jpg "Fájlrendszer címke #1") |

---

## 6. Példa C — **Lemez tisztítás (wipe)** egy kijelölt partíción

**Kiindulás:** `/dev/sdc` már két partícióval (Példa B). Csak **`sdc2`** adatait szeretnéd eltávolítani, **`sdc1`** érintetlen.

| Lépés | Menüút | Mit csinálsz |
|-------|--------|--------------|
| 1 | **Főmenü → `1`** | Lemez: **`sdc`**. |
| 2 | **Főmenü → `4` → `10`** Wipe | Cél: **`/dev/sdc2`** (ne a teljes lemez!). |
| 3 | Jelölőnégyzetek | Alap: **`wipefs`** (mindig). Opcionális: **[✓] Partíció nullázás** — ha adatot is felül akarsz írni. **[ ]** teljes lemez nullázás — **ne** kapcsold be. |
| 4 | Enter + megerősítés | A tábla és **`sdc1`** megmarad. |
| 5 | *(Opcionális)* **Particio formazas** | `sdc2` → **`vfat`** újra — üres, használható kötet. |

**Miért jobb, mint a teljes lemez nullázása?** A `dd` csak a **`sdc2`** méretű sávot írja (4 GiB), nem az egész 32 GiB-ot — **kevesebb flash kopás**.

---

## 7. Példa D — **egy partíció törlése**

**Cél:** `sdc2` **eltávolítása** a táblából; `sdc1` marad; a felszabadult hely később újra partícionálható.

| Lépés | Menüút | Mit csinálsz |
|-------|--------|--------------|
| 1 | **Főmenü → `1`** | Lemez: **`sdc`**. |
| 2 | **Főmenü → `3`** → **Particio torlese** | Válaszd **`sdc2`**-t. Megerősítés. |
| 3 | **Lemez attekintes** | `sdc2` sor **eltűnik**; szabad sáv jelenik meg. |

> **Megjegyzés:** a törlés **nem** nullázza biztonságosan az adatot — a régi bitek a lemezen maradhatnak, amíg felül nem írod őket. Adatmegsemmisítéshez: **Wipe** + nullázás a partíción **törlés előtt**, vagy törlés után új partíció + nullázás.

Ha a program **kernel figyelmeztetést** ad: húzd ki/csatlakoztasd újra az eszközt, vagy futtass `partprobe`-ot — a Partctl ezt jelzi a sikeres törlés után is.

---

## 8. Példa E — **minden partíció** megmarad, csak az aláírást távolítjuk el

**Cél:** GPT/MBR szerkezet **megmarad** (pl. előre definiált `sdc1`…`sdc4` Windows-elrendezés), de minden köteten eltűnjenek a régi FS/LVM jelzések — **teljes lemez `dd` nélkül**.

| Lépés | Menüút | Mit csinálsz |
|-------|--------|--------------|
| 1 | **Főmenü → `4` → `10`** Wipe | Cél: **teljes lemez** (`/dev/sdc`). |
| 2 | Jelölőnégyzetek | **[ ] Partíciós tábla törlése** — **ki** marad. **[✓]** jelölők / tipuskód, ha kell. |
| 3 | Futtatás | A Partctl **partíciónként** `wipefs`-ol (`sdc1`, `sdc2`, …). |

Ha nincs egyetlen partíció sem a lemezen, a program **nem** futtat teljes-lemez `wipefs`-t (védelem a véletlen táblatörlés ellen), és figyelmeztet: *„A GPT/MBR megőrzés aktív, de nem található törölhető partíció cél…”* — ilyenkor előbb hozz létre partíciókat, vagy kapcsold be a **tábla törlést**.

**Napló:** minden művelet visszakereshető a `log/partctl-*.log` fájlban és a program **Napló** paneljén.

*Partctl V1.0.0 — partíció és wipe menüútmutató. A menüsorszámok a futó program képernyőjén mindig ellenőrizendők (különösen a **Partíció kezelés** ábécésrendű listájában).*
