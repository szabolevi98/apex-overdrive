# Apex Overdrive — telepítési útmutató

Ez nyilvános, általános telepítési leírás. A példákban szereplő SSH-host,
felhasználó, port és kulcs helyettesítő érték; a saját környezetedhez igazítsd őket.
Éles hozzáférési adatokat, szerverleltárt és titkok tárolási helyét ne írd a repóba.

A projekt repója: `https://github.com/szabolevi98/apex-overdrive`.
A publikus játék: `https://racing.levente.net/`.

## Követelmények és felépítés

- Node.js 20 vagy újabb.
- Linuxon systemd; HTTPS-hez Apache és érvényes tanúsítvány.
- Opcionálisan MySQL/MariaDB a tartós eredménytároláshoz.
- Elegendő tárhely az autó-, pálya- és környezetmodelleknek.

A kliens nyers ES-modulokat használ, külön kliens-build nem szükséges.
Apache közvetlenül szolgálja ki a `web/` könyvtárat és a `/shared/` útvonalat.
Az `/api/` és `/ws` kéréseket a helyi Node-folyamatnak továbbítja.

```text
böngésző → Apache HTTPS → web/ és shared/
                       → /api/ és /ws → Node a loopback interfészen
```

Más webhelyek mellé külön VirtualHostot hozz létre.

## SSH-kapcsolat

A saját gépeden állíts be SSH-hostaliast. A következő `~/.ssh/config` részlet
kizárólag példa, nem az üzemelő játék kapcsolati adata:

```sshconfig
Host apex-overdrive-server
    HostName server.example.com
    User deploy
    Port 22
    IdentityFile ~/.ssh/example-deploy-key
    IdentitiesOnly yes
```

A további parancsok ezt a helyi aliast használják. A privát kulcsot soha ne
másold a repóba. A GitHubhoz használt szerveroldali deploy key ettől külön
kulcs legyen; ha csak kódot tölt le, elegendő a read-only jogosultság.
A szerver SSH-hostkulcsát megbízható csatornán ellenőrizd.

## Kód és assetek

A következő példák telepítési könyvtára `/opt/racing`; ha mást használsz,
a systemd- és Apache-mintákat is hozzá kell igazítani.

```bash
git clone git@github.com:szabolevi98/apex-overdrive.git /opt/racing
cd /opt/racing
npm ci --omit=dev
cp .env.example .env
```

A nagy bináris assetek nincsenek Gitben: GLB, GLTF, HDR, EXR, BIN és
pályatextúrák külön feltöltést igényelnek. A kézzel készített metaadatok,
például a rajtpontok, kapuk, boxbeállítások és zónatérképek verziókövetve vannak.

Helyi gépről, megfelelő céloldali írási jogosultsággal:

```bash
rsync -avP web/assets/ apex-overdrive-server:/opt/racing/web/assets/
```

Egy-egy fájl `scp`-vel is feltölthető. A célkönyvtárnak előbb léteznie kell.
Új pályához a `collision.bin`, autónál pedig a
`web/assets/cars/compressed/` alatti remote/ghost modell is szükséges.
Első telepítéskor egy autóval, egy pályával és egy égbolttal ellenőrizhető a rendszer.

## Környezeti beállítások és adatbázis

A `.env` fájlt kizárólag a szerveren töltsd ki; Gitben nem szerepelhet.
A jelszó az alábbi mintában kifejezetten helyettesítő érték:

```ini
HOST=127.0.0.1
PORT=3000
ALLOW_DEV_WRITES=0
DB_HOST=127.0.0.1
DB_PORT=3306
DB_NAME=example_game
DB_USER=example_game_user
DB_PASSWORD=REPLACE_WITH_A_UNIQUE_SECRET
```

Hozz létre külön adatbázist és olyan adatbázis-felhasználót, amely csak ehhez
az adatbázishoz fér hozzá. A táblákat az alkalmazás induláskor létrehozza.
Adatbázis nélkül is elindul, de a tartós eredménytárolás nem működik.

Élesben az `ALLOW_DEV_WRITES=0` kötelező: a fejlesztői mentési végpontok
nem publikus használatra készültek. A `HOST=127.0.0.1` a proxy mögött tartja
a Node-kiszolgálót; annak portját ne tedd közvetlenül elérhetővé az internetről.

A szolgáltatás külön, bejelentkezésre nem használható rendszerfelhasználóval fusson.
A kódot olvashassa, de ne írhassa; a `.env` csak az adminisztrátor és a
szolgáltatás felhasználója számára legyen olvasható, például `640` jogosultsággal.

## systemd

A követett minta: [deploy/systemd/racing.service](deploy/systemd/racing.service).
A benne szereplő rendszerfelhasználót és könyvtárat a telepítéshez igazítsd,
majd másold a megfelelő systemd-könyvtárba.

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now racing
sudo systemctl status racing
journalctl -u racing -n 50
```

Az alkalmazás a munkakönyvtár `.env` fájlját `dotenv` segítségével olvassa.
A szolgáltatás mintája ezért nem ad meg külön `EnvironmentFile` sort.

## Apache és HTTPS

A követett VirtualHost-minta:
[deploy/apache/racing.levente.net-le-ssl.conf](deploy/apache/racing.levente.net-le-ssl.conf).
Más domainnél a `ServerName`, a tanúsítvány útvonala és a könyvtárak is módosítandók.

Szükséges modulok:

```bash
sudo a2enmod proxy proxy_http proxy_wstunnel ssl rewrite headers deflate
sudo apache2ctl configtest
```

A konfigurációban:

- `DocumentRoot`: a projekt `web/` könyvtára.
- `Alias /shared/`: a projekt `shared/` könyvtára.
- `/ws`: `ws://127.0.0.1:3000/ws`.
- `/api/`: `http://127.0.0.1:3000/api/`.
- A gyökérútvonalat Apache szolgálja ki, nem a Node-proxy.
- A nagy asseteknél a MIME-típus, range-kérések és cache-fejlécek is számítanak.

A saját domainedhez állíts be DNS-t és érvényes TLS-tanúsítványt, valamint
ellenőrzött automatikus megújítást. A HTTP-kéréseket irányítsd HTTPS-re.
Sikeres konfigurációellenőrzés után alkalmazd a beállítást:

```bash
sudo systemctl reload apache2
```

## Frissítés

A szerveren, a telepítést kezelő felhasználóval:

```bash
cd /opt/racing
git pull --ff-only
# Csak függőségváltozás esetén:
npm ci --omit=dev
# Szerverkód vagy autó-/pálya-asset változása után:
sudo systemctl restart racing
sudo systemctl is-active racing
```

Az asset-manifestet a szerver gyorsítótárazza, ezért új vagy módosított
játékasset feltöltése után újraindítás szükséges. A Git pull önmagában
nem másolja fel a Gitből kizárt modelleket és ütközési fájlokat.
A csak statikus arculati fájlokat érintő frissítéshez nem kell újraindítás.

Feltöltés után hasonlítsd össze a helyi és távoli fájl méretét vagy SHA-256-át.
Nagy asseteknél a tartalomverziózott URL biztosítja a cache frissülését.

## Ellenőrzés és hibakeresés

A példabeli hostot cseréld a telepítés publikus domainjére:

```bash
curl -I https://game.example.com/
curl https://game.example.com/api/status
curl https://game.example.com/api/assets
```

Elvárt: a főoldal és az API 200-as választ adjon, a manifest az aktuális
asseteket tartalmazza. A WebSocket-kapcsolat upgrade-válasza
`101 Switching Protocols`; asset-range kérésre `206` várható.
Böngészőből ellenőrizd a menüt és a többjátékos kapcsolatot is.

- Főoldalhiba: VirtualHost, fájljogosultságok és Apache-napló.
- API-hiba: szolgáltatásállapot, Node-napló és loopback API-elérés.
- WebSocket-hiba: proxy-modulok és a `/ws` szabály sorrendje.
- Üres assetlista: feltöltött fájlok, olvasási jog és manifestfrissítés.
- Hiányzó pályaütközés: a megfelelő `collision.bin` feltöltése.

A saját szerver leltárát, kapcsolati adatait és privát üzemeltetési jegyzeteit
a nyilvános repótól elkülönítve tárold.
