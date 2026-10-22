# sentrylink-instalar

Arranque del instalador de **SentryLink Local v2**, en una línea, y todos los
comandos para instalar, operar y mantener una instalación.

Leé [`instalar`](instalar) antes de correrlo: es corto.

- [1. Instalar](#1-instalar)
- [2. Opciones del comando de una línea](#2-opciones-del-comando-de-una-línea)
- [3. Instalador en Debian / Ubuntu: `instalar-servidor.sh`](#3-instalador-en-debian--ubuntu-instalar-servidorsh)
- [4. Instalador en Proxmox: `instalar-proxmox.sh`](#4-instalador-en-proxmox-instalar-proxmoxsh)
- [5. Configuración de la obra: `instalar.sh`](#5-configuración-de-la-obra-instalarsh)
- [6. El stack: `init.sh` y Docker](#6-el-stack-initsh-y-docker)
- [7. El puente, servicio-comandos y el simulador](#7-el-puente-servicio-comandos-y-el-simulador)
- [8. Usuarios y tablero](#8-usuarios-y-tablero)
- [9. Respaldos](#9-respaldos)
- [10. Diagnóstico](#10-diagnóstico)
- [Reglas que no se negocian](#reglas-que-no-se-negocian)

Todos los comandos se corren **como root**. Los que dicen `cd /opt/genmonitor`
van en el servidor de la obra (en Proxmox, adentro del CT: `pct enter <CTID>`).

---

## 1. Instalar

### Requisitos

- **Debian 13** o **Ubuntu 26.04 LTS** (o un host **Proxmox**). Otras versiones
  frenan antes de instalar nada: el portal necesita Node 20.19 o más, y el
  instalador solo usa el Node de la distribución.
- Salida a internet (GitHub, Docker, los repositorios de la distribución).
- Un **token de GitHub** *fine-grained*, de los dos repos
  `walc72/sentrylink-local-v2` y `walc72/puente-modbus-mqtt`, con:
  - **Contents:** Read-only (para bajar el código);
  - **Administration:** Read and write (para cargar las deploy keys);
  - vencimiento de **1 día** (Administration también deja borrar el repo).

  Se crea en *GitHub → Settings → Developer settings → Personal access tokens →
  Fine-grained tokens*. **No es una deploy key:** las deploy keys las crea y
  las carga el instalador solo.

### Abrir una sesión que sobreviva a un corte

La instalación tarda (10 a 30 minutos según el disco). Si se corta el SSH a
mitad, el instalador muere sin aviso. Corré todo adentro de `tmux`:

```bash
apt install -y tmux
```
Instala tmux (si no está).

```bash
tmux new -s instalador
```
Abre una sesión con nombre. Si se corta la conexión, la instalación sigue.

```bash
tmux attach -t instalador
```
Vuelve a la sesión después de reconectar.

### El comando

```bash
wget -qO- https://raw.githubusercontent.com/walc72/sentrylink-instalar/main/instalar | sudo bash
```
Con un usuario que tiene `sudo` (Ubuntu). Pide el token, baja el último
release (tag `nubicom-AAAA.MM.N`) a `/root/sentrylink-instalador` y lanza el
instalador que corresponde: `instalar-proxmox.sh` en un host Proxmox,
`instalar-servidor.sh` en un Debian o Ubuntu.

```bash
wget -qO- https://raw.githubusercontent.com/walc72/sentrylink-instalar/main/instalar | bash
```
Lo mismo, si ya sos root (un Debian mínimo no trae `sudo`).

Se usa `wget` y no `curl` porque un Debian mínimo no trae `curl`.

---

## 2. Opciones del comando de una línea

Las opciones van después de `bash -s --`. Si ya sos root, sin el `sudo`.

```bash
wget -qO- https://raw.githubusercontent.com/walc72/sentrylink-instalar/main/instalar | sudo bash -s -- --respuestas /root/sentrylink-servidor.respuestas
```
Repite la instalación **sin las preguntas de la obra**, con las respuestas de
la corrida anterior (el token y la actualización del sistema se siguen
preguntando). Sirve para retomar después de un corte o de un reinicio: lo que
quedó completo se verifica y se saltea, lo que quedó a medias se rehace. Todo
lo que no sea `--ref` pasa tal cual al instalador.

```bash
wget -qO- https://raw.githubusercontent.com/walc72/sentrylink-instalar/main/instalar | sudo bash -s -- --ref <TAG_O_RAMA>
```
Instala otro release o una rama (para probar algo que todavía no tiene tag).
**Nunca una rama en una obra:** solo tags.

---

## 3. Instalador en Debian / Ubuntu: `instalar-servidor.sh`

Instala todo en la máquina misma, sin CT: Docker, deploy keys, SentryLink en
`/opt/genmonitor`, el puente con su cinturón y el simulador, y la verificación.
Lo lanza el comando de una línea; también se puede correr directo:

```bash
bash /root/sentrylink-instalador/instalar-servidor.sh
```
Pregunta todo (la obra) e instala.

```bash
bash /root/sentrylink-instalador/instalar-servidor.sh --respuestas /root/sentrylink-servidor.respuestas
```
Sin las preguntas de la obra, con las respuestas guardadas. Retoma donde quedó.

`--token-fd N` es solo para el arranque: le pasa el token por un descriptor
para no pedirlo dos veces. A mano no se usa: sin esa opción, el instalador
pregunta el token si hace falta.

Lo que **nunca** hace, y frena antes de tocar nada:
- en una máquina que ya tiene SentryLink instalado de otra forma (a mano, o
  adentro de un CT de `instalar-proxmox.sh`);
- en una máquina cuyo puente ya lee un PLC real;
- pasar a otro release una instalación que ya terminó (eso es actualizar,
  otro paso).

```bash
bash /root/sentrylink-instalador/instalar-servidor.sh --help
```
Muestra las opciones.

Lo que hace por su cuenta, y cuándo frena:
- **Versión del sistema:** si apt no trae Node ≥ 20.19, frena y dice cómo
  subir de versión. No la sube solo: se hace a mano, con la máquina a la vista.
- **Actualización del sistema:** obligatoria; pide escribir `si` antes del
  `apt full-upgrade`.
- **Reinicio pendiente:** frena y dice que reinicies y repitas con
  `--respuestas`.

Para subir Ubuntu a una versión soportada (a mano):

```bash
sudo apt update && sudo apt full-upgrade -y && sudo reboot
```
Deja la versión actual al día y reinicia.

```bash
sudo do-release-upgrade
```
Sube a la versión siguiente de Ubuntu (repetir hasta 26.04 LTS).

**Esta versión todavía no tiene** el paso al PLC real, Caddy con dominio ni
`--actualizar`: queda siempre contra el simulador. Eso hoy está solo en
`instalar-proxmox.sh`.

---

## 4. Instalador en Proxmox: `instalar-proxmox.sh`

Corre en el **host** Proxmox y crea un CT por obra. Lanzado por el comando de
una línea, pide el token **de nuevo** para las deploy keys (todavía no lo
recibe del arranque).

```bash
bash /root/sentrylink-instalador/instalar-proxmox.sh
```
Pregunta el CT (ID, IP, disco, memoria, carpeta de respaldos) y la obra; crea
el CT e instala todo adentro, contra el simulador.

```bash
bash /root/sentrylink-instalador/instalar-proxmox.sh --respuestas /root/sentrylink-<CTID>.respuestas
```
Sin preguntas, con las respuestas de una corrida anterior. Retoma.

```bash
bash /root/sentrylink-instalador/instalar-proxmox.sh --simular --respuestas /root/sentrylink-<CTID>.respuestas
```
Muestra lo que haría, sin crear nada.

```bash
bash /root/sentrylink-instalador/instalar-proxmox.sh --destruir <CTID>
```
Borra un CT que creó este instalador (pide escribir el número). Nunca toca
los CT de producción protegidos.

```bash
bash /root/sentrylink-instalador/instalar-proxmox.sh --preparar-proxy <CTID_DEL_CADDY>
```
Una sola vez por proxy: agrega al Caddyfile compartido el import de los sitios
de las obras. Pide "sí".

```bash
bash /root/sentrylink-instalador/instalar-proxmox.sh --plc-real <CTID>
```
Pasa el CT del simulador al PLC real de la obra, con sus barreras: que ningún
otro servidor esté conectado al PLC, la IP escrita de nuevo, y una lectura
Modbus (solo lectura) que confirma el selector en LOCAL y MANUAL remoto en 0.
Pide "sí". **Nunca escribe en el PLC.**

```bash
bash /root/sentrylink-instalador/instalar-proxmox.sh --volver-simulador <CTID>
```
Lo inverso: pone primero el cinturón y vuelve al simulador. Pide "sí".

```bash
bash /root/sentrylink-instalador/instalar-proxmox.sh --actualizar <CTID>
```
Lleva el CT al último release (respaldo antes si hay migraciones). Retomable.
Pide "sí". Para ver los releases nuevos pide un token de GitHub de **solo
lectura** (Contents: read de `sentrylink-local-v2`); con ENTER usa los tags
que ya tiene el clon del host. No hace falta traerlos a mano: el clon es
privado y un `git fetch` sin token pide usuario y contraseña.

```bash
bash /root/sentrylink-instalador/instalar-proxmox.sh --actualizar <CTID> <TAG>
```
Lo mismo, a un release puntual.

---

## 5. Configuración de la obra: `instalar.sh`

Lo llaman los dos instaladores. Escribe el usuario MQTT de servicio-comandos,
la ACL, el tablero, el cliente y los generadores, y llama a `init.sh`. Las
respuestas quedan en `/opt/genmonitor/instalar.respuestas` (sin secretos).

```bash
cd /opt/genmonitor && ./instalar.sh
```
Pregunta todo y arranca.

```bash
cd /opt/genmonitor && ./instalar.sh --respuestas instalar.respuestas
```
Sin preguntas. Con `PLC_DESTINO=simulador` en las respuestas no hace falta
nada más.

```bash
cd /opt/genmonitor && ./instalar.sh --respuestas instalar.respuestas --unico-servidor
```
Solo para pruebas y para la CI: acepta respuestas con `PLC_DESTINO=plc`
confirmando que ningún otro servidor lee ese PLC. **No lleva el servidor al
PLC real:** solo escribe la configuración de SentryLink; el puente sigue en el
simulador y con el cinturón. Al PLC real se pasa con
`instalar-proxmox.sh --plc-real` (en Debian/Ubuntu todavía no existe), que
tiene las barreras.

```bash
cd /opt/genmonitor && ./instalar.sh --simular
```
Muestra lo que haría, sin escribir nada.

```bash
cd /opt/genmonitor && ./instalar.sh --respuestas instalar.respuestas --validar
```
Solo valida las respuestas (no mira Docker).

```bash
cd /opt/genmonitor && ./instalar.sh --solo-configurar
```
Escribe la configuración sin Docker ni `init.sh` (pruebas).

`--recrear-portal`, con cualquiera de arriba: si el alta del cliente y los
generadores cambió algo, recrea el authservice sin preguntar (corta el portal
unos segundos).

---

## 6. El stack: `init.sh` y Docker

El stack (portal, authservice, Postgres, Mosquitto, Mailpit, respaldos) corre
con Docker Compose en `/opt/genmonitor`. **Siempre con `--env-file .env.local`:**
sin él, las contraseñas quedan vacías.

```bash
cd /opt/genmonitor && ./init.sh --local
```
Genera los secretos que falten, compila el portal y el authservice si
cambiaron, y levanta todo. Es lo que corre `instalar.sh`.

```bash
cd /opt/genmonitor && ./init.sh --local --forzar-build
```
Igual, pero compila aunque nada haya cambiado.

```bash
cd /opt/genmonitor && ./init.sh --local --build
```
Solo vuelve a compilar el portal. (`init.sh` siempre lleva `--local`: sin él
se niega a correr, a propósito.)

```bash
cd /opt/genmonitor && ./init.sh --local --reset
```
⚠ **Borra todos los datos** (base, usuarios, historial) y empieza de cero.
Solo en un servidor de prueba.

```bash
cd /opt/genmonitor && docker compose -f docker-compose.local.yml --env-file .env.local ps
```
Estado de los contenedores.

```bash
docker ps --format '{{.Names}} {{.Status}}'
```
Lo mismo, corto: todos tienen que decir `Up` y los que tienen chequeo,
`(healthy)`.

```bash
cd /opt/genmonitor && docker compose -f docker-compose.local.yml --env-file .env.local logs -f
```
Los logs de todo el stack, en vivo (Ctrl+C para salir).

```bash
docker logs --tail 100 -f gmlocal_authservice
```
Los logs de un contenedor: `gmlocal_authservice`, `gmlocal_nginx`,
`gmlocal_postgres`, `gmlocal_mosquitto`, `gmlocal_mailpit`,
`gmlocal_respaldos`.

```bash
cd /opt/genmonitor && docker compose -f docker-compose.local.yml --env-file .env.local up -d
```
Levanta lo que esté parado, sin tocar lo que anda.

```bash
cd /opt/genmonitor && docker compose -f docker-compose.local.yml --env-file .env.local up -d --force-recreate --no-deps authservice
```
Recrea un servicio (después de cambiar `.env.local`, por ejemplo). Corta el
portal unos segundos.

```bash
cd /opt/genmonitor && docker compose -f docker-compose.local.yml --env-file .env.local down
```
Para todo el stack (los datos quedan). Ver las reglas antes de hacerlo con el
mando en uso.

```bash
cd /opt/genmonitor && bash scripts/cert-mailpit.sh
```
Genera el certificado TLS de Mailpit (el correo de prueba). Si lo generó de
nuevo, hay que recrear Mailpit y el authservice para que lo tomen (corta el
portal unos segundos):

```bash
cd /opt/genmonitor && docker compose -f docker-compose.local.yml --env-file .env.local up -d --force-recreate --no-deps mailpit authservice
```

---

## 7. El puente, servicio-comandos y el simulador

Son servicios de systemd, fuera de Docker:

| Unidad | Qué es |
|---|---|
| `puente-modbus-mqtt` | Lee el PLC por Modbus y publica en MQTT |
| `servicio-comandos` | Lleva las órdenes del portal al PLC y escribe el latido del enlace |
| `simulador-plc` | Un PLC falso en `127.0.0.1:5020`, para probar sin tablero |
| `sentrylink-cinturon` | Regla nft que bloquea toda salida al PLC de la obra mientras se usa el simulador |

```bash
systemctl status puente-modbus-mqtt servicio-comandos simulador-plc sentrylink-cinturon
```
Estado de las cuatro.

```bash
systemctl is-active puente-modbus-mqtt servicio-comandos simulador-plc sentrylink-cinturon
```
Lo mismo, una palabra por unidad. **Contra el simulador**, las cuatro dicen
`active`. **En el PLC real** (después de `--plc-real`) solo el puente y
servicio-comandos: el simulador queda apagado y el cinturón, sacado.

```bash
journalctl -u puente-modbus-mqtt -f
```
Log del puente en vivo. Al arrancar tiene que decir `Modbus conectado a …`.

```bash
journalctl -u servicio-comandos -f
```
Log de servicio-comandos en vivo.

```bash
journalctl -u simulador-plc -n 20
```
Últimas líneas del simulador (cuenta ciclos y latido).

```bash
journalctl -u puente-modbus-mqtt --since "1 hour ago"
```
Log de la última hora.

```bash
systemctl restart puente-modbus-mqtt
```
Reinicia el puente (por ejemplo, después de cambiar su yaml).

```bash
nft list table inet sentrylink_cinturon
```
Muestra el cinturón: la IP del PLC que está bloqueada. En el PLC real da
error, porque el cinturón no está (es lo esperado).

**Nunca `systemctl restart sentrylink-cinturon`:** el puente y
servicio-comandos dependen de él y se reiniciarían con él.

Archivos de configuración:
- `/opt/puente-modbus-mqtt/config/puente-tablero.yaml`: el puente.
- `/etc/puente-modbus-mqtt/comandos.yaml`: servicio-comandos.
- `/etc/sentrylink/cinturon.nft`: el cinturón.

---

## 8. Usuarios y tablero

```bash
cd /opt/genmonitor && ./restablecer-superadmin.sh
```
Pone una contraseña nueva al superadmin `admin` (se pide dos veces por
teclado; no queda en el historial). Para cuando se perdió la que mostró la
instalación.

```bash
cd /opt/genmonitor && ./restablecer-superadmin.sh --usuario <USUARIO>
```
Lo mismo, para otro usuario.

```bash
cd /opt/genmonitor && ./sembrar-tablero.sh --respuestas instalar.respuestas
```
Crea el cliente y los generadores G-1 y G-2 del tablero, si faltan. Nunca
cambia uno que ya existe. Si creó algo, pregunta antes de recrear el portal.

```bash
cd /opt/genmonitor && ./sembrar-tablero.sh --respuestas instalar.respuestas --recrear-portal
```
Lo mismo, recreando el portal sin preguntar.

```bash
docker exec gmlocal_postgres psql -U genmonitor -d genmonitor -c "SELECT sistema, habilitado, actualizado_en FROM mando_tablero;"
```
Muestra si el mando remoto está habilitado en cada tablero.

---

## 9. Respaldos

El contenedor `gmlocal_respaldos` hace los respaldos programados; desde el
portal (Admin → Respaldos) se pide uno en el momento y se descarga. Quedan en
`/opt/genmonitor/respaldos/datos/`.

```bash
ls -lt /opt/genmonitor/respaldos/datos | head
```
Los respaldos más nuevos.

```bash
docker logs --tail 50 gmlocal_respaldos
```
Qué hizo el servicio de respaldos.

```bash
cd /opt/genmonitor && docker exec -i gmlocal_postgres pg_restore --list < respaldos/datos/<ARCHIVO>.dump | head -5
```
Comprueba que un respaldo se puede leer (no restaura nada).

```bash
docker exec gmlocal_postgres sh -c 'psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -tAc "SELECT extversion FROM pg_extension WHERE extname = '\''timescaledb'\''"'
```
Versión de TimescaleDB de la base.

**Copia en un NAS (NFS o SMB), en Debian/Ubuntu.** Se monta en el servidor y
se declara en `.env.local`.

- **En Proxmox, esto no:** el instalador ya monta la carpeta del host en el
  CT (`CARPETA_RESPALDOS` en sus respuestas) y escribe `RESPALDO_DESTINO`.
  Montar un NAS adentro del CT falla (CT sin privilegios) o pisa lo que dejó
  el instalador.
- **La línea de `/etc/fstab`** lleva opciones que importan: `soft` (si no, la
  copia se cuelga cuando el NAS no contesta), `x-systemd.before=docker.service`
  y permiso de escritura para el uid 70 del servicio. Copiala de
  `docs/operacion/respaldos.md`, sección 2.1, del repo `sentrylink-local-v2`.

```bash
mkdir -p /mnt/respaldos-nas && mount /mnt/respaldos-nas && findmnt /mnt/respaldos-nas
```
Monta la carpeta del NAS (con su línea en `/etc/fstab`) y comprueba que quedó
montada.

```bash
findmnt /mnt/respaldos-nas >/dev/null && touch /mnt/respaldos-nas/.sentrylink-destino && ls -la /mnt/respaldos-nas/.sentrylink-destino
```
Crea el testigo que le dice al servicio que esa carpeta es la de verdad (sin
él, la copia queda en "carpeta no montada").

```bash
cd /opt/genmonitor && docker compose -f docker-compose.local.yml --env-file .env.local up -d --force-recreate respaldos
```
Recrea el servicio de respaldos después de poner `RESPALDO_DESTINO=/mnt/respaldos-nas`
en `.env.local`.

```bash
ls -lt /mnt/respaldos-nas | head -3
```
Las últimas copias en el NAS.

**Restaurar** un respaldo es un procedimiento de varios pasos (base nueva,
`pg_restore`, verificar, cambiar de nombre): está en
`docs/operacion/respaldos.md` del repo `sentrylink-local-v2`. No lo hagas con
el mando en uso.

---

## 10. Diagnóstico

```bash
wget -qO- http://127.0.0.1/health
```
(Si en `.env.local` hay un `HTTP_PORT`, va ese puerto: `http://127.0.0.1:<HTTP_PORT>/health`.)
El portal contesta: `{"status":"ok",…}`.

```bash
ip -4 -o addr show
```
Las IP de la máquina (el portal se abre en `http://<IP>`).

```bash
ss -ltnp | grep -E ':(80|1883|5020|8025) '
```
Quién escucha en los puertos del portal (80), MQTT (1883), el simulador (5020)
y Mailpit (8025).

```bash
git -C /opt/genmonitor describe --tags
```
Qué release tiene instalado el servidor.

```bash
git -C /opt/puente-modbus-mqtt describe --tags
```
Qué versión tiene el puente.

```bash
df -h / && free -h
```
Disco y memoria libres.

```bash
cat /root/sentrylink-servidor.respuestas
```
Las respuestas de la obra (Debian/Ubuntu); en Proxmox, en el host:
`/root/sentrylink-<CTID>.respuestas`. No tienen secretos.

---

## Reglas que no se negocian

- **Nunca dos servidores contra el mismo PLC.** Un servidor nuevo se prueba
  contra el simulador, con el cinturón puesto. Al PLC real se pasa solo con
  `--plc-real`, y solo cuando el servidor anterior ya tiene detenidos el puente
  y servicio-comandos.
- **Antes de detener servicio-comandos o el stack:** el selector del tablero en
  **LOCAL**, MANUAL remoto en **0** y el mando deshabilitado. Si no, el tablero
  queda en MANUAL remoto y solo sale con el selector en la puerta.
- **El reset de la parada de emergencia nunca es remoto.** Es cableado.
- **Ningún script descarga ni arranca programas en el PLC.** Eso se hace a
  mano, con Machine Expert y el tablero a la vista.
- **`init.sh --local --reset` borra todo.** Solo en servidores de prueba.
