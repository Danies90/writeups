# 📂 CTF Writeups — danies90

Colección de writeups de máquinas resueltas en plataformas de hacking ético como **TryHackMe** y **HackTheBox**. Documentados con fines educativos para compartir metodología y aprendizajes con la comunidad.


---

## 🟢 TryHackMe — Easy

| Máquina | CVE / Técnica principal | Tags |
|---|---|---|
| [GhostCat](./tryhackme/easy/GhostCat.md) | CVE-2020-1938 · LFI via AJP · GPG Cracking · sudo zip | `lfi` `tomcat` `gpg` `privesc` |


---

## 🧭 Metodología general

Cada writeup sigue esta estructura:

1. **Reconocimiento** — Escaneo de puertos y servicios con Nmap
2. **Enumeración** — Análisis de versiones, tecnologías y vectores de ataque
3. **Explotación** — Abuso de vulnerabilidades identificadas
4. **Post-explotación** — Movimiento lateral, cracking de credenciales
5. **Escalada de privilegios** — Obtención de acceso root/SYSTEM
6. **Conclusión** — Lecciones aprendidas y mitigaciones

---

## ⚠️ Aviso legal

Todos los writeups documentan máquinas en entornos **controlados y legales** (TryHackMe, HackTheBox). El contenido es exclusivamente educativo. No apliques estas técnicas en sistemas sin autorización explícita.

---
