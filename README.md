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

Dentro da carpeta chamada 'config', debemos crear dos ficherios:

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
    forwarders {
        8.8.8.8;                        # Google DNS
        1.1.1.1;                        # Cloudflare DNS
    };
    listen-on { any; };                 # Escuchar en todas las interfaces
    listen-on-v6 { any; };
};
```

## Arquivo de zonas 

### db.asircastelao

```
$TTL    604800
@       IN      SOA     ns1.mydomain.com. admin.asircastelao.com. (
                        2         ; Serial
                        604800    ; Refresh
                        86400     ; Retry
                        2419200   ; Expire
                        604800 )  ; Negative Cache TTL

; Servidor de nombres (NS)
@       IN      NS      ns1.asircastelao.com.

; Registros A
ns1     IN      A       172.20.0.10
web     IN      A       192.168.1.100

; Registro CNAME
www     IN      CNAME   web.asircastelao.com.

; Registro TXT
@       IN      TXT     "Este es un registro de texto para asircastelao.com"
```
