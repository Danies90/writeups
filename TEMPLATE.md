# [Nombre de la máquina] — [Plataforma] Writeup

**Dificultad:** Easy / Medium / Hard  
**Plataforma:** TryHackMe / HackTheBox  
**CVE principal:** CVE-XXXX-XXXX *(si aplica)*  
**Técnicas:** Técnica 1 · Técnica 2 · Técnica 3

---

## Índice

1. [Reconocimiento](#1-reconocimiento)
2. [Enumeración de servicios](#2-enumeración-de-servicios)
3. [Explotación](#3-explotación)
4. [Acceso inicial](#4-acceso-inicial)
5. [Post-explotación](#5-post-explotación)
6. [Escalada de privilegios](#6-escalada-de-privilegios)
7. [Conclusión](#7-conclusión)

---

## 1. Reconocimiento

```bash
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn <IP> -oG scan.txt
```

**Puertos abiertos:**

```
PORT     STATE SERVICE
XX/tcp   open  XXXX
```

---

## 2. Enumeración de servicios

```bash
nmap -p<PUERTOS> -sCV <IP> -oG ports.txt
```

**Hallazgos relevantes:**

```
PORT    STATE SERVICE  VERSION
XX/tcp  open  XXXX     vX.X.X
```

---

## 3. Explotación

> Descripción breve de la vulnerabilidad o vector de ataque identificado.

```bash
# Comando de explotación
```

**Resultado:**

```
# Output relevante
```

---

## 4. Acceso inicial

```bash
ssh usuario@<IP>
```

---

## 5. Post-explotación

> Descripción de qué se encontró tras el acceso inicial.

```bash
# Comandos relevantes
```

---

## 6. Escalada de privilegios

```bash
sudo -l
```

**Vector encontrado:**

```
(root) NOPASSWD: /usr/bin/XXXX
```

```bash
# Exploit de escalada
```

---

## 7. Conclusión

| Fase | Técnica | Herramienta |
|---|---|---|
| Reconocimiento | Port scanning | Nmap |
| Explotación | ... | ... |
| Escalada | ... | ... |

### Lecciones aprendidas

- ...
- ...
- ...

---

*Writeup realizado con fines educativos en un entorno controlado de [Plataforma].*
