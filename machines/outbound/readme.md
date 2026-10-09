# 🚩HTB - OutBound 

**Categoría:** Web / Escalada de privilegios  
**Plataforma:** Hack The Box  
**Estado:** ✅ Rooted  

---

## Descripción general

OutBound es una máquina activa de Hack The Box (HTB) orientada a la explotación de un cliente de correo web vulnerable. A lo largo del proceso se aplicaron técnicas de reconocimiento, explotación de vulnerabilidades autenticadas, post-explotación y escalada de privilegios, todo dentro de un entorno Linux.

---

## Herramientas utilizadas

| Herramienta         | Propósito                                         |
|---------------------|--------------------------------------------------|
| `nmap`              | Reconocimiento de puertos y servicios            |
| `/etc/hosts`        | Resolución local de dominio                      |
| `echo + tee`        | Edición de hosts con privilegios                 |
| Navegador web       | Acceso a interfaz Roundcube                      |
| `exploit-db`        | Búsqueda de CVEs y PoCs                          |
| `Metasploit`        | Ejecución del exploit con reverse shell         |
| `nano` / `chmod`    | Edición y permisos del exploit                   |
| `base64`            | Decodificación de credenciales                   |
| `ssh`               | Acceso remoto con usuario escalado               |
| `python`            | Ejecución de script de escalada de privilegios  |

---

## Reconocimiento

Se realizó un escaneo con `nmap`, revelando servicios activos y el subdominio `mail.outbound.htb`.  
Se añadió al archivo `/etc/hosts` para la resolución DNS local:
```
echo "10.10.11.77 mail.outbound.htb" | sudo tee -a /etc/hosts
```
<img width="953" height="571" alt="Captura de pantalla 2025-08-02 090713" src="https://github.com/user-attachments/assets/b901e77f-3d89-4679-a1a7-9d4b9e738c85" />


# 🟢Acceso inicial
 
Al acceder a `http://mail.outbound.htb`, se presentó un panel de login de **Roundcube**.

HTB proporciona credenciales iniciales para el usuario `tyler`, permitiendo autenticarse en la interfaz.
<img width="978" height="558" alt="Captura de pantalla 2025-08-02 101953" src="https://github.com/user-attachments/assets/b0121334-5f78-42f8-ad56-a7caf0f561db" />

## Explotación

- **Versión identificada**: Roundcube 1.6.10  
- **Vulnerabilidad**: [CVE-2025-49113](https://www.exploit-db.com/exploits/XXXXX)  
- **Tipo**: Vulnerabilidad autenticada de deserialización PHP no segura en `upload.php`.  
- **Impacto**: Ejecución remota de código (RCE)

Se utilizó un exploit funcional disponible en Exploit-DB.


## Post-explotación

Una vez dentro del sistema:

- Se examinaron archivos del sistema y bases de datos SQL.
- Se identificaron credenciales codificadas en Base64, y una contraseña encriptada.
- Tras decodificarla se accedió como otro usuario.
  
_datos codificados en Base64 con información de los usuarios_
<img width="923" height="577" alt="image" src="https://github.com/user-attachments/assets/85b1479c-9280-4877-ba7c-0f98efaeb59d" />
_Acceso al nuevo usuario tras desencriptar la clave_
<img width="924" height="511" alt="image" src="https://github.com/user-attachments/assets/fb5f9d03-bcce-4c91-968a-6fb64ec661a9" />

En el buzón de este usuario se encontraron dos correos clave:
1. Notificación de **privilegios administrativos** otorgados.
2. Asignación de una **nueva contraseña** según políticas internas.
<img width="1035" height="390" alt="image" src="https://github.com/user-attachments/assets/c25e8aed-c20a-4cc1-b21b-eaa2800079dc" />

---

## Escalada de privilegios

Una vez sabiendo esto utilice SSH para conectarme al equipo remoto con IP 10.10.11.77 del usuario. Me pidio la contraseña pero no hubo drama ya que la habia visto en el inbox.
Y por ultimo, al obtener el acceso remoto, ejecute un exploit de python, que me otorgo acceso root.

---

# Resultado
✅ Acceso completo al sistema

📌 Flags de usuario y root capturadas


---

⚠️ Disclaimer
Esta máquina continúa activa en la plataforma Hack The Box al momento de escribir este documento.
No se incluyen PoCs ni comandos específicos de explotación directa, en cumplimiento con las normas de divulgación ética.

---

Autor:
Ian Solórzano
Estudiante de Ciberseguridad

**Referencias**

Exploit DB - CVE-2025-49113
Metasploit Framework
