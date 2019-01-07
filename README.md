# keycloak-tls
docker compose + nginx + keycloak + postgrse


[ references ]
Nell Medina (for a blog post with incomplete instructions.)
- https://nellmedina.github.io/install-keycloak-with-docker/


1. create a DNS record for the local server
we'll be using "keycloak.syntelli.com"

2. install certbot and use it
sudo certbot certonly --standalone -d keycloak.syntelli.com -d www.keycloak.syntelli.com

3. wip

3. symlink the letsencrpyt to the volume
cd volumes/letsencrypt_certificates
ln /etc/letsencrypt/live/keycloak.syntelli.com keycloak.syntelli.com