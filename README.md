# keycloak-tls
HTTPS-secured Keycloak server using certificates signed by a 3rd party CA.

## Background
I wanted to start using Keycloak in production. This meant two things:
- HTTPS, not the default HTTP.
- 3rd party CA certificates, not the default self-signed certificates.

I wrote this repository because Keycloak's documentation is bad (and no one else has written a complete example, either.)

## By the end of this document, you will have
- free SSL certificates (using Let's Encrypt)
- Keycloak running over HTTPS (without needing a reverse proxy.)

## Limitations of this repo (things I haven't yet figured out)
- no support for reverse proxying.
- no support for custom port numbers.

## Notes:
- All relative paths (in this README) will be given relative the GitHub project directory. (e.g. ~/keycloak-tls)
- This repo has been tested in an Ubuntu 18 environment, running on Amazon EC2. Other distros may need some modification.

# System Requirements
## 1. The following ports must be exposed:
- 80 (for certbot)
- 443 (for certbot)
- 8080 (for keycloak)
- 8443 (for keycloak)

## 2. Register a domain name to your server's public IP.
Create a DNS A record as follows:
- yourdomain.com -> your public ip
- www.yourdomain.com -> your public ip

## 3. Install the following programs
The following must be installed.
- docker (https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-18-04)
- docker-compose (https://docs.docker.com/compose/install/)

# Installation
## Create Docker volumes
The Keycloak database runs as a Service through Docker Compose. All local data is deleted when a Service is stopped. In order to persist data past the Service lifecycle, we need to store the database files in an external location using a Docker Volume:
```
docker volume create keycloak_postgresql_volume
```

For the same reason, we create another volume to hold our SSL certificates. Using a Docker Volume allows multiple Services to share access to the same directory resources:
```
docker volume create letsencrypt_certificates
```

## Generate SSL certificates
Note: replace "keycloak.syntelli.com" with your actual domain name.
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

(Special thanks to Steffen Bleul and the 'blacklabelops' project for making this easy.)

## Comply with Keycloak's hard-coded assumptions 
Note: replace "keycloak.syntelli.com" with your actual domain name.
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

The above step is where I got stuck. Here's an explanation why it's needed:
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

## BEFORE RUNNING DOCKER-COMPOSE (set up the initial username/passwords.)
Note: these instructions assume the Docker Volume "keycloak_postgresql_volume" is still empty.

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

### References

1 http://www.piotrnowicki.com/java/2017/01/09/keycloak-docker-with-ssl-proxy/

2 https://github.com/Codingpedia/codingmarks-api/wiki/Keycloak-Setup-for-Production
