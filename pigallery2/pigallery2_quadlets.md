# Instal·lació i desplegament de PiGallery2, MariaDB i Nginx amb Podman Quadlets

Fins ara fèiem els desplegaments dels contenidors amb `podman-compose` i generàvem els serveis amb `podman generate systemd`. Aquest mètode crea serveis basats en un contenidor que ja està desplegat; per tant, qualsevol canvi al contenidor obliga a regenerar els serveis.

Amb Quadlets, en canvi, definim els serveis de manera declarativa. Els fitxers no canvien quan modifiquem el contenidor, són portables, mantenibles i molt més nets. Realment val la pena utilitzar-los.

Intentarem explicar com hem posat en marxa PiGallery2 amb MariaDB i Nginx fent servir Quadlets, és per veure-ho a la xarxa de casa, el domini per accedir a la galeria serà: <https://pigallery2.cim.lan>

Hem fet una barreja d'imatges oficials de <https://hub.docker.io> i de <https://linuxserver.io>. Moltes gràcies.

## Introducció

Tornem a ser dins d'una Fedora 43, cal vigilar com sempre el SELinux, els arxius .container de Quadlets estaran dins de `~/.config/containers/systemd` i la resta d'arxius que necessiten els contenidors, com sempre dins del meu `home`.

## Instal·lació de les eines necessàries

- Com és una instal·lació neta, necessitem fer uns `dnf install` per tenir el que cal:

  ```bash
  sudo dnf install podman openssl
  ```

  > NOTA: Ja no ens cal `podman-compose`

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

- Per a Nginx (dominis i certificats):

  ```bash
  mkdir -p ~/nginx/config/certs
  mkdir -p ~/nginx/config/conf.d
  ```

- Per a PiGallery2 (fotos i dades):

  ```bash
  mkdir -p ~/pigallery2/config ~/pigallery2/images ~/pigallery2/tmp ~/pigallery2/db
  ```

  > NOTA: El directori `db` només el fa servir si fem ús de `sqlite` i no pas de `MariaDB`.

- Per a MariaDB:

  ```bash
  mkdir -p ~/mariadb/config
  ```

## Generació de certificats autosignats

- Certificat per al domini intern pigallery2.cim.lan, aquesta comanda l'hem d'executar dins de `~/nginx/config/certs`:

  ```bash
  openssl req -x509 -nodes -days 10950 -newkey rsa:4096 \
  -keyout selfsigned.key \
  -out selfsigned.crt \
  -subj "/CN=cim.lan" \
  -addext "subjectAltName = DNS:cim.lan,DNS:*.cim.lan"
  ```

## Creació de la xarxa dedicada dels contenidors

- Els contenidors han de parlar-se entre ells (necessiten una xarxa), per això farem una xarxa amb quadlets, la configurem i li fem un start.

- Escrivim la xarxa fent un `vim ~/.config/containers/systemd/cim_lan.network`:

  ```text
  [Unit]
  Description=Xarxa per a cim.lan

  [Network]
  NetworkName=cim_lan-net
  Subnet=10.91.0.0/24
  Gateway=10.91.0.1
  ```

- La posem en marxa, primer sempre hem de fer un `systemctl --user daemon-reload` per a que llegeixi els canvis en els arxius de systemd, això crea el .service corresponent que és el que després haurem d'engegar amb:

  ```
  systemctl --user daemon-reload
  systemctl --user start cim_lan-network.service
  ```

  > COMPTE: encara que l'arxiu de la xarxa era `cim_lan.network`, el servei associat creat en fer el `systemctl --user daemon-reload` és `cim_lan-network.service`

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
  Environment=MYSQL_ROOT_PASSWORD=RootPassword
  Environment=MYSQL_DATABASE=pigallery2
  Environment=MYSQL_USER=pi_user
  Environment=MYSQL_PASSWORD=PiPassword
  Volume=%h/mariadb/config:/config:Z
  Network=cim_lan-net

  [Install]
  WantedBy=default.target
  ```

## Creació del Quadlet pigallery2.container

- Fem un `vim ~/.config/containers/systemd/pigallery2.container` i escrivim:

  ```text
  [Unit]
  Description=Servei PiGallery2
  # Amb això ens assegurem que la base de dades estigui activa abans d'aixecar pigallery2.service
  Requires=mariadb.service
  After=mariadb.service

  [Container]
  ContainerName=pigallery2
  AutoUpdate=registry
  Image=docker.io/bpatrik/pigallery2:latest
  Environment=NODE_ENV=production

  # Les variables de connexió a MariaDB no es posen aquí per a aquesta imatge de PiGallery2

  Volume=%h/pigallery2/config:/app/data/config:Z
  Volume=%h/pigallery2/images:/app/data/images:ro,Z
  Volume=%h/pigallery2/tmp:/app/data/tmp:Z
  Volume=%h/pigallery2/db:/app/data/db:Z

  Network=cim_lan-net

  [Service]
  Restart=always
  RestartSec=10

  [Install]
  WantedBy=default.target
  ```

## Creació del Quadlet nginx.container

- Fem un `vim ~/.config/containers/systemd/nginx.container` i escrivim:

  ```text
  [Unit]
  Description=Nginx Reverse Proxy Principal
  Wants=mariadb.service pigallery2.service
  After=mariadb.service pigallery2.service

  [Container]
  ContainerName=nginx
  AutoUpdate=registry
  Image=docker.io/nginx:latest
  Volume=%h/nginx/config/conf.d:/etc/nginx/conf.d:ro,Z
  Volume=%h/nginx/config/certs:/etc/nginx/certs:ro,Z
  Network=cim_lan-net
  PublishPort=80:80
  PublishPort=443:443

  [Install]
  WantedBy=default.target
  ```

## Configuració de Nginx

- Fitxer de configuració per al proxy invers cap a PiGallery2, fem un `vim ~/nginx/config/conf.d/pigallery2.conf` i escrivim:

  ```text
  # Redirecció HTTP → HTTPS
  server {
      listen 80;
      server_name pigallery2.cim.lan _;
      return 301 https://$host$request_uri;
  }

  # Servidor HTTPS
  server {
      listen 443 ssl;
      server_name pigallery2.cim.lan _;

      # DNS de la xarxa cim_lan-net (Gateway)
      resolver 10.91.0.1 valid=30s;

      ssl_certificate /etc/nginx/certs/selfsigned.crt;
      ssl_certificate_key /etc/nginx/certs/selfsigned.key;

      server_tokens off;

      # Headers de seguretat
      add_header X-Content-Type-Options nosniff always;
      add_header X-Frame-Options DENY always;
      add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
      add_header Referrer-Policy "strict-origin-when-cross-origin" always;
      add_header X-XSS-Protection "1; mode=block" always;

      # Backend real de PiGallery2 (port 80)
      set $upstream_pigallery pigallery2;

      location /pgapi {
          proxy_pass http://$upstream_pigallery:80;
          proxy_http_version 1.1;
          proxy_set_header Host $host;
          proxy_set_header Upgrade $http_upgrade;
          proxy_set_header Connection "upgrade";
          proxy_set_header X-Real-IP $remote_addr;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_set_header X-Forwarded-Proto $scheme;
          proxy_buffering off;
          proxy_redirect off;
      }

      location / {
          proxy_pass http://$upstream_pigallery:80;
          proxy_http_version 1.1;
          proxy_set_header Host $host;
          proxy_set_header Upgrade $http_upgrade;
          proxy_set_header Connection "upgrade";
          proxy_set_header X-Real-IP $remote_addr;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_set_header X-Forwarded-Proto $scheme;
          proxy_buffering off;
          proxy_redirect off;
      }
  }
  ```

## Comprovació que tot funciona

- Arrencar els contenidors fent primer un `systemctl --user daemon-reload`

  ```bash
  systemctl --user daemon-reload
  systemctl --user start mariadb pigallery2 nginx
  ```

- Aquí ja hauríem de poder veure la web a <https://pigallery2.cim.lan>

- Aquesta primera arrancada ens crearà un arxiu `~/pigallery2/config/config.json`, en aquests moments la web funciona però sense MariaDB, llavors:

- Aturem pigallery2.service:

  ```bash
  systemctl --user stop pigallery2
  ```

- Modifiquem l'arxiu `config.json` en la secció dedicada a les bases de dades `Database`, aquí es fan els canvis per fer servir MariaDB i també introduim els paràmetres per connectar amb MariaDB, ha de quedar així:

  ```text
    "Database": {
        "//[type]": "SQLite is recommended.",
        "type": "mysql",
        "//[dbFolder]": "All file-based data will be stored here (sqlite database, job history data).",
        "dbFolder": "/app/data/db",
        "sqlite": {
            "//[DBFileName]": "Sqlite will save the db with this filename.",
            "DBFileName": "sqlite.db"
        },
        "mysql": {
            "host": "mariadb",
            "port": 3306,
            "database": "pigallery2",
            "username": "pi_user",
            "password": "PiPassword"
        }
    },
  ```

- Una vegada modificat aquest arxiu tornem a aixecar PiGallery2:

  ```bash
  systemctl --user start pigallery2
  ```

- Ara ja tenim PiGallery2 amb MariaDB com a base de dades i Nginx fent de proxy, a gaudir.

## Backups:

- Containers, .json i .conf:

  ```
  cp ~/.config/containers/systemd/*.* ~/backups/
  cp ~/pigallery2/config/config.json ~/backups/
  cp ~/nginx/config/conf.d/*.conf ~/backups/
  ```

- Database:

  Backup `podman exec mariadb mariadb-dump -u pi_user -p'PiPassword' pigallery > ~/db_pigallery_$(date +%F).sql`

  Restore `podman exec -i mariadb mariadb -u pi_user -p'PiPassword' pigallery < ~/db_pigallery_YYYY-MM-DD.sql`

- Ja ho tenim gairebé tot, a gaudir!

## Important activar

- Activarem l'actualització automàtica de les imatges podman:

  ```
  systemctl --user enable --now podman-auto-update.timer
  ```

## Per saber que hi ha dins d'un contenidor i configurar correctament

- Crear un contenidor temporal, per exemple de nginx:

  ```bash
  podman create --name tmpnginx docker.io/nginx:latest
  ```

- Mirar dins del contenidor:

  ```bash
  podman start tmpnginx
  podman exec tmpnginx ls -als /etc/nginx <-- si volem saber que hi ha dins
  podman exec -it tmpnginx sh <-- obrir una shell i mirar dins
  ```

- Copiar un directori del contenidor:

  ```bash
  podman cp tmpnginx:/etc/nginx ./nginx-config-oficial
  ```

- Eliminar el contenidor temporal

  ```bash
  podman rm -f tmpnginx
  ```
