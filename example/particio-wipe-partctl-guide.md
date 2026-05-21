# Partíció létrehozás, törlés és célzott wipe — `partctl.sh` menüútmutató

> **Miről szól ez az útmutató?**  
> Megmutatja, hogyan hozol létre és távolítasz el partíciókat a **Partctl** (`bash partctl.sh`) programmal úgy, hogy elkerüld a „látszólag még ott lévő”, hibás lemezállapotot (régi fájlrendszer-jelzések, elavult kernel-információ). Ugyanakkor **kíméled** a flash- vagy SSD-meghajtót: nem mindig kell a teljes lemezt nullázni — gyakran elég **egyetlen partíció** törlése vagy wipe-ja.

**Figyelem:** a partíciós tábla módosítása, a partíciók törlése, a **wipe** és a **formázás** **adatvesztéssel** jár. Csak **mentett**, **leválasztott** (unmount) adathordozón dolgozz, soha a futó rendszerlemezen. A **céllemezt** (pl. `/dev/sdc`) minden lépés előtt ellenőrizd a **Lemez attekintes** képernyőn.

---

## 0. Gyors áttekintés

![](/img/particio-wipe-resized.png)

| Mit szeretnél elérni? | Hol a Partctlben? | Mi történik a háttérben? |
|------------------------|-------------------|---------------------------|
| „Szellem” fájlrendszer-jelzés, de a tábla még látszik | **Lemez kezeles → Disk cleanup (Wipe)** | `wipefs -a`; opcionálisan jelölők, típuskód, nullázás |
| Teljesen üres lemez, új indulás | Wipe + **partíciós tábla törlés** → **Particios tabla letrehozasa** | `sgdisk --zap-all` / `wipefs`, majd `parted mklabel` |
| Új partíció, jó igazítással | **Particio letrehozasa** | Kezdő **`2048s`**, `parted -a optimal`, `partprobe` |
| Csak egy zóna adatainak törlése | Wipe **egy partícióra** (nem a teljes lemezre) | `wipefs` pl. csak `/dev/sdc2`-n; opcionális `dd` nullázás |
| Partíció eltávolítása, a többi marad | **Particio torlese** | `parted rm`, majd `partprobe` |
| Tábla megmarad, minden aláírás megy | Teljes lemez wipe, **tábla törlés nélkül** | `wipefs` **partíciónként** (nem a lemez csomóponton) |
| Kötet felismerhető neve (címke) | **Lemez kezeles → Fajlrendszer cimke** (`4` → `5`) | pl. `fatlabel` — **opcionális**, lásd §4.5 |
| **Cisco IOS pendrive** (9200 / 9300 / …) | Wipe → **MBR** → **1×** partíció → **`vfat`** | **Egy** `sda1` → switch-en `usbflash0:` — lásd §4.6, §5, [Cisco útmutató](cisco-usb-flash-partctl-guide.md) |

---

## 1. Mi az a „látszólagos” artíciós tábla? (Ghost FS)

Akkor alakul ki, miután az MBR-, vagy a GPT tábla letörlése után nem történt teljes-, vagy célzott lemez tisztítás (wipe). Ettől függetlenül még az aláírások meg megmaradnak. Így ha pont ugyan attól a kezdő szektortól partícionálunk „látszólagos”, azaz fals fájlrendszert kapunk (VESZÉLYES). Illetve még pár helyzetben is előfordulhat:

1. **Régi fájlrendszer- vagy LVM-aláírások** — a `lsblk` még `ext4`, `ntfs` vagy `LVM2_member` jelzést mutat, holott a partíciós bejegyzés már törölve lett vagy más.
2. **GPT másodlagos fejléc, maradék GUID** — korábbi táblából maradt metaadat.
3. **Kernel-gyorsítótár** — a tábla már frissült, de a régi `/dev/sdc2` node még látszik, amíg nincs `partprobe` vagy az eszköz nincs újradugva.
4. **Félbemaradt művelet** — pl. csak `dd` nullázás **tábla törlés nélkül**, vagy fordítva: a tábla elment, de az aláírások megmaradtak.

**Hogyan segít a Partctl?**

- **Wipe** előtt: a kötet leválasztása.
- **Partíció törlés** előtt: friss kernel-tábla (`refresh_disk_block_layer`).
- **Új particiós tábla** előtt: `sgdisk --zap-all` és `wipefs -a` (ha elérhető).
- **Teljes lemez wipe** táblatörlés **nélkül**: a program **nem** a `/dev/sdc` eszközön futtatja a `wipefs`-t (az törölné a táblát), hanem **minden partíción külön** (`/dev/sdc1`, `sdc2`, …).

---

## 2. Lemez tisztítás (Wipe) — mit válassz a partíció listából?

**Menüút:** **főmenü → `4` Lemez kezeles → `10` Disk cleanup (Wipe)**.

### 2.1 Cél kiválasztása

| Cél | Mikor érdemes? |
|-----|----------------|
| **Teljes lemez** (`/dev/sdc`) | Nulláról indulsz, vagy minden partíciót egyszerre tisztítasz |
| **Egy partíció** (`/dev/sdc2`) | Csak egy zónát törölsz — **kevesebb írás**, a többi érintetlen marad |

![](/img/wipe_2.jpg)

### 2.2 Lemez tisztítás (wipe) - mit válassz a végrehajtási listából?

| Opció | Teljes lemez | Egy partíció | Mit csinál? |
|-------|:------------:|:------------:|-------------|
| **`wipefs -a`** (mindig) | ✓ | ✓ | Fájlrendszer- és LVM-**aláírások** törlése |
| **Partíciós tábla törlése** | ✓ | — | A lemez **struktúrája** is törlődik (üres lemez) |
| **GPT/MBR jelölők** | ✓ | ✓ | `parted set … off` |
| **Partíció típuskód** | ✓ | ✓ | Hex / GUID visszaállítás |
| **Partíció nullázás (`dd`)** | — | ✓ | Csak a kiválasztott sáv nullázása |
| **Teljes lemez nullázás** | ✓ (opc.) | — | Nagyon lassú, maximális kopás |
| **LVM (LV / VG / PV)** | ✓ | ✓ | Kapcsolódó LVM-bejegyzések törlése wipe előtt |

> Megjegyzés: **Kopás flash / SSD esetén** megnőhet

![](/img/wipe_3.jpg)

| Művelet | Írási terhelés | Jellemző használat |
|---------|----------------|---------------------|
| `wipefs` egy partíción | Alacsony | Formázás előtti tisztítás. Csak azokat a „magic string” / aláírás bájtokat törli, amelyeket a libblkid felismer. |
| Partíció törlése | Minimális | A táblában eltűnik a bejegyzés; az adat a sávban **maradhat** |
| Partíció `dd` nullázás | Közepes | Adat megsemmisítése **egy zónában** |
| Teljes lemez `dd` nullázás | Maximális | Biztonságos megsemmisítés vagy teljes újrakezdés |

> **Fontos:** a `wipefs` **nem** biztonságos adattörlés — csak az aláírásokat távolítja el. Valódi megsemmisítéshez használj **nullázást**, vagy titkosítást (LUKS) kulcs eldobásával.

---

## 3. Indítás és navigáció

```bash
bash partctl.sh
```

![](/img/terminal_1.jpg)

**Főmenü (rögzített sorszámok):**

| # | Angol | Magyar |
|---|--------|--------|
| **1** | Select Disk | Lemez kivalasztasa |
| **2** | Disk Overview | Lemez attekintes |
| **3** | Partition management | Particio kezeles |
| **4** | Disk management | Lemez kezeles |
| **5** | Setup | Beallitasok |
| **6** | About | Rolunk |
| **7** | Exit | Kilepes |

**Vezérlés:** `Fel` / `Le` (vagy `k` / `j`), majd **Enter**; vagy a sor eleji **`N.`** szám begépelése. **Vissza:** **Backspace** vagy **`q`**.

A **Particio kezeles** menü **ábécérendben** van — a konkrét sorszámot mindig a **futó programban** nézd meg. Az alábbi lépések a **menüpont nevét** használják, nem a sorszámot.

---

## 4. Partíció típuskód — MBR és GPT (FAT32 / FAT16)

A **`vfat`** (FAT32) és **`fat16`** formázás után a Partctl **automatikusan** beállítja a partíció típuskódját — a lemez táblájától függően:

| Tábla | FAT32 (`vfat`) formázás után | FAT16 (`fat16`) formázás után |
|-------|------------------------------|-------------------------------|
| **MBR** | **`0C`** — *W95 FAT32 (LBA)* | **`0E`** — *W95 FAT16 (LBA)* |
| **GPT** | **Microsoft basic data** (`EBD0A0A2-…`) + `msftdata` jelölő | ugyanígy: **Microsoft basic data** + `msftdata` |

> Megjegyzés: **NTFS** ugyanezen a logikán megy: MBR-en **`07`**, GPT-n **Microsoft basic data** — mint a FAT32 GPT ág.

**Kézi beállítás** (ha ellenőrzésnél rossz érték látszik):

- **MBR:** **főmenü → `3`** → **MBR particio tipuskod**
- **GPT:** **főmenü → `3`** → **GPT particio tipuskod**

> Megjegyzés: A típuskód **nem** helyettesíti a fájlrendszert: a formázás hozza létre a FAT-ot; a kód azt jelzi, **milyen szerepet** vár el tőle a Windows, Linux OS.

### 4.1 Gyors választási tábla (MBR)

| Partíció szerepe | Formázás | MBR kód a listában | Megjegyzés |
|------------------|----------|--------------------|------------|
| Adat / pendrive (általános, PC) | **`vfat`** | **`0C`** — *W95 FAT32 (LBA)* | **Ajánlott** modern USB, és HDD háttértárolóknál |
| **Cisco IOS USB flash** (9200 / 9300 / …) | **`vfat`** | **`0C`** — *W95 FAT32 (LBA)* | **Egy** partíció, **MBR** — lásd §4.6 |
| Adat (régi környezet) | **`vfat`** | **`0B`** — *W95 FAT32* | LBA nélkül; ma ritkán |
| NTFS / exFAT adat | **`ntfs`** / exFAT | **`07`** — *Microsoft basic data* | NTFS után a Partctl **automatikusan** `07`-re állít |
| Extended konténer | *ne formázd* | **`0F`** — *Extended (LBA)* | Pl. **`sdc4`** a §6 példában |
| Logikai FAT32 | **`vfat`** | **`0C`** — *W95 FAT32 (LBA)* | **Ajánlott** modern USB, és HDD háttértárolóknál |
| Linux adat | **`ext4`** stb. | **`83`** | - |
| EFI (MBR-en ritka) | **`vfat`** | **`EF`** | - |

> **Ne keverd össze:** a **`07`** típuskód az NTFS/exFAT adatpartícióhoz való. Sima **FAT32 pendrive**-ra **`0C`** típuskód kell, nem pedig a `07`. A Cisco IOS / IOS XE **FAT32** pendrive-ot **`0C`** + **egyetlen** partícióval várja — a `07` kód **nem** helyettesíti a helyes FAT32 előkészítést (részletek: [Cisco útmutató](cisco-usb-flash-partctl-guide.md)).

![](/img/mbr_particio_tipuskod_1.jpg)

### 4.2 Mi fut a háttérben? (*Particio formazas* után)

A formázás varázsló egy második lépésben állítja a típuskódot (`format_operation_step_typecode`):

| Fájlrendszer | MBR | GPT |
|--------------|-----|-----|
| **`vfat`** | `sfdisk` / `fdisk` → **`0C`** | `sgdisk --typecode=…:EBD0A0A2-…` + `parted … msftdata on` |
| **`fat16`** | → **`0E`** | `sgdisk --typecode=…:EBD0A0A2-…` + `parted … msftdata on` |
| **`ntfs`** | → **`07`** | `sgdisk --typecode=…:EBD0A0A2-…` + `parted … msftdata on` |

**Ellenőrzés:** **Lemez attekintes** → partíció → **Enter**. MBR-n a **PARTTYPE** sorban pl. **`0C`** (FAT32) vagy **`0F`** (extended). GPT-n **Microsoft basic data** / **`0700`** jellegű kód. Ha **`83`** (Linux) maradt, futtasd újra a megfelelő típuskód varázslót (§4 első táblázata).

### 4.3 GPT — mikor kell még kézzel beállítani?

A **FAT32 / FAT16 formázás GPT-n is automatikusan** *Microsoft basic data* GUID-ot kap — **nem** kell utána külön kiválasztani a listából, ha a formázás sikeres volt.

Kézi **GPT particio tipuskod** akkor kell, ha:

- a formázás típuskód-lépése hibára futott (hiányzó `sgdisk` / `parted`),
- régi, rossz GUID maradt, és csak a típust javítod formázás nélkül.

| Szerep | GPT típus (lista / automatikus FAT32 után) |
|--------|------------------------------------------|
| Általános adat (FAT32, FAT16, NTFS, exFAT) | **Microsoft basic data** — `EBD0A0A2-B9E5-4433-87C0-68B6B72699C7` |
| EFI rendszer (boot, kis ESP) | **EFI System** — `C12A7328-F81F-11D2-BA4B-00A0C93EC93B` — ezt **nem** állítja a sima `vfat` adat-formázás |

![](/img/gpt_particio_tipuskod_1.jpg)

### 4.4 §6 példa — partíciónkénti kód

| Partíció | Típuskód | Formázás |
|----------|----------|----------|
| `sdc1` … `sdc3` | **`0C`** | **`vfat`** |
| `sdc4` | **`0F`** (Extended) | **Ne** formázd — konténer |
| `sdc5` … `sdc8` | **`0C`** | **`vfat`** |

> Megjegyzés: MBR partíciós tábla esetében ha formázás után **`83`** vagy **`07`** típuskód látszik akkor, állítsd át **`0C`** típuskódra minden FAT32 fájlrendszer esetében ezeken a partíciókon. A **`0F`** típuskódot a **`sdc4`** extended partícion kell beállítani (A formázás általában már **`0C`**-t állít — lásd §4 első táblázat.).

### 4.5 Fájlrendszer címke *(opcionális)*

**Menü:** **főmenü → `4` → `5` Fajlrendszer cimke** (a Lemez kezeles almenü **ötödik** sora).

| | FAT32 (`vfat`) |
|---|----------------|
| **Mikor?** | Formázás és (ha kell) típuskód után — kötet **leválasztva** |
| **Háttér** | `fatlabel /dev/sdc1 UJ_CIMKE` (`dosfstools`) |
| **Hossz** | Max. **11 karakter**, szóköz nélkül |
| **Kötelező?** | Nem |

**Lépések:** válaszd a partíciót → írd be a címkét (pl. **`CISCO_USB`**) → ha kéri, erősítsd a **leválasztást**, vagy előbb **Kotet lecsatolasa** (`4` → `7`) → ellenőrzés: **Lemez attekintes** → **Enter** → **Fajlrendszer címke** sor.

**Fontos:** Az **`sdc4`** extended konténerre **nincs** fájlrendszer — oda címkét ne állíts.

> Megjegyzés: a címke **nem** helyettesíti a **`0C`** típuskódot. Windows és Linux a címkét kötetnévként mutatja; a Cisco továbbra is a **`usbflash0:`** számot használja.

![](/img/fajlrendszer_cimke_1.jpg)

### 4.6 Cisco Catalyst USB flash — **egyetlen partíció** (IOS / IOS XE)

A **Catalyst 9200**, **9300**, **3850**, **3650** és hasonló switch-ek **külső USB flash** meghajtója IOS image és konfiguráció szempontjából **nem** PC-s többpartíciós pendrive:

| Szabály | Érték |
|---------|--------|
| **Partíciók száma** | **1 db** (pl. `sdc1`) — **ne** hozz létre `sdc2`, `sdc3` … |
| **Partíciós tábla** | **MBR (msdos)** — GPT-s pendrive sok IOS verzión problémás |
| **Fájlrendszer** | **FAT32** (`vfat`) — típuskód **`0C`** |
| **Méret** | **16–32 GB** ajánlott; nagyobb sticken **egy** ~30 GiB partíció, maradék **unallocated** |
| **Switch CLI** | Egy mount: **`usbflash0:`** — a 2. FAT32 partíció **nem** jelenik meg megbízhatóan |

**Wipe → újraépítés Cisco-célra (Partctl):**

1. **Lemez kezeles → Disk cleanup (Wipe)** — teljes lemez + **[✓] Partíciós tábla törlése**.
2. **Particios tabla letrehozasa** — **`2` MBR (msdos)**.
3. **Particio letrehozasa** — **egyszer** (eredmény: `sdc1`); **Vég:** `100%` (≤32 GB) vagy `+30GiB` nagyobb lemezen.
4. **Particio formazas** — `sdc1` → **`vfat`** (automatikus **`0C`**).
5. *(Opc.)* **Fajlrendszer cimke** — pl. **`CISCO_USB`**.

A §6 **több FAT32 partíció** példa **Linux/Windows** környezetre szól (pl. 4 GiB-os fájlhatár megkerülése **PC-n**, külön mount pontokkal) — **nem** helyettesíti a Cisco **egy kötetes** előkészítést. Teljes Cisco menüút és hibaelhárítás: **[Cisco kompatibilis USB flash útmutató](cisco-usb-flash-partctl-guide.md)**.

---

## 5. Példa A — egy FAT32 pendrive (~32 GB) — **Cisco IOS és általános adathordozó**

**Cél:** Egyetlen, jól felismerhető FAT32 kötet — **Catalyst 9200 / 9300 / 3850 / 3650** IOS image és konfigurációhoz (**§4.6**), valamint általános PC-s adathordozónak. Példa lemez: **`/dev/sdc`**, ~29 GiB.

> **Cisco:** csak **`sdc1`** legyen; a **Create partition** menüpontot **ne** futtasd második kötetre. Nagyobb fizikai pendrive-nál a **Vég:** mezőben **`+30GiB`** is elég — a maradék terület maradjon **allokálatlan**.

| Lépés | Menüút | Teendő |
|-------|--------|--------|
| 1 | **Főmenü → `1`** | Lemez: **`sdc`** — ellenőrizd, hogy **nem** a rendszerlemez. |
| 2 | **Főmenü → `4` → `10`** | Wipe: **teljes lemez**, **[✓] Partíciós tábla törlése**. |
| 3 | **Főmenü → `3`** → **Particios tabla letrehozasa** | **`2` — MBR (msdos)**. |
| 4 | **Főmenü → `3`** → **Particio letrehozasa** | **Egyszer** — eredmény **`sdc1`**. Kezdő: **Enter** → **`2048s`**. Vég: **`100%`** (vagy **`+30GiB`** nagyobb lemezen; maradék unallocated). |
| 5 | **Főmenü → `3`** → **Particio formazas** | `sdc1` → **`vfat`**. Utána automatikus típuskód: MBR → **`0C`**, GPT → *Microsoft basic data* (§4). |
| 6 | *(Ellenőrzés)* típuskód | MBR: **MBR particio tipuskod** → **`0C`**, ha még nem az. GPT: részletekben *Microsoft basic data*. |
| 7 | **Főmenü → `2`** | Ellenőrzés: `msdos`, `vfat`, típus **`0C`**. |
| 8 | *(Opc.)* **Főmenü → `4` → `5`** | Címke: pl. **`CISCO_USB`** (max. 11 kar.). |

**Eredmény**

| Eszköz | Tábla | Címke | Fájlrendszer | Megjegyzés |
|--------|-------|-----|---------------|------------|
| `sdc` | `msdos` | — | — | Pendrive |
| `sdc1` | `vfat` | `CISCO_USB` | Max. **~4 GiB / fájl** (FAT32 korlát) | **Egyetlen** kötet — Cisco: `usbflash0:` |
| *(nincs `sdc2`)* | — | — | — | Második partíció **szándékosan nincs** (IOS nem támogatja) |

| Lemez áttekintés | Partíció részletei |
| --- | --- |
| ![usb_partition_1](/img/usb_partition_1.jpg) | ![usb_partition_2](/img/usb_partition_2.jpg) |

---

## 6. Példa B — több FAT32 partíció (**Linux / Windows — nem Cisco IOS pendrive**)

**Cél:** Egy pendrive-on **több külön FAT32 kötet** — **PC-n** külön mount pontokkal, hogy a **4 GiB-os egyfájl-limit** több kötetre legyen „szétosztható” (**egy fájl max. ~4 GiB** partíciónként, de **több partíció = több ilyen fájl**). Példa lemez: **`/dev/sdc`**, ~32 GiB.

> **Fontos — Cisco switch:** A Catalyst 9200 / 9300 / … **nem** kezeli megbízhatóan a több FAT32 partíciót — IOS alatt tipikusan **csak** `usbflash0:` (egy kötet) érhető el. Firmware / IOS image **switch-re** töltéshez használd a **§5** / **§4.6** **egy partíciós** elrendezést és a [Cisco útmutatót](cisco-usb-flash-partctl-guide.md). A §6 példa **Partctl MBR/extended/logical** gyakorlatnak és **Linuxos** archiválásnak szól.

> Megjegyzés: a FAT32 **4 GiB-os határa egyetlen fájlra** vonatkozik, **nem** a partíció méretére. Egy 32 GB-os FAT32 partíción is legfeljebb ~4 GiB lehet egy fájl.

Az MBR **legfeljebb négy primary** partíciót enged — több zónához **extended** konténer kell (**`sdc4`**), benne **logikai** partíciókkal (`sdc5`…`sdc8`).

| Lépés | Menüút | Teendő |
|-------|--------|--------|
| 1 | **Főmenü → `1`** | Lemez: **`sdc`**. |
| 2 | **Főmenü → `4` → `10`** | Wipe: teljes lemez + tábla törlés. |
| 3 | **Főmenü → `3`** → **Particios tabla letrehozasa** | **MBR (msdos)**. |
| 4a–g | **Particio letrehozasa** | Hét partíció: kezdő **`2048s`**, vég **`+4GiB`** (a program javasolt kezdőértékeit használd a 2.–8. sávnál). |
| 5a–h | **Particio formazas** | `sdc1`–`sdc3`, `sdc5`–`sdc8` → **`vfat`**. **`sdc4`** extended: **ne** formázd. |
| 6 | *(Ellenőrzés)* típuskód | Formázás után automatikus: FAT32 → **`0C`** (MBR). **`sdc4`**: **`0F`**. Lásd §4. |
| 7 | *(Opc.)* **Főmenü → `4` → `5`** | Címkék: **`DATA1`** … **`DATA2`**. |
| 8 | **Főmenü → `2`** | Ellenőrzés: nyolc sor, típusok, címkék. |

**Eredmény**

| Partíció | Méret | FS | Címke | Példa tartalom |
|----------|-------|-----|-------|----------------|
| `sdc1` | 4 GiB | `vfat` | `DATA1` | `file1.bin` (≤4 GiB) — **PC mount** |
| `sdc2` | 4 GiB | `vfat` | `DATA2` | `file2.bin` |
| `sdc3` | 4 GiB | `vfat` | `DATA3` | `file3.bin` |
| `sdc4` | ~1 KiB | — | — | Extended (LBA), konténer |
| `sdc5` | 4 GiB | `vfat` | `DATA4` | … |
| `sdc6`–`sdc8` | 4 GiB | `vfat` | `DATA5`–`7` | … |

| Lemez áttekintés | Fájlrendszer címke |
| --- | --- |
| ![usb_multiple_partition_1](/img/usb_multiple_partition_1.jpg) | ![cisco_fat32_2](/img/usb_multiple_partition_2.jpg) |

---

## 7. Példa C — Lemez tisztítás (wipe) egyetlen partíción

**Kiindulás:** a §6 szerinti elrendezés. Csak **`sdc2`** adata törlődik, **`sdc1`** érintetlen marad.

| Lépés | Menüút | Teendő |
|-------|--------|--------|
| 1 | **Főmenü → `1`** | Lemez: **`sdc`**. |
| 2 | **Főmenü → `4` → `10`** | Cél: **`/dev/sdc2`** (ne a teljes lemez!). |
| 3 | Jelölők | **`wipefs`** mindig; opcionálisan **[✓] Partíció nullázás**. Teljes lemez nullázás: **ki**. |
| 4 | Megerősítés | A tábla és **`sdc1`** megmarad. |
| 5 | *(Opc.)* **Particio formazas** | `sdc2` → **`vfat`** újra. |

**Miért jobb, mint a teljes lemez nullázása?** A `dd` csak a **`sdc2`** ~4 GiB sávját írja felül, nem az egész 32 GiB-ot — **kevesebb kopás**.

---

## 8. Példa D — egy partíció törlése a táblából

**Cél:** **`sdc2`** eltűnik a partíciós táblából; **`sdc1`** megmarad; a felszabadult hely később újra partícionálható.

| Lépés | Menüút | Teendő |
|-------|--------|--------|
| 1 | **Főmenü → `1`** | Lemez: **`sdc`**. |
| 2 | **Főmenü → `3`** → **Particio torlese** | Válaszd **`sdc2`**-t, erősítsd meg. |
| 3 | **Főmenü → `2`** | `sdc2` eltűnik; szabad sáv látszik. |

> A törlés **nem** biztonságos adatmegsemmisítés — a régi adat a lemezen maradhat. Erre: **Wipe** + nullázás **törlés előtt**, vagy új partíció + nullázás utána.

Ha **kernel figyelmeztetés** jön: húzd ki és csatlakoztasd újra az eszközt, vagy futtass **`partprobe`**-ot.

---

## 9. Példa E — tábla és a partíciók megmaradnak, de az aláírások törlődnek

**Cél:** a partíciós **szerkezet megmarad** (pl. `sdc1`…`sdc4`), de minden köteten eltűnnek a régi fájlrendszer- és LVM-jelzések — **teljes lemez `dd` nélkül**.

| Lépés | Menüút | Teendő |
|-------|--------|--------|
| 1 | **Főmenü → `4` → `10`** | Cél: **teljes lemez** (`/dev/sdc`). |
| 2 | Jelölők | **[ ] Partíciós tábla törlése** — **ki**. Szükség szerint jelölők / típuskód. |
| 3 | Futtatás | Partíciónkénti `wipefs` (`sdc1`, `sdc2`, …). |

Ha **nincs egyetlen partíció sem**, a program véletlen táblatörlés ellen **nem** wipe-olja a lemez csomópontot — előbb hozz létre partíciókat, vagy kapcsold be a **tábla törlést**.

---

**Napló:** `log/partctl-*.log` mappában megtalálható és a program **Napló** panelján is megtekinthető.

```markdown
https://github.com/drcyberg/partctl/blob/main/example/cisco-usb-flash-partctl-guide.md
```

### Fő oldal (Partctl)

- [Partctl](https://drcyberg.github.io/partctl/web/partctl)

### Kapcsolódó útmutatók

- [Cisco kompatibilis USB flash és USB 3.0 SSD előkészítése](cisco-usb-flash-partctl-guide.md) — **1 db partíció**, FAT32, MBR, Catalyst 9200 / 9300 / …
- [Cisco útmutató (web)](https://drcyberg.github.io/partctl/web/cisco-usb-flash-partctl-guide)

### Köszönöm ha támogatsz

- ***Buy me a coffee***: [LINK](https://buymeacoffee.com/drcyberg)
- ***Paypal (QR Code)***: [LINK](https://github.com/drcyberg/partctl/blob/main/img/qrcode.png)
- ***Paypal (URL)***: [LINK](https://paypal.me/Kunee82)

*Utolsó frissítés jelleg: Partctl V1.0.0 viselkedés — **Cisco IOS USB flash: 1 db FAT32 partíció (MBR)**; §6 többpartíciós példa **PC-re**, nem switch-re. A **Particio kezeles** lista ábécérendje miatt a konkrét **sorszámok** mindig a futó programban ellenőrizendők.*
