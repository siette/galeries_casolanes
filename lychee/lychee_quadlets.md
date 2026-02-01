# Instal·lació i desplegament de Lychee, MariaDB i Nginx amb Podman Quadlets

Igual que amb [PiGallery2 amb Quadlets](./pigallery2/pigallery2_quadlets.md), aquí farem servir el mateix sistema on aconseguim un *server* amb una galeria en qüestió de minuts.

Intentarem explicar com hem posat en marxa el Lychee amb MariaDB i Nginx fent servir Quadlets, com sempre per a la xarxa de casa, el domini per accedir a la galeria serà: <https://lychee.cim.lan>

Les *images* són les mateixes que en la instal·lació sense Quadlets, les de <https://linuxserver.io>. Moltes gràcies.

## Introducció

Un cop més tornem a ser dins d'una Fedora 43, cal vigilar com sempre el SELinux, els arxius .container de Quadlets estaran dins de `~/.config/containers/systemd` i la resta d'arxius que necessiten els contenidors, com sempre dins del meu `home`.

## Instal·lació de les eines necessàries

- Com és una instal·lació neta, necessitem fer uns `dnf install` per tenir el que cal:

  ```bash
  sudo dnf install podman openssl
  ```

## Activació de linger per a l’usuari

- Permet que els serveis systemd d’usuari funcionin sense sessió iniciada:

  ```bash
  loginctl enable-linger joan
  ```

## Activació de ports privilegiats per a Podman Quadlets

- Permet que contenidors no root escoltin al port 80 i d'altres necessaris com el 443:

  ```bash
  echo "net.ipv4.ip_unprivileged_port_start=80" | sudo tee /etc/sysctl.d/99-podman-ports.conf
  sudo sysctl --system
  ```

## Configuració del tallafoc

- Obre HTTP i HTTPS al sistema:

  ```bash
  sudo firewall-cmd --permanent --add-service=http
  sudo firewall-cmd --permanent --add-service=https
  sudo firewall-cmd --reload
  ```

## Creació de l'estructura de directoris abans d'aixecar els contenidors

- Per posar tots els Quadlets:

  ```bash
  mkdir -p ~/.config/containers/systemd
  ```

- Per a Nginx (configuració i certificats):

  ```bash
  mkdir -p ~/nginx/config/keys
  mkdir -p ~/nginx/config/nginx/site-confs
  ```

  > NOTA: segons l'imatge de Nginx potser hem de fer uns directoris o uns altres, compte amb això.

- Per a Lychee (configuració i les imatges):

  ```bash
  mkdir -p ~/lychee/config ~/lychee/pictures
  ```

- Per a MariaDB:

  ```bash
  mkdir -p ~/mariadb
  ```

## Generació de certificats autosignats

- Certificat per al domini intern pigallery2.cim.lan, aquesta comanda l'hem d'executar dins de `~/nginx/config/keys`:

  ```bash
  openssl req -x509 -nodes -newkey rsa:4096 \
  -keyout cert.key \
  -out cert.crt \
  -days 365 \
  -subj "/CN=lychee.cim.lan"
  ```

## Creació de la xarxa dedicada dels contenidors

- Els contenidors han de parlar-se entre ells (necessiten una xarxa), per això farem una xarxa amb Quadlets, la configurem i li fem un *start*.

- Escrivim la xarxa fent un `vim ~/.config/containers/systemd/cim_net.network`:

  ```text
  [Unit]
  Description=Xarxa systemd-cim_net

  [Network]
  NetworkName=systemd-cim_net
  Subnet=10.90.0.0/24
  Gateway=10.90.0.1
  Driver=bridge
  ```

- La posem en marxa, primer sempre hem de fer un `systemctl --user daemon-reload` perquè llegeixi els canvis en els arxius de systemd, això crea el .service corresponent que és el que després haurem d'engegar amb:

  ```
  systemctl --user start cim_net-network.service
  ```

  > COMPTE: encara que l'arxiu de la xarxa era `cim_net.network`, el servei associat creat en fer el `systemctl --user daemon-reload` és `cim_net-network.service`

## Creació del Quadlet mariadb.container

- Fem un `vim ~/.config/containers/systemd/mariadb.container` i escrivim:

  ```text
  [Unit]
  Description=MariaDB Database Server

  [Container]
  ContainerName=mariadb
  AutoUpdate=registry
  Image=lscr.io/linuxserver/mariadb:latest
  Environment=PUID=1000
  Environment=PGID=1000
  Environment=TZ=Europe/Madrid
  Environment=MYSQL_ROOT_PASSWORD=ROOT_PASSWORD
  Environment=MYSQL_DATABASE=lychee
  Environment=MYSQL_USER=lychee
  Environment=MYSQL_PASSWORD=LycheeDB_Password
  Volume=%h/mariadb:/config:Z
  Network=systemd-cim_net

  [Install]
  WantedBy=default.target
  ```

## Creació del Quadlet lychee.container

- Fem un `vim ~/.config/containers/systemd/lychee.container` i escrivim:

  ```text
  [Unit]
  Description=Servei Lychee
  Requires=mariadb.service
  After=mariadb.service

  [Container]
  ContainerName=lychee
  AutoUpdate=registry
  Image=lscr.io/linuxserver/lychee:latest
  Environment=PUID=1000
  Environment=PGID=1000
  Environment=TZ=Europe/Madrid
  Environment=DB_CONNECTION=mysql
  Environment=DB_HOST=mariadb
  Environment=DB_PORT=3306
  Environment=DB_USERNAME=lychee
  Environment=DB_PASSWORD=LycheeDB_Password
  Environment=DB_DATABASE=lychee
  Environment=APP_URL=https://lychee.cim.lan
  Environment=TRUSTED_PROXIES=*
  Volume=%h/lychee/config:/config:Z
  Volume=%h/lychee/pictures:/pictures:Z
  Network=systemd-cim_net

  [Install]
  WantedBy=default.target
  ```

## Creació del Quadlet nginx.container (Nginx)

- Fem un `vim ~/.config/containers/systemd/nginx.container` i escrivim:

  ```text
  [Unit]
  Description=Nginx Reverse Proxy
  Wants=mariadb.service lychee.service 
  After=mariadb.service lychee.service

  [Container]
  ContainerName=nginx
  AutoUpdate=registry
  Image=lscr.io/linuxserver/nginx:latest
  Environment=PUID=1000
  Environment=PGID=1000
  Environment=TZ=Europe/Madrid
  Volume=/home/joan/nginx/config:/config:Z
  Network=systemd-cim_net
  PublishPort=80:80
  PublishPort=443:443

  [Install]
  WantedBy=default.target
  ```

## Configuració de Nginx

Hem de posar la configuració en un dels directoris que ha creat:

- Fitxer de configuració per al proxy invers cap a Lychee, fem un `vim ~/nginx/config/nginx/site-confs/lychee.conf` i escrivim:

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

      resolver 10.90.0.1 valid=30s;

      location / {
          # Definim el nom en una variable
          set $upstream_lychee lychee;
          proxy_pass http://$upstream_lychee:80;

          proxy_set_header Host $host;
          proxy_set_header X-Real-IP $remote_addr;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_set_header X-Forwarded-Proto https;
      }
  }
  ```

## Comprovació que tot funciona

- Arrencar els contenidors fent primer un `systemctl --user daemon-reload`

  ```bash
  systemctl --user daemon-reload
  systemctl --user start mariadb lychee nginx
  ```

- Aquí ja hauríem de poder veure la web a <https://lychee.cim.lan>

- Ara ja tenim Lychee amb MariaDB com a base de dades i Nginx fent de proxy, a gaudir-ne.

