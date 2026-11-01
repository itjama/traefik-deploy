# traefik-deploy

Spoločná reverse proxy (Traefik v3) pre aplikácie na VPS. Automatické HTTPS (Let's Encrypt, HTTP challenge),
presmerovanie HTTP → HTTPS. Zvonku sú otvorené iba porty 80 a 443 tohto kontajnera; aplikácie nemajú `ports:`.

| Aplikácia | Doména | Repo | Priečinok na VPS |
|---|---|---|---|
| Traefik | – | `traefik-deploy` | `/opt/traefik` |
| Hypergrid | `hypergrid.app.iteo.com` | `hypergird-deploy` | `/opt/hypergrid` |
| Spolu | `spolu.app.iteo.com` | `household-deploy` | `/opt/household-deploy` (existujúci) |

Aplikácia sa pripojí tak, že jej webová služba je v externej sieti `proxy` a má labely:

```yaml
labels:
  - traefik.enable=true
  - traefik.http.routers.<meno>.rule=Host(`<doména>`)
  - traefik.http.routers.<meno>.entrypoints=websecure
  - traefik.http.services.<meno>.loadbalancer.server.port=<port v kontajneri>
```

`<meno>` musí byť v každej aplikácii iné. Databázy nikdy nie sú v sieti `proxy`.

---

## Ručné kroky (Majo)

### 1. DNS na Websupporte
V zóne `iteo.com` pridať záznam `A`, názov `*.app`, hodnota = IP adresa VPS. Ak má VPS IPv6, pridať aj `AAAA`.
Overiť: `dig +short hypergrid.app.iteo.com` a `dig +short spolu.app.iteo.com` vrátia IP VPS.

### 2. Firewall
Povoliť iba porty 22, 80 a 443 (`ufw` aj firewall v paneli poskytovateľa VPS).
Pravidlo `443/udp` (HTTP/3 pri Caddy) už netreba.

### 3. Spustenie na VPS

Poradie je dôležité: Caddy zo Spolu drží porty 80/443, preto sa najprv nasadí nový `household-deploy` (bez Caddy),
až potom Traefik. Medzi týmito dvoma krokmi Spolu krátko nebude dostupné.

```bash
docker network create proxy

# Spolu — existujúci priečinok a dáta (projekt "household-deploy", volume sa nemenia)
cd /opt/household-deploy && git pull
nano .env                                  # DOMAIN=spolu.app.iteo.com
./backup.sh db                             # záloha pred zmenou
./deploy.sh                                # odstráni starý kontajner caddy (--remove-orphans)

# Traefik
cd /opt/traefik && cp .env.example .env    # ACME_EMAIL
docker compose up -d

# Hypergrid
cd /opt/hypergrid && cp .env.example .env  # doplniť hodnoty
./deploy.sh 0.1.0
```

Po overení zmazať nepoužívané volume Caddy: `docker volume rm household-deploy_caddy_data household-deploy_caddy_config`.

### 4. Overenie
- Otvoriť `https://hypergrid.app.iteo.com` a `https://spolu.app.iteo.com` — platný certifikát.
- `curl -sI http://spolu.app.iteo.com | head -3` → presmerovanie na `https://`.
- Pri problémoch: `cd /opt/traefik && docker compose logs traefik --tail=100`
  (chyby ACME = DNS ešte nesmeruje na VPS alebo je zatvorený port 80).

### 5. Zálohy
- Pravidelný `pg_dump` oboch databáz (`backup.sh` v oboch deploy repách, cron) a kópia záloh mimo VPS.
- Zálohovať `/opt/traefik/letsencrypt` (obsahuje `acme.json` s privátnymi kľúčmi certifikátov — nikdy do gitu).
