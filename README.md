http://www.piotrnowicki.com/java/2017/01/09/keycloak-docker-with-ssl-proxy/
https://github.com/Codingpedia/codingmarks-api/wiki/Keycloak-Setup-for-Production

# keycloak-tls
docker compose + nginx + keycloak + postgrse


1. install docker

2. docker volume create letsencrypt_certificates

3. get certificates (as follows)
docker run --rm \
    -p 80:80 \
    -p 443:443 \
    --name letsencrypt \
    -v letsencrypt_certificates:/etc/letsencrypt \
    -e "LETSENCRYPT_EMAIL=admin@syntelli.com" \
    -e "LETSENCRYPT_DOMAIN1=keycloak.syntelli.com" \
    -e "LETSENCRYPT_DOMAIN2=www.keycloak.syntelli.com" \
    blacklabelops/letsencrypt install

4. symbolically link the certificates so that Keycloak can find them:
sudo su -
cd /var/lib/docker/volumes/letsencrypt_certificates/_data
mkdir keycloak
cd keycloak
ln -s ../live/keycloak.syntelli.com/cert.pem tls.crt
ln -s ../live/keycloak.syntelli.com/privkey.pem tls.key
exit

#########################

[ references ]
Nell Medina (for a blog post with incomplete instructions.)
- https://nellmedina.github.io/install-keycloak-with-docker/

1. install docker & docker-compose
2. create volumes:
docker volume create keycloak_postgresql_volume

###

1. create a DNS record for the local server
we'll be using "keycloak.syntelli.com"

2. install certbot and use it
sudo certbot certonly --standalone -d keycloak.syntelli.com -d www.keycloak.syntelli.com

3. wip

3. symlink the letsencrpyt to the volume
cd volumes/letsencrypt_certificates
ln /etc/letsencrypt/live/keycloak.syntelli.com keycloak.syntelli.com