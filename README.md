# sentrylink-instalar

Arranque del instalador de SentryLink Local v2, en una línea. En un Debian 13,
un Ubuntu 26.04 o un host Proxmox, como root:

```bash
curl -fsSL https://raw.githubusercontent.com/walc72/sentrylink-instalar/main/instalar | sudo bash
```

Pide un token de GitHub (no se guarda), baja el último release del repo
privado `walc72/sentrylink-local-v2` y lanza el instalador que corresponde a la
máquina. Leé [`instalar`](instalar) antes de correrlo: es corto.
