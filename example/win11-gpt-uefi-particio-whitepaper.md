# Windows 11 rendszer partíciók beállítása — útmutató kizárólag a Partctl (`partctl.sh`) menüin keresztül

> **Cél:** Egy **UEFI + GPT** felépítéshez hasonló, Microsoftos sorrendű partíciók létrehozása **csak** a Partctl ncurses felületén, a **`bash partctl.sh`** indítással — **menüpont → menüpont** útvonallal.  
> A példa a korábban ismertetett **`/dev/sda`** elrendezésre támaszkodik (ESP + MSR + Windows + WinRE).

**Figyelem:** Wipe, particiós tábla újralétrehozása, partíciók létrehozása és formázás **adatvesztéssel** jár. Csak olyan lemezen dolgozz, amelynek minden fontos adata mentve van, és a céllemez ne legyen véletlenül a futó rendszer lemeze.

---

## 0. Folyamat + példa oszlop (`/dev/sda`)

Az alábbi táblázat **ugyanazt a menetet** követi, mint a fejezetek **3–9**, és minden sorhoz hozzárendeli, **mit vársz** a korábban dokumentált **`/dev/sda`** példa-lemezen (≈ **223,6 GiB**, **GPT**, a négy partíció után **marad allokálatlan** hely is).

| Fázis | Partctl útvonal (rövid) | Példa: mit csinálsz `/dev/sda`-n | Eredmény (áttekintés / részletek) |
|-------|-------------------------|-----------------------------------|-----------------------------------|
| Lemez kiválasztás | Főmenü **`1`** → listában **`sda`** | Kiválasztod a **`sda`** sort | A további varázslók a **`/dev/sda`** céllemezt használják. |
| Wipe (opcionális nulláról) | Főmenü **`4`** → **`10`** Disk cleanup (Wipe) → **whole disk** | Teljes lemez cél, szükség szerint **partíciós tábla törlés** bejelölve | Aláírások / (opcionálisan) tábla eltűnik; **üres lemez** felé haladsz. |
| GPT tábla | Főmenü **`3`** → *Create partition table* → **`1` GPT** | Megerősíted a figyelmeztetést | **`gpt`** tábla; még nincs partíció (vagy csak a tábla új). |
| 4× partíció | Főmenü **`3`** → *Create partition* (négy alkalommal) | Lásd §5: pl. **`+512MiB`**, **`+16MiB`**, **`+120GiB`**, **`+1024MiB`** | **`sda1`…`sda4`** létrejön a választott méretekkel. |
| 4× típus-GUID | Főmenü **`3`** → *Partition type code (GUID, GPT)* | Mind a négy partícióra a §6 szerinti listaelem | **`PARTTYPE`** mezők: EFI / MSR / basic data / WinRE GUID (§1 táblázat). |
| Formázás | Főmenü **`3`** → *Partition format* | `sda1` → **vfat**; `sda3` / `sda4` → **ntfs**; `sda2` MSR → kihagyva | Példa: **`sda1`** `vfat`; **`sda3`** `ntfs` (pl. csatolva: `/media/…/Partctl1`); **`sda4`** `ntfs` vagy üres FS jelzés a WinRE típus mellett. |
| (Opc.) WinRE bitek | Főmenü **`3`** → *GPT attributes* → **`sda4`** | *Required* + *No automount* jellegű bitek | Példa `sgdisk`: **`Attribute flags: 8000000000000001`**. |
| Ellenőrzés | Főmenü **`3`** → *Verify GPT*; majd **`2`** áttekintés + **Enter** részletek | Végigmész a négy partíción | GPT OK; áttekintőben **sorrend + méret** egyezik §1-gyel; részletekben **GPT attributum** a WinRE-n (ha `sgdisk` elérhető). |

---

## 1. Példa — célállapot (`/dev/sda`)

| Partíció | Méret (példa) | Fájlrendszer (tipikus) | GPT szerep | Példa: konkrét meta (`/dev/sda`) |
|----------|----------------|------------------------|------------|-----------------------------------|
| `sda1` | 512 MiB | FAT32 / `vfat` | EFI System (ESP) | `PARTTYPE` **EFI System**; GUID **`C12A7328-F81F-11D2-BA4B-00A0C93EC93B`** |
| `sda2` | 16 MiB | *MSR: Windows alatt tipikusan nincs formázva* | Microsoft Reserved (MSR) | `PARTTYPE` **Microsoft reserved**; GUID **`E3C9E316-0B5C-4DB8-817D-F92DF00215AE`** (Linux néha téves FS-jelzést mutathat) |
| `sda3` | 120 GiB | NTFS | Microsoft basic data (rendszer) | `PARTTYPE` **Microsoft basic data**; GUID **`EBD0A0A2-B9E5-4433-87C0-68B6B72699C7`**; példa csatolás: **`/media/llukacsi/Partctl1`** |
| `sda4` | 1 GiB (1024 MiB) | NTFS vagy üres FS jelzés | Windows Recovery (WinRE) | `PARTTYPE` **Windows recovery environment**; GUID **`DE94BBA4-06D1-4D40-A16A-BFD50179D6AC`**; tipikus attribútum: **`0x8000000000000001`** |

A lemez többi része lehet allokálatlan; ez nem akadály a Windows telepítőnek, ha a négy partíció sorrendje és típusa rendben van.

---

## 2. Indítás és közös vezérlés

```bash
bash partctl.sh
```

![](https://github.com/drcyberg/partctl/blob/main/img/terminal_1.jpg)

A launcher a **`python3 -m partctl_ncurses_app`** modult indítja (`PYTHONPATH` + `--lang-dir`).

- **Menü:** kurzor (`Fel` / `Le`, vagy `k` / `j`), **Enter** a kiválasztott sorra.  
- **Gyors választás:** a sor elején látható **`N.`** sorszám begépelése, majd **Enter**.  
- **Vissza / kilépés:** a súgó szerint általában **Backspace** vagy **`q`** (panelenként eltérhet).

**Rögzített főmenü-sorszámok** (minden nyelven ugyanaz a sorrend):

| # | Angol | Magyar (`hu.json`) |
|---|--------|---------------------|
| **1** | Select Disk | Lemez kivalasztasa |
| **2** | Disk Overview | Lemez attekintes |
| **3** | Partition management | Particio kezeles |
| **4** | Disk management | Lemez kezeles |
| **5** | Setup | Beallitasok |
| **6** | About | Rolunk |
| **7** | Exit | Kilepes |

![](https://github.com/drcyberg/partctl/blob/main/img/lemez_kivalasztasa_2.jpg)

Először mindig: **főmenü → `1`** — **Lemez kivalasztasa** — a listában válaszd ki a **`sda`** (vagy cél) sort (**sorszám + Enter** vagy kurzor + Enter).

**Példa (`/dev/sda`):** a lemezlista után a kiválasztott lemez **≈ 223,6 GiB**; a későbbi **Lemez attekintes** ezt a lemezt mutatja **GPT** táblával (ha már felépült a példa-elrendezés).

![](https://github.com/drcyberg/partctl/blob/main/img/lemez_kivalasztasa_1.jpg)

---

## 3. Kiindulás: teljes lemez „wipe” (aláírások + opcionálisan tábla)

**Útvonal:** **főmenü → `4` Lemez kezeles** → a **Lemez kezeles** almenüben a következő sorrend **rögzített** (angol feliratokkal):

| # | Menüpont (angol, `en.json`) |
|---|------------------------------|
| 1 | EXT4 FS check |
| 2 | EXT4 FS size increase |
| 3 | EXT4 FS size decrease |
| 4 | Filesystem repair |
| 5 | Filesystem label |
| 6 | Mount volume |
| 7 | Unmount volume |
| 8 | NTFS FS size increase |
| 9 | NTFS FS size decrease |
| **10** | **Disk cleanup (Wipe)** |
| 11 | Back |

![](https://github.com/drcyberg/partctl/blob/main/img/partctl_menu_1.jpg)

Tehát: **főmenü → `4` → `10` — Disk cleanup (Wipe)**.

![](https://github.com/drcyberg/partctl/blob/main/img/lemez_tisztitas_wipe_1.jpg)

A varázslóban:

1. **Cél kiválasztása:** a listában válaszd a **teljes lemezt** (pl. *whole disk (/dev/sda)* / első sor, ha így jelenik meg).  
2. **Jelölőnégyzetek (Space):** a teljes lemezes célnál megjelenhet többek között a **partíciós tábla törlése (GPT/MBR)** jellegű opció — Win11 „nulláról” felépítéshez érdemes **bekapcsolni**, ha tényleg üres lemezt szeretnél.  
3. **Enter** a végrehajtáshoz, majd erősítse meg a **megerősítő** párbeszédeket.

**Példa eredmény (`/dev/sda`):** a wipe és a tábla-opciók függvényében a **Lemez attekintes**ben a korábbi **`sda1`–`sda4`** sorok **eltűnnek**; a lemez **üres** vagy csak **struktúra nélküli** állapotba kerül, amíg újra fel nem veszed a GPT táblát és a partíciókat.

**Megjegyzés:** A későbbi **„Particios tabla letrehozasa”** varázsló is futtat tisztító lépéseket (`sgdisk` / `wipefs`) a tábla felvétele előtt. A **Wipe** elsősorban fájlrendszer- és LVM-aláírások, illetve explicit „tiszta lap” igénye miatt hasznos **kiindulási** lépés.

---

## 4. GPT particiós tábla felvétele

**Útvonal:** **főmenü → `3` Particio kezeles** → keresd a sort: **Create partition table** / **Particios tabla letrehozasa** (a **Particio kezeles** lista **ábécérendbe** van rendezve — a sorszámot mindig a **képernyőn** ellenőrizd).

1. **Enter** a varázslón.  
2. **Tábla típusa:** válaszd **`1` — GPT** (a másik opció az MBR / `msdos`).  
3. Olvasd el a **figyelmeztetést** (minden meglévő partíció törlődik), majd erősítsd meg.  
4. Szükség esetén oldj fel **csatolásokat**, ha a program kéri (például ha a **`sda3`** még csatolva volt — a példa szerint pl. **`/media/llukacsi/Partctl1`**).

**Példa eredmény (`/dev/sda`):** sikeres **`mklabel gpt`** után az áttekintőben **Partíciós tábla: GPT**, **0** partíció (vagy üres lemez-sor), majd jöhet a §5 négy **Create partition** lépése.

![](https://github.com/drcyberg/partctl/blob/main/img/particio_tabla_letrehozasa_1.jpg)

---

## 5. Négy partíció létrehozása (üres bejegyzések, sorrend: ESP → MSR → Windows → WinRE)

**Útvonal (mind a négy lépésnél):** **főmenü → `3`** → **Create partition** / **Particio letrehozasa**.

![](https://github.com/drcyberg/partctl/blob/main/img/particio_letrehozasa_1.jpg)

A **Particio parameterek** panel a **Start** és **End** mezőket kéri (`parted` szintaxis: pl. `2048s`, relatív méret **`+512MiB`**, vagy a szabad tartomány **vége** `100%`). A program kiírja a szabad **szektortartományt** és a javasolt kezdőértéket — **követd a képernyőn látható** szabad sávot minden új partíció után.

**Javasolt példa-sorozat** (tipikus `512` B szektor mellett; finomhangolás a saját szabad sávod szerint):

| Lépés | Start (általában) | End (példa) | Eredmény (általános) | Példa: eredmény `/dev/sda` |
|-------|---------------------|-------------|----------------------|-----------------------------|
| 1. ESP | a javasolt igazított kezdő (pl. `2048s`) | **`+512MiB`** | ~512 MiB ESP-hez | **`sda1`** — **512 MiB** |
| 2. MSR | a következő szabad kezdő (általában a program javaslata) | **`+16MiB`** | ~16 MiB MSR | **`sda2`** — **16 MiB** |
| 3. Windows | javasolt kezdő | **`+120GiB`** | ~120 GiB rendszerhez (igény szerint más méret) | **`sda3`** — **120 GiB** |
| 4. WinRE | javasolt kezdő | **`+1024MiB`** *vagy* a maradék **`100%`** | ~1 GiB WinRE-hez | **`sda4`** — **1024 MiB** (ha `+1024MiB`; `100%` esetén a **maradék** méret) |

Minden lépés után várható egy **siker / hiba** összegző panel. Ha a negyedik partíciót **`100%`**-ig nyújtod, a WinRE mérete a **maradék** lesz (nem feltétlenül pont 1 GiB).

**Példa (`/dev/sda`):** a négy lépés után az áttekintőben **négy sor** (`sda1`…`sda4`), a lemez végén **allokálatlan** terület is maradhat (a példa-lemezen a partíciók összmérete kisebb, mint a teljes kapacitás).

![](https://github.com/drcyberg/partctl/blob/main/img/particio_parameterek_1.jpg)

---

## 6. GPT típus-GUID beállítása (Partctl listából)

**Útvonal:** **főmenü → `3`** → **Partition type code (GUID, GPT)** / **GPT particio tipuskod** → válaszd ki a partíciót → a listából a megfelelő **GUID + név** pár:

| Partíció | A Partctl listában választandó név (rövidítve) | Példa: eszköz + GUID (`/dev/sda`) |
|----------|-----------------------------------------------|-------------------------------------|
| ESP | **EFI System** | **`sda1`** → `C12A7328-F81F-11D2-BA4B-00A0C93EC93B` |
| MSR | **Microsoft reserved** | **`sda2`** → `E3C9E316-0B5C-4DB8-817D-F92DF00215AE` |
| Windows | **Microsoft basic data** | **`sda3`** → `EBD0A0A2-B9E5-4433-87C0-68B6B72699C7` |
| WinRE | **Windows Recovery Environment (WinRE)** | **`sda4`** → `DE94BBA4-06D1-4D40-A16A-BFD50179D6AC` |

*(A lista a `GPT_GUID_TYPE_CHOICES` bejegyzéseit mutatja.)*

![](https://github.com/drcyberg/partctl/blob/main/img/gpt_particio_tipuskod_1.jpg)

---

## 7. Formázás (ESP: vfat; Windows + WinRE: NTFS)

**Útvonal:** **főmenü → `3`** → **Partition format** / **Particio formazas** → cél partíció → fájlrendszer a listából.

| Partíció | Javasolt formázás a Partctl listájából | Példa: eredmény (`/dev/sda`) |
|----------|----------------------------------------|-------------------------------|
| ESP (`sda1`) | **vfat** / FAT32 (ha szerepel a listán) | **`vfat`** az áttekintőben |
| MSR (`sda2`) | **Hagyd üresen** (Windows tipikusan nem formázza az MSR-t). Ha mégis formázod kísérletként, az nem „Windows hivatalos” viselkedés. | FS nélkül / eszközfüggő jelzés |
| Windows (`sda3`) | **ntfs** | **`ntfs`**; ha csatolod: pl. **`/media/llukacsi/Partctl1`** |
| WinRE (`sda4`) | **ntfs** (a valódi WinRE fájlokat később a Windows telepítő / `reagentc` kezeli) | **`ntfs`** vagy üres típus a WinRE GUID mellett, amíg nincs tartalom |

Formázás előtt a partíciónak **ne legyen biztonságosan** fontos adata; a varázsló **leválasztást** is kérhet.

![](https://github.com/drcyberg/partctl/blob/main/img/formazas_1.jpg)

---

## 8. (Opcionális) WinRE GPT attribútum bitek

**Útvonal:** **főmenü → `3`** → **GPT attributes** / **GPT attributumok** → válaszd a WinRE partíciót → a bitek közül a szükségesek (pl. *Required*, *No automount* — a pontos bitneveket a felület mutatja). **Íráshoz** tipikusan **root** kell.

**Példa (`sda4`):** a háttér `sgdisk` kimenetéhez hasonlóan gyakori érték: **`Attribute flags: 8000000000000001`**; a **Particio reszletek** nézetben összefoglaló hex + bit-címkék jelenhetnek meg.

![](https://github.com/drcyberg/partctl/blob/main/img/gpt_attributumok.jpg)

---

## 9. Ellenőrzés a Partctl-ben (parancssor nélkül)

| # | Lépés | Példa: mit látsz `/dev/sda`-n |
|---|--------|--------------------------------|
| 1 | **főmenü → `3`** → **Verify GPT partition table** / **GPT tabla ellenorzes** | Sikeres ellenőrzés üzenet (integritás OK). |
| 2 | **főmenü → `2` Lemez attekintes** | **GPT** tábla; sorok: **`sda1`** 512 MiB, **`sda2`** 16 MiB, **`sda3`** 120 GiB, **`sda4`** 1 GiB; típus/GUID oszlopok az §1 szerint. |
| 3 | Áttekintőben egy partíción **Enter** — részletek | Pl. **`sda4`**: WinRE GUID; **GPT attributum** sor (ha `sgdisk` elérhető) — lásd §8. |

### Lemez áttekintés
![](https://github.com/drcyberg/partctl/blob/main/img/lemez_attekintes_1.jpg)

### Partíció részletei
![](https://github.com/drcyberg/partctl/blob/main/img/particio_reszletei_1.jpg)

---

## 9.1 Ellenőrzés a Windows 11 OS kitelepítésével

### Helykiválasztás
![](https://github.com/drcyberg/partctl/blob/main/img/win11_1.png)

### Lemezkezelés
![](https://github.com/drcyberg/partctl/blob/main/img/lemez_kezeles_1.png)
![](https://github.com/drcyberg/partctl/blob/main/img/lemez_kezeles_2.png)

---

## 10. Amit a Partctl nem pótol

- **Windows telepítő** / bootmgr / BCD / **WinRE.wim** másolása és **`reagentc`** beállítások.  
- **TPM 2.0**, **Secure Boot**, firmware-beállítások.

---

## 11. Verzió és link

| Mező | Érték |
|------|--------|
| Dokumentum | Win11-szerű GPT elrendezés — `partctl.sh` menüútvonalak + **`/dev/sda` példa oszlop** |
| Példa lemez | `/dev/sda` — ~223,6 GiB lemez, GPT, `sda1`…`sda4` (512 MiB / 16 MiB / 120 GiB / 1 GiB), maradék allokálatlan hely lehetséges |

README-ből:

Win11 GPT — Partctl menü útmutató
```markdown
https://github.com/drcyberg/partctl/example/win11-gpt-uefi-particio-whitepaper.md
```
