# docker-nginx

Reverse-proxy nginx (`nginx:stable`) auto-mis à jour par un watchtower externe (label `com.centurylinklabs.watchtower.enable=true`), avec logs envoyés dans journald (pilote de logs Docker) et analysés par CrowdSec sur l'hôte.

## Arborescence

- `conf.d/docker-nginx.conf` : conf partagée niveau `http`, commentée (templates générés, TLS 1.3, HTTP/2, zones de rate limiting, serveur par défaut sur le port 80, include des services)
- `templates/` : templates `envsubst` remplis au démarrage dans `/etc/nginx/generated` (`00-resolver.conf` avec le resolver lu depuis `/etc/resolv.conf`, `00-acme.conf` avec l'émetteur Let's Encrypt et `ACME_EMAIL`)
- `snippets/` : à inclure dans les `server {}` des services (`include snippets/security-headers.conf;`, `include snippets/ssl-hardening.conf;`)
- `certs/` : état du module ACME (clé de compte, certificats), non versionné
- `services.d/` : confs nginx des services (`*.conf` à la racine uniquement : nginx n'a pas d'include récursif) ; pointer `SERVICES_DIR` vers le clone de `raspi-01-nginx-conf` dans `.env`

## Déploiement

Déploiement manuel (Ansible ne gère que la structure des serveurs : Docker, CrowdSec…) :

```sh
git clone git@github.com:Haelle/docker-nginx.git ~/apps/docker-nginx
git clone git@github.com:Haelle/raspi-01-nginx-conf.git ~/apps/raspi-01-nginx-conf
cd ~/apps/docker-nginx
cp .env.example .env   # ACME_EMAIL, SERVICES_DIR=/home/<user>/apps/raspi-01-nginx-conf
docker compose up -d
```

`ACME_DIRECTORY` (défaut : production Let's Encrypt) permet de pointer sur le serveur de staging pendant les tests : les validations HTTP-01 ratées comptent dans les limites de production. Vider `certs/` en repassant en production.

Recharger la conf :

```sh
docker compose down && docker compose up -d
```

Toute commande `nginx` lancée dans le conteneur relit la conf et doit porter le `-g`, sinon elle échoue sur `unknown directive "acme_issuer"` : c'est le cas de `nginx -s reload` et `nginx -s reopen`.

## Certificat d'un service

```nginx
server {
    listen 443 ssl;
    server_name app.example.com;

    acme_certificate    letsencrypt;
    ssl_certificate     $acme_certificate;
    ssl_certificate_key $acme_certificate_key;
    ...
}
```

Le challenge HTTP-01 est intercepté par le module sur tout `server` en écoute sur le port 80, y compris un bloc `if ($host = ...) { return 301 ...; } return 404;` ; à défaut, le serveur par défaut de `conf.d/docker-nginx.conf` le traite et redirige le reste en HTTPS.

## Choix

- `network_mode: host` : nginx voit les vraies IP clientes et le trafic passe par la chaîne `INPUT`, donc les bans CrowdSec (bouncer nftables) et les règles UFW s'appliquent (avec des ports publiés, Docker passe par `FORWARD` et contourne les deux). Les services sont joints via `127.0.0.1:<port>`.
- Logs : l'image envoie `access.log` sur stdout et `error.log` sur stderr ; le pilote `journald` les écrit dans le journal systemd avec le champ `CONTAINER_NAME=nginx` (rotation gérée par journald, pas de logrotate). CrowdSec les lit via le socket Docker (source `docker`, `container_name: nginx`, cf. rôle `install_crowdsec` de home_cloud_ansible) : le nom du conteneur doit rester `nginx`. Consultation : `journalctl CONTAINER_NAME=nginx -f` ou `docker logs -f nginx` ; `-p err` pour les seules erreurs (stderr).
- Zones de rate limiting (`sync`, `web`, `auth`, `media`, `download`, `admin`, `chat`) dans `conf.d/docker-nginx.conf`, statut 429 : chaque rejet est journalisé « limiting requests » et déclenche le scénario CrowdSec `nginx-req-limit-exceeded`.
- TLS 1.3 uniquement (`conf.d/docker-nginx.conf`, profil « modern » de Mozilla) : pas de `ssl_ciphers`, TLS 1.3 choisit ses suites. `ssl_protocols` est au niveau `http` et non dans `snippets/ssl-hardening.conf` pour qu'un service puisse le surcharger : déclaré deux fois dans un même `server`, nginx cumule les valeurs au lieu de remplacer (simple warning).
- TLS : module ACME officiel (`ngx_http_acme_module`, livré dans `nginx:stable`) chargé via `command: nginx -g "load_module ..."` : `load_module` est interdit dans `conf.d` (inclus dans `http {}`) et on ne remplace pas le `nginx.conf` de l'image pour garder ses évolutions avec les auto-updates. `/etc/letsencrypt` est monté en lecture seule pour inspection.
