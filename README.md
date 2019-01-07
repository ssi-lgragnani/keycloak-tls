http://www.piotrnowicki.com/java/2017/01/09/keycloak-docker-with-ssl-proxy/
https://github.com/Codingpedia/codingmarks-api/wiki/Keycloak-Setup-for-Production

# keycloak-tls
docker compose + nginx + keycloak + postgres


Notes:
- All relative paths (in this README) will be given relative the GitHub project directory. (e.g. ~/keycloak-tls)
- This repo is tested Ubuntu 18 running on Amazon EC2. Other distros may need some modification.


System Requirements / Environment setup;
1. The following ports must be exposed:
- 80
- 443
- 8080
- 8443

2. in your domain registrar, create two A records:
- yourdomain.com -> your public ip
- www.yourdomain.com -> your public ip


installation steps:
1. clone this repository.
    cd ~
    git clone https://github.com/ssi-lgragnani/keycloak-tls.git
    cd keycloak-tls
2. install docker
    google it
3. install docker-compose
    google it
4. install certbot
    google it


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
chown 1000:root archive
chown 1000:root live
ln -s live/keycloak.syntelli.com/cert.pem tls.crt
ln -s live/keycloak.syntelli.com/privkey.pem tls.key
chown 1000:root tls.crt
chown 1000:root tls.key


###########
ignore everything below this line.
openssl pkcs12 -export -name server-cert -in tls.crt -inkey tls.key -out serverkeystore.p12
# (press enter twice to leave password blank)
chown ubuntu:root serverkeystore.p12
exit


#########################
setup the keystore
docker-compose exec keycloak bash
cd /etc/x509/https
keytool -importkeystore -destkeystore ~/keycloak.jks -srckeystore serverkeystore.p12 -srcstoretype pkcs12 -alias server-cert

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