# Velyric NFC névjegy – VPS + Cloudflare Tunnel

Ez önálló, statikus weboldal. Nem igényli a Velyric alkalmazást, Node.js-t,
adatbázist vagy MinIO-t. A VPS-en meglévő Docker és Docker Compose elegendő.
A mellékelt beállítás az Nginx hivatalos, stabil 1.30.5-alpine kiadását használja.

Útvonal: NFC-címke → https://nfc.velyric.com → Cloudflare Tunnel →
127.0.0.1:8088 → a névjegyoldal.

## 1. Könyvtár létrehozása a VPS-en

SSH-val belépve, rootként:

```bash
mkdir -p /opt/velyric-nfc
```

## 2. Fájlok feltöltése a saját gépedről

A következő parancsot a Windows PowerShellben futtasd, ne a VPS termináljában.
A `VPS_IP` helyére a VPS valódi IP-címét írd. A gép `tesztuponak` neve nem
feltétlenül használható internetes címként.

```powershell
Set-Location 'C:\Users\upoka\Desktop\Fontosak\Velyric\Velyric-NFC-main'
scp index.html logo.svg apple-touch-icon.png velyric.vcf velyric-de.vcf velyric-en.vcf compose.yaml nginx.conf root@VPS_IP:/opt/velyric-nfc/
```

Ha az SSH nem a 22-es porton működik, a parancsban az `scp` után add meg a
`-P PORTSZAM` kapcsolót. SFTP klienssel is feltöltheted ugyanezt a nyolc fájlt
a `/opt/velyric-nfc/` mappába.

A fájlok közvetlenül ebbe a mappába kerüljenek. A `.htaccess` az Apache-hoz
tartozik; az Nginx számára a névjegy fájltípusát már beállítottuk az `nginx.conf`-ban.

## 3. Indítás a VPS-en

A következő parancsokat ismét a VPS SSH-termináljában futtasd, rootként:

```bash
cd /opt/velyric-nfc
chmod 644 index.html logo.svg apple-touch-icon.png velyric.vcf velyric-de.vcf velyric-en.vcf compose.yaml nginx.conf
systemctl enable --now docker
docker compose config --quiet
docker compose up -d --wait --wait-timeout 120
curl -I http://127.0.0.1:8088/
curl -I http://127.0.0.1:8088/velyric.vcf
```

Mindkét kérésre `200 OK` a várt válasz. A névjegy válaszában
`Content-Type: text/vcard; charset=utf-8` szerepeljen.
Az oldal a háttérben fut, a terminál bezárása után is, és a Dockerrel együtt
újraindul a VPS újraindításakor. A 8088-as port csak a VPS-en belül érhető el.

## 4. Cloudflare Tunnel

A `velyric.com` domainnek Cloudflare-ben kezelt domainnek kell lennie.
A meglévő, ezen a VPS-en futó Tunnel is használható.

A Cloudflare irányítópulton: **Networking → Tunnels → saját tunnel →
Routes → Add route → Published application**. A Zero Trust felületén ugyanez
**Networks → Connectors → saját tunnel → Published application routes**
néven is megjelenhet.

Állítsd be:

| Mező | Érték |
| --- | --- |
| Subdomain | `nfc` |
| Domain | `velyric.com` |
| Path | maradjon üres |
| Service type | `HTTP` |
| Service URL | `127.0.0.1:8088` |

Ha a felület egyetlen teljes URL-t kér, `http://127.0.0.1:8088` legyen az érték.
A látogatók HTTPS-en nyitják meg az oldalt, a helyi szolgáltatás HTTP-n fut.
A publikus névjegy elé ne állíts be Cloudflare Access bejelentkezést.

A fenti cím akkor helyes, ha a `cloudflared` közvetlenül a VPS-en fut rendszer-
szolgáltatásként, ahogy a Velyric telepítője beállítja. Külön Docker-konténerben
futó `cloudflared` esetén a `127.0.0.1` azt a konténert jelenti, más címzés szükséges.

Ellenőrzés a VPS-en:

```bash
systemctl is-active cloudflared
```

Ha `active`, és a Cloudflare felületén a Tunnel `Healthy`, kész a kapcsolat.
Ha a szolgáltatás még nincs telepítve, a Tunnel **Install and Run** részén
látható Linux telepítési parancsot futtasd a VPS-en. A `cloudflared` program
a korábbi naplód alapján már telepítve van, de a Tunnel szolgáltatás beállításáig
a sikertelen Velyric-telepítés még nem jutott el.
Meglévő szolgáltatás esetén ne telepíts rá másodikat; annak állapotát ellenőrizd.

A korábban megosztott Tunnel-token helyett a Cloudflare-ben cserélt új tokent
használd. Tokent ne tölts fel a weboldal fájljai közé.

## 5. Telefonos próba és NFC

Nyisd meg telefonon: https://nfc.velyric.com

Ellenőrizd a névjegy mentését is. Az oldal meglévő kódja mobilon automatikusan
is megpróbálja megnyitni a `velyric.vcf` fájlt; a mentés véglegesítése a telefonon
történik. A „Névjegy mentése” gomb külön is használható.

Ezután az NFC-címkére egy **URL / webhivatkozás** rekordot írj ezzel az értékkel:

```text
https://nfc.velyric.com
```

Az NFC-címke a címet tárolja, a weboldal fájljai a VPS-en maradnak.

## Frissítés és hibakeresés

Módosítás után töltsd fel újra az érintett fájlokat ugyanoda, majd a VPS-en:

```bash
cd /opt/velyric-nfc
docker compose up -d --force-recreate --wait --wait-timeout 120
```

Ez a fájlcsatolásokat is frissíti, ha a feltöltőprogram cserével írja a fájlokat.

Állapot és napló:

```bash
cd /opt/velyric-nfc
docker compose ps
docker compose logs --tail=80 web
journalctl -u cloudflared -n 50 --no-pager
```

Ha helyben nincs `200 OK`, előbb a webkonténer hibáját javítsd.
Ha helyben jó, de a publikus oldal 502-t ad, ellenőrizd a Tunnel célcímét és
a `cloudflared` futási helyét. Ha DNS-rekordütközést ír a Cloudflare,
ellenőrizd a meglévő `nfc` rekord célját, mielőtt módosítod.

Hivatalos leírások:
- https://hub.docker.com/_/nginx
- https://developers.cloudflare.com/tunnel/get-started/
