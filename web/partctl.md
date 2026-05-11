# Partctl V1.0.0 — Felhasználói kézikönyv

A `Partctl` program egy ncurses (pythone) alapú terminal (CLI) lemez-, és partíció kezelő eszköz Linuxra OS rendszerekhez tervezve. Az ötletet a GParted program adta. A projekt két fő belépési pontra bontható:

- **`partctl.sh`**: A Partctl ncurses alkalmazás indítója.
- **`setup.sh`**: Telepítő és egyben ellenőrző ncurses alkalmazás (függőségek ellenőrzése, telepítés Internet/Lokális módban)

| Telepítés | Menü rendszer | Lemez áttekintés |
| --- | --- | --- |
| ![telepites](/partctl/img/telepites_1.png "Telepítés") | ![menu_rendszer](/partctl/img/menu_1.png "Menü rendszer") | ![lemez_attekintes](/partctl/img/lemez_attekintes_1.jpg "Lemez áttekintes") |

## Fontos biztonsági megjegyzés

Ez az eszköz **adatvesztést okozó** műveletekre képes (tábla törlés/újralétrehozás, formázás, wipe, partíció törlés, méretezés). Mindig legyen mentésed, és csak akkor folytasd, ha pontosan érted a kiválasztott művelet hatását.

## Gyors kezdés

1. Terminálban lépj a projekt mappájába (Partctl-V1-0-0)
2. Futtasd a telepítőt/ellenőrzőt rendszergazdai jogosúltsággal (sudo/root):

```bash
bash setup.sh
```

3. Futtasd a Partctl programot rendszergazdai jogosúltsággal (sudo/root):

```bash
bash partctl.sh
```

## Követelmények

- **Linux OS**: Debian, Ubuntu
- **Program**: python3, util-linux, parted, gdisk, lvm2, e2fsprogs, dosfstools, ntfs, kpartx, fuser
- **Terminál méret**: javasolt legalább kb. 124×24 (a nagyon kicsi termináloknál a UI korlátozott)
- **Minimum képernyőfelbontás**: 1024x768

A pontos függőségi listát a `setup.sh` **Ellenőrzés** menüpontja kijelzi.

## Telepítés a `setup.sh`-val

A `setup.sh` egy ncurses telepítő UI-t indít (`setup_ncurses.py`), amely:

- **Ellenőrzi** a szükséges parancsokat/csomagokat
- **Telepíti** a hiányzókat
- Kezeli a **Nyelv** és a **Csomagkezelés (Lokális/Internet)** beállításokat (`conf/config.json`)

### Csomagforrás módok

- **Internet**: a rendszer csomagkezelőjét használja (pl. Ubuntu/Debian: `apt-get update` + `apt-get install -y ...`).
- **Lokális (offline)**: `.deb` csomagokat telepít a projekt `pack/` mappájából.

### Lokális telepítés (offline) – mappa kiválasztás disztró alapján

Ubuntu/Debian (APT) esetén a setup a rendszer `Disztro:` értékéből (pl. `Ubuntu 24.04.4 LTS`) automatikusan választ:

- Példa: **Ubuntu 24.04.x → `pack/apt/ubuntu-24-04/`**

Ha a lokális csomag mappa hiányzik vagy nem található benne `.deb`, a setup egy "Warn" panelen jelzi.

## A Partctl futtatása (`partctl.sh`)

Indítás:

```bash
bash partctl.sh
```

### Fő funkciók (röviden)

- **Lemez kezelés**: Lemez kiválasztása és áttekintése, partíció és a kötet részletek megtekintése
- **Partíció menedzsment**: létrehozás, törlés, átnevezés, méretezés, formázás, flag-ek, typecode (MBR/GPT), GPT műveletek, LVM kezelés
- **Lemez menedzsment**: ideiglenes csatolás/leválasztás, fájlrendszer javítás (támogatott típusoknál), wipe (aláírások törlése)
- **Napló panel**: a műveletek és a UI üzenetek is visszakereshetők
- **(Szín)Jelzések**: menü elemeknek a sorszám színjelzései: sárga = további almenü; zöld = művelet indító; sárga panel = figyelmeztetés; zöld panel = információ ; piros panel = hiba
- **Navigálás**: numerikus számkombináció és a kurzor billentyű (navigációs billentyű) = menü elemek kiválasztása; backspace/q = vissza; enter = művelet indítás; tab = opció kiválasztása, R = újratöltés; PgUp/PgDown/Home/End = gyors lapozás
- **Nyelv**: magyar, angol

## Naplózás (logok)

A logok a `log/` mappába kerülnek.

- **Partctl**:
  - Launcher log: a `partctl.sh` hozza létre és átadja a Python UI-nak
  - Futás közbeni log: `log/partctl-YYYYMMDD-HHMMSS.log`
- **Setup**:
  - Launcher log: `log/setup-launcher-YYYYMMDD-HHMMSS.log`
  - Ncurses runtime log: `log/setup-ncurses-YYYYMMDD-HHMMSS.log`
  - Összefoglaló: `log/setup-ncurses-summary-*.log` és `log/setup-ncurses-summary-latest.log`

### UI üzenetek a naplóban

A Partctl és a Setup is úgy van kialakítva, hogy a felugró **Info/Warn/Error** panelon az üzenetek és a műveleti eredmények **naplózódjanak** (így később is visszanézhetők a log panelen / log fájlban).

## Nyelv (i18n)

- Partctl: `lang/en.json`, `lang/hu.json`
- Setup: a `setup_nc_*` kulcsok a fenti nyelvi fájlokban

A setup és a Partctl UI-ban a nyelv a menüből választható, és a beállítás mentésre kerül.

## Hibaelhárítás

### `command not found` admin eszközöknél (Debian/Ubuntu)

Egyes admin parancsok (`fdisk`, `sfdisk`, `lvm2` eszközök, stb.) gyakran `/usr/sbin` alatt vannak (Debian OS).
Az indítók ezért kiegészítik a PATH-ot:

- `export PATH="/usr/sbin:/sbin:${PATH}"`

Ha mégis hiányt jelez, futtasd a `setup.sh`-t (Ellenőrzés/Telepítés), és nézd meg a logot a `log/` alatt.

### Terminál túl kicsi

Növeld a terminál ablak méretét (vagy használj nagyobb betűméretet), majd nyomj egy gombot az újrarajzoláshoz.

## Licenc / Projekt

A projekt információi a program **About / Rólunk** menüpontjában találhatók.

## További fejlesztési pontok

- [ ] További támogatott OS: Windows (WSL), Fedora, Arch
- [ ] S.M.A.R.T. támogatás (smartmontools)
- [ ] Tartós kötet felcsatolása és menedzselése (fstab)
- [ ] SWAP beállítás menedzselése (swapon, swapoff)
- [ ] Lemeztitkosítási módszer menedzselése (LUKS)
- [ ] További fájlrendszer támogatás bevezetése: btrfs, xfs, zfs, jfs
- [ ] Lemez monitorozása grafikon ábrával (iostat, ttyplot, iotop)
- [ ] MBR partíció tábla biztonsági mentés készítés
- [ ] Lemez-, és fájlrendszer klónozás és biztonsági mentés készítés
- [ ] Web UI bevezetés
- [ ] Oktatóanyaggal és esettanulmánnyal kapcsolatos oldalak létrehozása

### Esettanulmány (Case Study)

- [Windows 11 rendszer partíciók beállítása](https://drcyberg.github.io/partctl/web/win11-gpt-uefi-particio-whitepaper)

### Köszönöm ha támogatsz

- ***Buy me a coffee***: [LINK](https://buymeacoffee.com/drcyberg)
- ***Paypal***: [LINK](https://github.com/drcyberg/partctl/blob/main/img/qrcode.png)
