# 🛡️ Práctica Guiada Paso a Paso

## Proxy Directo (Squid) + Proxy Inverso (NGINX) con Autenticación y TLS

**Asignatura:** Seguridad y Alta Disponibilidad
**Ciclo:** Administración de Sistemas Informáticos en Red (2 ASIR)

---

> 📌 **Cómo usar este documento**
>
> Este Markdown está pensado para que **NO te pierdas en ningún momento**.
>
> En cada sección verás siempre:
>
> * 📍 **Dónde estás** (cliente, proxy, DMZ, red interna)
> * ❓ **Por qué haces este paso**
> * ⏭️ **Qué desbloquea para el siguiente paso**
> * ⌨️ **Código exacto para copiar, pegar y pulsar Enter**

No hay que improvisar nada.

---

## 1️⃣ DÓNDE ESTAMOS Y QUÉ VAMOS A MONTAR

Estamos simulando una **empresa real** donde:

* Un **cliente interno** NO puede acceder directamente a un servidor web.
* Todo el tráfico debe pasar por **controles de seguridad**.

📡 El camino obligatorio será:

```
CLIENTE → PROXY DIRECTO (Squid) → PROXY INVERSO (NGINX HTTPS) → APP INTERNA
```

Esto se conoce como **defensa en profundidad**.

---

## 2️⃣ PRIMER PASO – CREAR LA ESTRUCTURA DEL PROYECTO

📍 **Dónde estamos:** En el sistema anfitrión (tu Linux).

❓ **Por qué:** Docker necesita que los archivos existan ANTES de levantar contenedores.

### 📂 Ejecuta exactamente esto:

```bash
mkdir -p lab-proxies/{squid,reverse/certs,app}
cd lab-proxies
```

Ahora crea los archivos vacíos:

```bash
touch docker-compose.yml
nano squid/squid.conf
nano squid/passwd
nano reverse/nginx.conf
nano reverse/htpasswd
nano app/index.html
```

👉 **No rellenes nada aún**, eso viene en los siguientes pasos.

---

## 3️⃣ DOCKER COMPOSE – LA INFRAESTRUCTURA

📍 **Dónde estamos:** Definiendo TODA la red y servicios.

❓ **Por qué:** Docker Compose describe la arquitectura completa (redes, contenedores, aislamiento).

### 📄 docker-compose.yml

👉 Copia TODO tal cual:

```yaml
services:
  client:
    image: curlimages/curl:8.10.1
    container_name: client
    command: ["sh", "-c", "sleep infinity"]
    networks:
      - client_net

  forward-proxy:
    image: ubuntu/squid:5.2-22.04_beta
    container_name: forward-proxy
    volumes:
      - ./squid/squid.conf:/etc/squid/squid.conf:ro
      - ./squid/passwd:/etc/squid/passwd:ro
    networks:
      - client_net
      - dmz_net
    ports:
      - "3128:3128"

  reverse-proxy:
    image: nginx:1.27-alpine
    container_name: reverse-proxy
    volumes:
      - ./reverse/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./reverse/htpasswd:/etc/nginx/htpasswd:ro
      - ./reverse/certs:/etc/nginx/certs:ro
    networks:
      - dmz_net
      - internal_net
    ports:
      - "8443:443"

  app:
    image: nginx:1.27-alpine
    container_name: app
    volumes:
      - ./app/index.html:/usr/share/nginx/html/index.html:ro
    networks:
      - internal_net

networks:
  client_net:
    driver: bridge
  dmz_net:
    driver: bridge
  internal_net:
    driver: bridge
```

⏭️ **Qué desbloquea:** Ya tenemos la topología de red definida.

---

## 4️⃣ APP INTERNA – EL SERVIDOR FINAL

📍 **Dónde estamos:** Red interna (internal_net).

❓ **Por qué:** Necesitamos algo sencillo para comprobar si llegamos al final del camino.

### 📄 app/index.html

```html
<!doctype html>
<html lang="es">
<head>
  <meta charset="utf-8">
  <title>APP Interna</title>
</head>
<body>
  <h1>APP interna OK</h1>
  <p>Si ves esto, has pasado por Squid + NGINX.</p>
</body>
</html>
```

⏭️ **Qué desbloquea:** Ya hay un recurso protegido.

---

## 5️⃣ PROXY INVERSO – NGINX (HTTPS + AUTH)

📍 **Dónde estamos:** DMZ → red interna.

❓ **Por qué:** El proxy inverso es quien publica el servicio y termina TLS.

### 📄 reverse/nginx.conf

```nginx
events {}

http {
    upstream app_upstream {
        server app:80;
    }

    server {
        listen 443 ssl;
        server_name reverse-proxy;

        ssl_certificate     /etc/nginx/certs/server.crt;
        ssl_certificate_key /etc/nginx/certs/server.key;
        ssl_protocols       TLSv1.2 TLSv1.3;

        auth_basic "Acceso restringido";
        auth_basic_user_file /etc/nginx/htpasswd;

        location / {
            proxy_pass http://app_upstream;
            proxy_set_header Host $host;
            proxy_set_header X-Forwarded-For $remote_addr;
        }
    }
}
```

---

## 6️⃣ PROXY DIRECTO – SQUID

📍 **Dónde estamos:** Cliente → DMZ.

❓ **Por qué:** Controla QUIÉN puede salir y a dónde.

### 📄 squid/squid.conf

```conf
http_port 3128
visible_hostname forward-proxy

acl SSL_ports port 443
acl Safe_ports port 80 443
acl CONNECT method CONNECT

http_access deny !Safe_ports
http_access deny CONNECT !SSL_ports

auth_param basic program /usr/lib/squid/basic_ncsa_auth /etc/squid/passwd
auth_param basic realm Proxy-Directo
acl autenticado proxy_auth REQUIRED

http_access allow autenticado
http_access deny all

cache deny all
```

---

## 7️⃣ CREDENCIALES (COPIAR Y PEGAR)

📍 **Dónde estamos:** Host.

### Squid (alumno / asir)

```bash
openssl passwd -apr1 asir
```

👉 Copia el hash y pega en `squid/passwd`:

```
alumno:$apr1$HASH
```

### NGINX (web / asir)

```bash
openssl passwd -apr1 asir
```

Pega en `reverse/htpasswd`:

```
web:$apr1$HASH
```

---

## 8️⃣ CERTIFICADO TLS

📍 **Dónde estamos:** Proxy inverso.

```bash
openssl req -x509 -nodes -newkey rsa:2048 \
  -keyout reverse/certs/server.key \
  -out reverse/certs/server.crt \
  -days 365 \
  -subj "/CN=reverse-proxy"
```

---

## 9️⃣ ARRANCAR TODO

```bash
docker compose up -d
docker compose ps
```

---

## 🔟 PRUEBAS (EN ORDEN, SIN SALTAR)

```bash
docker compose exec client sh
```

### ❌ Sin proxy

```bash
curl http://app/
```

### ❌ Proxy sin auth

```bash
curl -k -x http://forward-proxy:3128 https://reverse-proxy/
```

### ❌ Proxy OK, web no

```bash
curl -k -x http://forward-proxy:3128 --proxy-user alumno:asir https://reverse-proxy/
```

### ✅ TODO OK

```bash
curl -k -x http://forward-proxy:3128 --proxy-user alumno:asir -u web:asir https://reverse-proxy/
```

---

## 🏁 CONCLUSIÓN

Has montado una **arquitectura real de empresa**, entendiendo:

* Segmentación
* Proxies
* TLS
* Autenticación en capas

👉 **Esto no es una demo, es seguridad real.**
