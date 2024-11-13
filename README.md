# P7-DNS_Linux

Para configurar todo, temos que primeiro crear un arquivo, onde crear a docker-compose.yml, os ficheiros de configuración e os arquivos de zonas. 

## Docker-compose.yml

Unha vez feito, podemos empezar a crear os ficheiros, e no me caso empecei co docker-compose.yml:

```
services:
  bind9:
    container_name: justoserver
    image: internetsystemsconsortium/bind9:9.18
    platform: linux/amd64
    ports:
      - 55:53/tcp
      - 55:53/udp
    networks:
      alex_subnet:
        ipv4_address: 172.30.8.1
    volumes:
      - ./conf:/etc/bind
      - ./zonas:/var/lib/bind
    restart: always
  cliente:
    container_name: cliente2
    image: alpine
    platform: linux/amd64
    tty: true
    stdin_open: true
    # configuramos para que el cliente use nuestro dns
    dns:
      - 172.30.8.1
    networks:
      alex_subnet:
        ipv4_address: 172.30.8.2
        
networks:
  alex_subnet:
    driver: bridge
    ipam:
      config:
        - subnet: 172.30.0.0/16
          ip_range: 172.30.8.0/24
          gateway: 172.30.8.254
```

Como observamos, no noso servidor chamado justoserver fixen o seguinte: empregamos una imaxe bind9.18; definin os portos tcp e udp; asignei unha red personalizada que eu definin,neste caso 172.30.8.2; e engadimos os volumes, que son os arquivos que máis adiante configuraremos. 

No cliente, definimoslle una imaxe alpine, ademáis coas opciones 'tty' e 'stdin_open' permitimos que teña unha terminal, e finalemnte definimos a red DNS coa IP do servidor e definimos a IP fixa do cliente, neste caso 172.30.8.2.

Finalemente, definin a red empregada, neste caso como tipo bridge, e fixen o seguinte: definin o rango de Ip en 172.30.8.0/26, coa ip_range limite as IP asignables a un rango específico, neste caso 172.30.8.0/24; e por ultimo definin a IP gateway en 172.30.8.254.

## Arquivos de configuración

Dentro da carpeta chamada 'config', debemos crear tres ficherios:

### named.conf.local

Neste ficheiro, engadimos o seguinte:

```
zone "asircastelao.com" {
    type master;
    file "/etc/bind/db.asircastelao.com";   # Archivo con los registros DNS de la zona
};
```


### named.conf.options

```
options {
    directory "/var/cache/bind";
    recursion yes;                      # Permitir la resolución recursiva
    allow-query { any; };               # Permitir consultas desde cualquier IP
    dnssec-validation no;
    forwarders {
        8.8.8.8;                        # Google DNS
        1.1.1.1;                        # Cloudflare DNS
    };
    listen-on { any; };                 # Escuchar en todas las interfaces
    listen-on-v6 { any; };
};
```

### named.conf

Neste arquivo, debemos enlazar os dous arquivos anteriores, como se ve a continuación.

```
include "/etc/bind/named.conf.options";
include "/etc/bind/named.conf.local";
```

## Arquivo de zonas 

### db.asircastelao

```
$TTL    604800
<<<<<<< HEAD
@       IN      SOA     ns.asircastelao.int. admin.asircastelao.int. (
                        2         ; Serial
                        604800    ; Refresh
                        86400     ; Retry
                        2419200   ; Expire
                        604800 )  ; Negative Cache TTL

@       IN      NS      ns.asircastelao.int.
ns       IN      A       172.30.8.1
test    IN      A       172.30.8.4
alias   IN      CNAME   web.asircastelao.com.
texto       IN      TXT     "Este es un registro de texto para asircastelao.com"
```


## Comprobación

Agora que xa fixemos os documentos, necesitamos arrincar o servidor, co comando 'docker compose up -d', o que nos permite iniciar o servidor pero tendo disponible a terminal. Agora necesitamos arrincar o cliente, que se fai co comando 'docker exec -it cliente2 /bin/sh', e xa entramos na terminal do cliente.

Nesta terminal, necesitamos descargar dig, onde primeiro empreguei o comando 'apk update', empregamos apk porque estamos nunha imaxe alpine, e finalmente o comando 'apk add --update bind-tools' para instalar o dig.

Finalemente, co dig disponible, empregamos o comando 'dig @172.30.8.1 ejemplo.asircastelao.inet' e observaremos o seguinte, o que significa que todo funcionou correctamente

```
; <<>> DiG 9.18.27 <<>> @172.30.8.1 ejemplo.asircastelao.inet
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: 59946
;; flags: qr rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: 444f62cfab6bd7aa010000006733ad5dd371b9b09805231e (good)
;; QUESTION SECTION:
;ejemplo.asircastelao.inet.	IN	A

;; AUTHORITY SECTION:
.			900	IN	SOA	a.root-servers.net. nstld.verisign-grs.com. 2024111201 1800 900 604800 86400

;; Query time: 31 msec
;; SERVER: 172.30.8.1#53(172.30.8.1) (UDP)
;; WHEN: Tue Nov 12 19:32:45 UTC 2024
;; MSG SIZE  rcvd: 157


```


