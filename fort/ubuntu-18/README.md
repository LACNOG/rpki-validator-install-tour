# FORT Validator en Ubuntu 18 LTS

## Instalación manual

Todo esto fue probado en Ubuntu 18.04 LTS Server recién instalado (solo agregamos el paquete de _SSH_ durante la instalación) y los comandos se ejecutan como el usuario _root_ (en caso contrario usar `sudo` donde corresponda).

Se optó por este método de instalación debido a que se tiene control de cada aspecto del sistema, en vez de depender de paquetes adaptados para cada tipo de sistema operativo.

En el caso de FORT nos encontramos con que los paquetes _DEB_ están armados para instalarse en Debian Buster y alguna que otra vez vimos alguna incompatibilidad mínima al usarlos en entornos Ubuntu.

A futuro vamos a cubrir instalaciones más automatizadas mediante el uso de paquetes y playbooks.

## Requisitos previos

El autor de este tutorial recomienda disponer de al menos 1 gigabyte de memoria RAM en la VM en donde instalemos este sistema.

## Instalación

```bash
# Nos volvemos root
sudo su

# Creamos el usuario que va a ejecutar todo
useradd --system rpki --shell /sbin/nologin

# Creamos los directorios necesarios
mkdir -p /opt/fort/{etc,tal,slurm,repo,csv}

# Instalamos el software base
apt update && apt install -y ntp rsync wget curl autoconf automake build-essential libjansson-dev libssl-dev pkg-config rsync libcurl4-openssl-dev libxml2-dev

systemctl enable ntp
systemctl start ntp

cd /usr/src

# Al momento de escribir este texto la última versión disponible era 1.4.2
# Por favor chequear si existe una nueva y reemplazar el siguiente comando de descarga 
wget https://github.com/NICMx/FORT-validator/releases/download/v1.4.2/fort-1.4.2.tar.gz

tar xvzf fort-1.4.2.tar.gz
cd fort-1.4.2/

# Compilamos
./configure
make
make install

# Instalamos los TAL (Aceptando explícitamente el ARIN RPA)
cp examples/tal/*.tal /opt/fort/tal/
wget -O /opt/fort/tal/arin.tal https://www.arin.net/resources/manage/rpki/arin.tal

# Creamos archivo de configuracion
cat << 'EOF' > /opt/fort/etc/config.json
{
  "tal": "/opt/fort/tal",
  "local-repository": "/opt/fort/repo",
  "work-offline": false,
  "shuffle-uris": false,
  "maximum-certificate-depth": 32,
  "mode": "server",
  "server": {
    "port": "8323",
    "backlog": 64,
    "interval": {
      "validation": 3600,
      "refresh": 3600,
      "retry": 600,
      "expire": 7200
    }
  },
  "slurm": "/opt/fort/slurm",
  "log": {
    "enabled": true,
    "level": "warning",
    "output": "syslog",
    "color-output": false,
    "file-name-format": "global-url",
    "facility": "daemon"
  },
  "validation-log": {
    "enabled": false,
    "level": "warning",
    "output": "console",
    "color-output": false,
    "file-name-format": "global-url",
    "facility": "daemon",
    "tag": "Validation"
  },
  "http": {
    "enabled": true,
    "priority": 60,
    "retry": {
      "count": 2,
      "interval": 5
    },
    "user-agent": "fort/1.4.2",
    "connect-timeout": 30,
    "transfer-timeout": 0,
    "idle-timeout": 15,
    "ca-path": "/usr/lib/ssl/certs"
  },
  "rsync": {
    "enabled": true,
    "priority": 50,
    "strategy": "root-except-ta",
    "retry": {
      "count": 2,
      "interval": 5
    },
    "program": "rsync",
    "arguments-recursive": [
      "--recursive",
      "--delete",
      "--times",
      "--contimeout=20",
      "--timeout=15",
      "$REMOTE",
      "$LOCAL"
    ],
    "arguments-flat": [
      "--times",
      "--contimeout=20",
      "--timeout=15",
      "--dirs",
      "$REMOTE",
      "$LOCAL"
    ]
  },
  "incidences": [
    {
      "name": "incid-hashalg-has-params",
      "action": "ignore"
    },
    {
      "name": "incid-obj-not-der-encoded",
      "action": "ignore"
    },
    {
      "name": "incid-file-at-mft-not-found",
      "action": "error"
    },
    {
      "name": "incid-file-at-mft-hash-not-match",
      "action": "error"
    },
    {
      "name": "incid-mft-stale",
      "action": "error"
    },
    {
      "name": "incid-crl-stale",
      "action": "error"
    }
  ],
  "output": {
    "roa": "/opt/fort/csv/roas.csv",
    "bgpsec": "/opt/fort/csv/bgpsec.csv"
  },
  "asn1-decode-max-stack": 4096,
  "stale-repository-period": 43200
}
EOF

# Establecemos permisos
chown -R rpki:rpki /opt/fort

# Creamos archivo de servicio
cat << 'EOF' > /etc/systemd/system/fort.service
[Unit]
Description=FORT RPKI Validator
Documentation=https://nicmx.github.io/FORT-validator/
After=network-online.target
AssertFileIsExecutable=/usr/local/bin/fort

[Service]
WorkingDirectory=/opt/fort/

User=rpki
Group=rpki

ExecStart=/usr/local/bin/fort -f /opt/fort/etc/config.json

Restart=always
RestartPreventExitStatus=7
RestartForceExitStatus=3

LimitNOFILE=65536

TimeoutStopSec=infinity
SendSIGKILL=no

[Install]
WantedBy=multi-user.target
EOF

# Configuramos el servicio
systemctl daemon-reload
systemctl enable fort.service
systemctl start fort.service

# Chequeamos si el servicio está en ejecución
systemctl status fort.service

# Chequear que el puerto 8323 esté abierto (PUEDE TARDAR VARIOS MINUTOS HASTA APARECER)
netstat -tple | grep 8323

```

Una vez que FORT descarque todos los ROA de los 5 RIRs podremos establecer una sesión _RTR_ contra la IP de nuestro validador en el puerto 8323

# Fin

### Autor
Ariel S. Weher <ariel [at] weher [dot] net>