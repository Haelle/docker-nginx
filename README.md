# docker-nginx

Reverse-proxy nginx (`nginx:stable`) auto-mis à jour par un watchtower externe (label `com.centurylinklabs.watchtower.enable=true`), avec logs fichiers dans `/var/log/nginx/` lus par fail2ban sur l'hôte.

## Arborescence

- `conf.d/` : confs partagées (`00-rate-limiter`, `00-security-headers`) et `zz-services.conf` qui inclut les confs des services
- `templates/` : templates `envsubst` remplis au démarrage (`00-resolver.conf`, resolver lu depuis `/etc/resolv.conf`)
- `snippets/` : à inclure dans les `server {}` des services (`acme-challenge.conf`)
- `services.d/` : confs nginx déposées par les autres applis (surcharger avec `SERVICES_DIR=/chemin`)
- `logrotate/nginx` : à copier dans `/etc/logrotate.d/nginx`

## Démarrage

```sh
sudo mkdir -p /var/log/nginx /var/www/certbot
sudo cp logrotate/nginx /etc/logrotate.d/nginx
docker compose up -d
```

Recharger après ajout d'une conf de service : `docker exec nginx nginx -t && docker exec nginx nginx -s reload`

## Choix

- `network_mode: host` : nginx voit les vraies IP clientes et le trafic passe par la chaîne `INPUT`, donc les bans fail2ban et les règles UFW s'appliquent (avec des ports publiés, Docker passe par `FORWARD` et contourne les deux). Les services sont joints via `127.0.0.1:<port>`.
- Le bind mount de `/var/log/nginx` masque les symlinks `stdout`/`stderr` de l'image : nginx écrit dans `access.log` / `error.log`, chemins par défaut des jails `nginx-*` de fail2ban.
- `zone=one` (rate limiter) sert au jail `nginx-limit-req` : `limit_req zone=one burst=10;` dans les services.
- TLS : certs certbot montés en lecture seule depuis `/etc/letsencrypt` ; renouvellement en mode webroot (`certbot certonly --webroot -w /var/www/certbot`), le plugin `--nginx` ne fonctionne plus.
