http://www.piotrnowicki.com/java/2017/01/09/keycloak-docker-with-ssl-proxy/
https://github.com/Codingpedia/codingmarks-api/wiki/Keycloak-Setup-for-Production

# keycloak-tls
docker compose + letsencrypt SSL + keycloak + postgres


## Notes:
- All relative paths (in this README) will be given relative the GitHub project directory. (e.g. ~/keycloak-tls)
- This repo is tested Ubuntu 18 running on Amazon EC2. Other distros may need some modification.


# System Requirements
## 1. The following ports must be exposed:
- 8080
- 8443

## 2. in your domain registrar, create two A records:
- yourdomain.com -> your public ip
- www.yourdomain.com -> your public ip

## 3. Install the following programs.
- docker
- docker-compose
- certbot

# Installation
## Set up SSL
First we need to create a Docker volume. This will act as shared storage between our Docker containers.
```
docker volume create letsencrypt_certificates
docker volume create keycloak_postgresql_volume
```

Next we'll use Docker to geenrate our SSL certificates.
In the following code, replace "keycloak.syntelli.com" with the DNS Alias you configured earlier.
(Special thanks to Steffen Bleul and the 'blacklabelops' project for making this easy.)
```
docker run --rm \
    -p 80:80 \
    -p 443:443 \
    --name letsencrypt \
    -v letsencrypt_certificates:/etc/letsencrypt \
    -e "LETSENCRYPT_EMAIL=admin@syntelli.com" \
    -e "LETSENCRYPT_DOMAIN1=keycloak.syntelli.com" \
    -e "LETSENCRYPT_DOMAIN2=www.keycloak.syntelli.com" \
    blacklabelops/letsencrypt install
```

The next step is to wire Let's Encrypt into Keycloak.
```
sudo su -
cd /var/lib/docker/volumes/letsencrypt_certificates/_data
chown 1000:root archive
chown 1000:root live
ln -s live/keycloak.syntelli.com/cert.pem tls.crt
ln -s live/keycloak.syntelli.com/privkey.pem tls.key
chown 1000:root tls.crt
chown 1000:root tls.key
```

> If you're curious what that just did:
```
Let's Encrypt provided us with free SSL certificates, which go into our Docker volume.

Unfortunately, Keycloak defaults to self-signed certificates (which are verboten in production.)

Fortunately, JBoss's official Keycloak Docker image supports external CA's (aka Let's Encrypt.)

Unforunately, the expected filenames ("tls.crt" and "tls.key") are hardcoded.
The expected filepath is ALSO hardcoded ("/etc/x509/https" directory.)
Finally, the files need to be readable by user "jboss" (ID 1000.)

So we need to provide these hard-coded expectations manually, by editing our Docker volume.
Fortunately, the volume is stored as a directory in the host OS (because we used an external Docker volume.)

So the above code does the following things:
1. We change directory into the path (which we know from `docker volume inspect letsencrypt_certificates`.)
2. We then create the "tls.crt" and "tls.key" files as symbolic links.
3. Finally, we modifiy the ownership of the files.

That's it--this is everything needed to import the external CA certificate into the keystore.
(special thanks to: https://developer.jboss.org/thread/278360?_sscc=t for the ultimate answer.)
```

## Change the usernames and passwords:
The keycloak admin console defaults to "admin/admin".

The postgres database user defaults to "keycloak/keycloak".

To change these, edit the `docker-compose.yml` file's environment variables:
```
postgres:
    ....
    environment:
          - POSTGRES_DB=keycloak
          - POSTGRES_USER=keycloak
          - POSTGRES_PASSWORD=keycloak
          - POSTGRES_ROOT_PASSWORD=keycloak
    ...
keycloak:
    ...
    environment:
        - KEYCLOAK_USER=admin
        - KEYCLOAK_PASSWORD=admin
        - DB_DATABASE=keycloak
        - DB_USER=keycloak
        - DB_PASSWORD=keycloak
    ...
```

## Change the listening ports:
I have no idea how to do this. If you figure it out, let me know.

## Deploy the server
Run the following to deploy any changes to the 'docker-compose.yml' file:

```git pull && docker-compose build && docker-compose down && docker-compose up -d```

Run the following to view logs:

```docker-compose logs keycloak```

Note that the Keycloak container takes about 30-40 seconds to spin up.
