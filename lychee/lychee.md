# Instal·lació i desplegament de Lychee amb Podman, MariaDB i Nginx

Intentarem explicar com hem posat en marxa Lychee amb MariaDB i Nginx fent servir Podman, és per a una xarxa local casolana (homelab), no exposada a internet, per això és molt senzilla la configuració, el domini intern per veure lychee a casa serà: <https://lychee.cim.lan>

En aquest cas fem servir les imatges de <https://www.linuxserver.io/our-images>, moltes gràcies a tots ells :)

## Introducció

El desplegament l'hem fet en una Fedora 43, per això cal tenir present el tema de SeLinux i els serveis systemd per deixar-ho tot automatitzat, tot s'ha fet dins de `/home/joan`, el meu usuari.

## Instal·lació de les eines necessàries

- Com és una instal·lació neta, necessitem fer uns `dnf install` per tenir el que cal:

  ```bash
  sudo dnf install podman podman-compose openssl
  ```

## Activació de linger per a l’usuari

- Permet que els serveis systemd d’usuari funcionin sense sessió iniciada:

  ```bash
  loginctl enable-linger joan
  ```

## Activació de ports privilegiats per a Podman

- Permet que contenidors no root escoltin al port 80:

  ```bash
  echo "net.ipv4.ip_unprivileged_port_start=80" | sudo tee /etc/sysctl.d/99-podman-ports.conf
  sudo sysctl --system
  ```

## Creació de la xarxa dedicada dels contenidors

- L'hem d'executar només una vegada:

  ```bash
  podman network create lychee_net
  ```
  
## Configuració del tallafoc

- Obre HTTP i HTTPS al sistema:

  ```bash
  sudo firewall-cmd --permanent --add-service=http
  sudo firewall-cmd --permanent --add-service=https
  sudo firewall-cmd --reload
  ```

## Creació de l'estructura de directoris abans de començar l'instal·lació propiament dita

- Per Lychee + MariaDB

  ```bash
  mkdir -p /home/joan/lychee/db
  mkdir -p /home/joan/lychee/config
  mkdir -p /home/joan/lychee/pictures
  ```

- Per Nginx

  ```bash
  mkdir -p /home/joan/nginx/config/keys
  mkdir -p /home/joan/nginx/config/nginx/site-confs
  ```

## Generació de certificats autosignats

- Certificat per al domini intern lychee.cim.lan, aquesta comanda l'hem d'executar dins de `/home/joan/nginx/config/keys`:

  ```bash
  openssl req -x509 -nodes -newkey rsa:4096 \
  -keyout cert.key \
  -out cert.crt \
  -days 365 \
  -subj "/CN=lychee.cim.lan"
  ```

## Creació de docker-compose.yml per a Lychee + MariaDB

- Fem un `vim /home/joan/lychee/docker-compose.yml` i escrivim:

  ```text
  version: "3.8"
  
  services:
    mariadb:
      image: lscr.io/linuxserver/mariadb:latest
      container_name: lychee_db
      environment:
        - PUID=1000
        - PGID=1000
        - TZ=Europe/Madrid
        - MYSQL_ROOT_PASSWORD=Super_Password_de_ROOT
        - MYSQL_DATABASE=lychee
        - MYSQL_USER=lychee
        - MYSQL_PASSWORD=Password_de_la_DB
      volumes:
        - ./db:/config:Z
      restart: unless-stopped
      networks:
        - lychee_net

    lychee:
      image: lscr.io/linuxserver/lychee:latest
      container_name: lychee
      environment:
        - PUID=1000
        - PGID=1000
        - TZ=Europe/Madrid
  
        - DB_CONNECTION=mysql
        - DB_HOST=lychee_db
        - DB_PORT=3306
        - DB_USERNAME=lychee
        - DB_PASSWORD=Password_de_la_DB
        - DB_DATABASE=lychee

        - APP_URL=https://lychee.cim.lan
        - TRUSTED_PROXIES=*
      volumes:
        - ./config:/config:Z
        - ./pictures:/pictures:Z
      depends_on:
        - mariadb
      restart: unless-stopped
      networks:
        - lychee_net

  networks:
    lychee_net:
      external: true
  ```

## Creació de docker-compose.yml per a Nginx

- Fem un `vim /home/joan/nginx/docker-compose.yml` i escrivim:

  ```text
  version: "3.8"

  services:
    nginx:
      image: lscr.io/linuxserver/nginx:latest
      container_name: nginx
      environment:
        - PUID=1000
        - PGID=1000
        - TZ=Europe/Madrid
      volumes:
        - ./config:/config:Z
      ports:
        - 80:80
        - 443:443
      restart: unless-stopped
      networks:
        - lychee_net

  networks:
    lychee_net:
      external: true
  ```

## Configuració de Nginx

- Fitxer de configuració per al proxy invers cap a Lychee, fem un `vim /home/joan/nginx/config/nginx/site-confs/lychee.conf` i escrivim:

  ```text
  server {
      listen 80;
      server_name lychee.cim.lan;

      return 301 https://$host$request_uri;
  }

  server {
      listen 443 ssl;
      server_name lychee.cim.lan;

      ssl_certificate /config/keys/cert.crt;
      ssl_certificate_key /config/keys/cert.key;

      add_header Strict-Transport-Security "max-age=31536000" always;

      location / {
          proxy_pass http://lychee:80;
          proxy_set_header Host $host;
          proxy_set_header X-Real-IP $remote_addr;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_set_header X-Forwarded-Proto https;
      }
  }
  ```

## Comprovació que tot funciona

- Arrencar els contenidors des dels seus directoris correctes, primer ho fem amb Lychee:

  ```bash
  cd ~/lychee
  podman-compose up -d
  ```

- Ara li toca a Nginx:

  ```bash
  cd ~/nginx
  podman-compose up -d
  ```

- Aquí ja hauríem de poder veure la web a <https://lychee.cim.lan>

- Automatitzem tot, primer generem els serveis systemd per Lychee i MariaDB (des del directori correcte):

  ```bash
  cd ~/lychee
  podman generate systemd --new --files --name lychee
  podman generate systemd --new --files --name lychee_db
  ```

- Ara el mateix per al servei systemd per Nginx (des del seu directori):

  ```bash
  cd ~/nginx
  podman generate systemd --new --files --name nginx
  ```

- Instal·lem els serveis generats:

  ```bash
  mkdir -p ~/.config/systemd/user
  mv ~/lychee/container-lychee.service ~/.config/systemd/user/
  mv ~/lychee/container-lychee_db.service ~/.config/systemd/user/
  mv ~/nginx/container-nginx.service ~/.config/systemd/user/
  ```

- Recarreguem systemd:

  ```bash
  systemctl --user daemon-reload
  ```

- Activem i arrenquem els serveis:

  ```bash
  systemctl --user enable --now container-lychee_db.service
  systemctl --user enable --now container-lychee.service
  systemctl --user enable --now container-nginx.service
  ```

  > NOTA: Ara mateix tenim el Lychee funcionant, a cada reinici els serveis aixecaran els contenidors i tot funcionarà sense tocar res, sense login.

## Per si volem mirar logs si hi ha hagut alguna errada

- Aquests logs ens donen informació important en cas que alguna cosa no funcioni bé:

  ```bash
  podman logs -f lychee
  podman logs -f lychee_db
  podman logs -f nginx
  ```

- Per veure els contenidors aixecats, volums i les imatges tenim això:

  ```bash
  podman ps -a
  podman images
  podman volume ls
  ```

## Final
 
Quan entres a <https://lychee.cim.lan> per primer cop, Lychee et demana un usuari `admin` i un *password*, després de fer-ho, ja pots gaudir de `Lychee` i començar a pujar fotos i configurar-lo al teu gust.

## Si t'has *enmerdat*  molt i vols començar de zero sempre pots fer això

- Aquestes comandes ho reventen tot:

  ```bash
  sudo podman rm -af
  sudo podman rmi -af
  sudo podman volume rm -a
  sudo podman network rm -a
  podman system reset
  ```

  > COMPTE: vigila que això es una escombrada absoluta xD

Doncs això és tot per ara, falten coses però a poc a poc, a gaudir!

