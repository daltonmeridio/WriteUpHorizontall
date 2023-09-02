# WriteUp Horizontall

Horizontall es una máquina Linux de fácil dificultad donde solo están expuestos los servicios HTTP y SSH. La enumeración del sitio web revela que está construido utilizando el marco Vue JS. Al revisar el código fuente del archivo Javascript, se descubre un nuevo host virtual. Este host contiene el "Strapi Headless CMS", que es vulnerable a dos CVE, lo que permite a posibles atacantes obtener la ejecución remota de código en el sistema como usuario de "strapi". Luego, después de enumerar los servicios que escuchan solo en el host local de la máquina remota, se descubre una instancia de Laravel. Para acceder al puerto en el que escucha Laravel, se utiliza un túnel SSH. El marco Laravel instalado está desactualizado y se ejecuta en modo de depuración. Se puede explotar otro CVE para obtener la ejecución remota de código a través de Laravel como "root".

| Información | Horizontall |
| --- | --- |
| Categorías | Evaluación de vulnerabilidad |
| Área de interés | Análisis de código fuente
Software obsoleto |
| Lenguajes | javascript |

## Matriz de la máquina

![Untitled](WriteUp%20Horizontall%20cbd8d8a9999d4a3b8aec86f3261aeac8/Untitled.png)

## Enumeración

Iniciamos con un escaneo 

sudo nmap -sS --min-rate 5000 -p- --open -n -Pn 10.129.114.150 -oG allPorts

**-sS**: Para el análisis de puertos TCP SYN

**—min-rate**: para que envié 5000 paquetes por segundo (En esta ocasión usamos esta opción           dado que estamos en un entorno controlado)

**-p-** : Nos analiza todo el rango de puerto, los 65536 puertos.

**-n** : Para que nos quite la resolución DNS

**-Pn**: Deshabilita la detección de host

Todo esto lo vamos a pasar al fichero allPorts en formato Grepeable, pues se van a extraer los puertos para su análisis en versiones de puertos y servicios, esto lo haremos con la utilidad de S4vitar extractPorts.

Extraemos y hacemos un escaneo con más profundidad con los puertos encontrados 

![Untitled](WriteUp%20Horizontall%20cbd8d8a9999d4a3b8aec86f3261aeac8/Untitled%201.png)

sudo nmap -sCV -p 22,80 10.129.115.107 -vvv -oN Targeted

![Untitled](WriteUp%20Horizontall%20cbd8d8a9999d4a3b8aec86f3261aeac8/Untitled%202.png)

Podemos ver que tenemos el puerto 22 y 80 abierto, con esto podemos hacer un escaneo de tecnologías, a demás podemos ver en la salida del puerto 80 el nombre de dominio **http://horizontall.htb**

Esto es importante ya que al momento de querer escanear no pude ya que no se conocía el host, así que tuve que agregar la información a mi archivo hosts 

![Untitled](WriteUp%20Horizontall%20cbd8d8a9999d4a3b8aec86f3261aeac8/Untitled%203.png)

Una vez echo esto pude hacer un escaneo de tecnologías, pero lamentablemente no hay nada que me pueda ayudar a pwnear la maquina 

**whatweb -v http://horizontall.htb/ > scanWeb**

![Untitled](WriteUp%20Horizontall%20cbd8d8a9999d4a3b8aec86f3261aeac8/Untitled%204.png)

La enumeración continua con un fuzzing de directorios y un fuzzing a subdominios, podría ser ya que tenemos un nombre de dominio 

**sudo gobuster dns -d www.horizontall.htb -w /usr/share/wordlists/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt -t 100**

Sin embargo a la hora de hacer la enumeración dns no obtenemos nada, que pena 

![Untitled](WriteUp%20Horizontall%20cbd8d8a9999d4a3b8aec86f3261aeac8/Untitled%205.png)

Hagamos el fuzzing de directorios a ver que tal 

**sudo gobuster dir -u http://horizontall.htb -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -t 100**

![Untitled](WriteUp%20Horizontall%20cbd8d8a9999d4a3b8aec86f3261aeac8/Untitled%206.png)

Esta vez si tenemos algo de información aunque no suficientes pues esos directorios no son visibles, que pena de nuevo, seguiremos enumerando, pero toca visitar la web a ver que tal 

La pagina no muestra mucho y no tiene enlaces 

![Untitled](WriteUp%20Horizontall%20cbd8d8a9999d4a3b8aec86f3261aeac8/Untitled%207.png)

Hay un pequeño formulario de contacto al final de la misma, intente enviar información pero en realidad no envía nada 

![Untitled](WriteUp%20Horizontall%20cbd8d8a9999d4a3b8aec86f3261aeac8/Untitled%208.png)

## Punto de apoyo inicial

Este punto es critico ya que me quede atorado un buen rato, hasta que hice lo que siempre hago en entornos reales, jugar con la consola del programador

Viendo las peticiones que hace el servidor hubo dos cosas que me llamaron la atención: **app.c68eb462.js** y **chunk-vendors.0e02b89e.js**

![Untitled](WriteUp%20Horizontall%20cbd8d8a9999d4a3b8aec86f3261aeac8/Untitled%209.png)

Cuando continuo al depurador me doy cuenta que lo mismo que me llamo la atención están aquí, sin embargo **chunk-vendors.0e02b89e.js** no tiene algo interesante todo viene en texto plano y no encuentro algo que me sea útil

Por otro lado **app.c68eb462.js** si lo encuentro útil ya que para empezar por que esta ofuscado, si no tuviera algo que valga la pena ocultar no estaría oculto, ¿No? ¿Tu que piensas? 

![Untitled](WriteUp%20Horizontall%20cbd8d8a9999d4a3b8aec86f3261aeac8/Untitled%2010.png)

Si así es, así se ve algo ofuscado, llevemos esto a un desofuscador elegí este de [aquí](https://deobfuscate.io/) y realice la desofuscación 

![Untitled](WriteUp%20Horizontall%20cbd8d8a9999d4a3b8aec86f3261aeac8/Untitled%2011.png)

Para mayor comodidad llevé esto a un editor de textos, créeme es más cómodo, es mucho código tomate tu tiempo, cuando estuve leyendo el código pude notar esto 

![Untitled](WriteUp%20Horizontall%20cbd8d8a9999d4a3b8aec86f3261aeac8/Untitled%2012.png)

Así es, una llamada a un subdominio, siempre es bueno revisar todo lo que hay en la pagina o el servidor, en este caso fue vital ya que solo tenemos dos puertos abiertos y el fuzzing de directorios y dns con las listas y herramientas convencionales no descubrió nada, siempre debemos fijarnos en lo que ocurre tras bambalinas, así podemos tener todo el panorama completo 

Teniendo esto en cuenta, ahora podemos agregar esto a nuestro archivo hosts y vemos que trae esto en el navegador 

![Untitled](WriteUp%20Horizontall%20cbd8d8a9999d4a3b8aec86f3261aeac8/Untitled%2013.png)

Si visitamos la pagina tal cual la encontramos nos encontramos un archivo JSON  

Y si visitamos la pagina sin el directorio solo tenemos un mensaje de bienvenida 

![Untitled](WriteUp%20Horizontall%20cbd8d8a9999d4a3b8aec86f3261aeac8/Untitled%2014.png)

Por lo que el siguiente paso o lo que se me ocurrió fue hacer un fuzzing de directorios 

**sudo gobuster dir -u http://api-prod.horizontall.htb/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -t 100**

Lo cual ahora si nos revela información relevante, como primer instancia hay directorios que nos llaman la atención como admin, que pudiera ser un panel del administración o información hay que darle una visitada 

![Untitled](WriteUp%20Horizontall%20cbd8d8a9999d4a3b8aec86f3261aeac8/Untitled%2015.png)

Y en efecto es un panel de loggin para el administrador sin embargo no tenemos credenciales, intente buscar las credenciales predeterminadas pero no funcionaron las que encontre, entonces decidí por clásicos ataques, que tampoco dieron éxito 

![Untitled](WriteUp%20Horizontall%20cbd8d8a9999d4a3b8aec86f3261aeac8/Untitled%2016.png)

En esta parte como la enumeración con gobuster no nos arrojo mas información, decidí repetir el ejercicio anterior y ver que recursos que el servidor llama por detrás 

De esto me llamo la atención el archivo init, lo cual usualmente son archivos de configuración 

![Untitled](WriteUp%20Horizontall%20cbd8d8a9999d4a3b8aec86f3261aeac8/Untitled%2017.png)

Hay dos enlaces, tuve que visitar los dos, fue el segundo enlace el que me dio la información requerida, la versión de strapi 

![Untitled](WriteUp%20Horizontall%20cbd8d8a9999d4a3b8aec86f3261aeac8/Untitled%2018.png)

Ya teniendo la versión del CMS pude hacer una búsqueda de vulnerabilidades y exploits 

Con esto a la mano encontré varios CVE que son 

- **CVE-2019-18818**
    
    Esta vulnerabilidad maneja mal el restablecimiento de la contraseña en **packages/strapi-admin/controller/Auth.js** y **packages/strapi-plugin-users-permissions/controllers/Auth.js** esto nos permite cambiar la contraseña de un usuario de altos privilegios y asimismo garantizar el acceso no autorizado
    
- **CVE-2019-19609**
    
    Documentada tiempo después se encontró esta vulnerabilidad la cual permite la ejecución de código remoto , para la explotación de esta vulnerabilidad necesitamos el JSON Wen Token 
    

En este sentido encontré dos diferentes exploits para la primer vulnerabilidad, el primero este de [aquí](https://www.exploit-db.com/exploits/50237) sin embargo no es tan funcional, para que no pierdas tanto tiempo. 

Ambos exploits se complementan, por lo que use uso de este [exploit](https://www.exploit-db.com/exploits/50239) 

la forma de  descargar estos archivos desde github es: 

**wget http://raw.githubusercontent.com/z9fr/CVE-2019-19609/main/exploit.py**

Siempre te recomiendo usar el archivo crudo, si el exploit es de database, ahí mismo tiene la opción de descarga 

Su uso es meramente trivial

![Untitled](WriteUp%20Horizontall%20cbd8d8a9999d4a3b8aec86f3261aeac8/Untitled%2019.png)

Esto es importante ya que esto nos arroja el JSON Web Token, que usaremos en el exploit de la CVE-2019-19609, ´para este exploit encontré una [PoC](https://www.youtube.com/watch?v=alhZJmuUd2s&feature=youtu.be&themeRefresh=1) y también el enlace al exploit [aquí](https://github.com/z9fr/CVE-2019-19609/tree/main) 

Su uso al igual que el exploit pasado es trivial, el mismo video lo explica, ahora usando estos dos exploits nos permite obtener una shell sobre el sistema 

![Untitled](WriteUp%20Horizontall%20cbd8d8a9999d4a3b8aec86f3261aeac8/Untitled%2020.png)

Recuerda levantar un oyente para obtener la shell con **rlwrap nc -nlvp 9001**

![Untitled](WriteUp%20Horizontall%20cbd8d8a9999d4a3b8aec86f3261aeac8/Untitled%2021.png)

Ahora ya tenemos una shell del sistema 

## Elevación de privilegios

Para la elevación de privilegios, intente los clásicos comandos de enumeración, sin embargo no fueron de éxito, sin embargo encontré el directorio **/opt/strapi.** Cuando no encontré nada use el comando **netstat -auntp** que me permitió ver las comunicaciones internas del servidor, con lo que pude ver que el puerto 8000 se esta comunicando con algo, use **curl** para ver cada puerto local 

![Untitled](WriteUp%20Horizontall%20cbd8d8a9999d4a3b8aec86f3261aeac8/Untitled%2022.png)

![Untitled](WriteUp%20Horizontall%20cbd8d8a9999d4a3b8aec86f3261aeac8/Untitled%2023.png)

**curl 127.0.0.1:8000**

![Untitled](WriteUp%20Horizontall%20cbd8d8a9999d4a3b8aec86f3261aeac8/Untitled%2024.png)

Por lo que vemos es un servicio que usa Laravel v8, evidentemente jaja, con la información de la versión busquemos si hay algún exploit o vulnerabilidad para este servicio que tenemos 

Para esta versión de Laravel encontré el **CVE-2021-3129** encontrarlo fue relativamente fácil, también me encontré con [este](https://github.com/nth347/CVE-2021-3129_exploit) exploit público que me permitiría ejecutar código de forma remota

Sin embargo, esto no podría funcionar así de fácil, en principio el servicio no esta expuesto al internet, pues en un principio no lo vimos cuando se hicieron los escaneos con nmap, se comunican  de forma interna, así que tendremos que hacer un reenvió de puertos (port forwarding)

## Port forwarding

Permite que servidores y dispositivos remotos en internet accedan a servidores dentro de redes privadas y viceversa. Sin el reenvió de puertos solo los dispositivos y servidores que se encuentran dentro de la red privada pueden comunicarse entre si, el reenvió de puertos permite que se puedan comunicar con quien sea en internet. 

Para hacer el reenvió de puertos usaremos **ssh** para poder tener un shell y obtener el servicio expuesto en nuestro navegador. 

Primero tenemos que crear unas llaves **ssh**

![Untitled](WriteUp%20Horizontall%20cbd8d8a9999d4a3b8aec86f3261aeac8/Untitled%2025.png)

Ahora tiendes tu llave publica y la llave privada, necesitamos la publica, esta la pasáramos a la maquina, al directorio **/opt/strapi** dentro de ella hay que crear el directorio **.ssh** usemos **mkdir .ssh** y ahí lo que tenemos que hacer es levantar un servidor con **python** en nuestra maquina local y descargar la llave publica desde la maquina 

![Untitled](WriteUp%20Horizontall%20cbd8d8a9999d4a3b8aec86f3261aeac8/Untitled%2026.png)

Con esto ya en la maquina tenemos que reemplazar la llave con authorized_keys

**mv id_rsa.pub authorized_keys**

![Untitled](WriteUp%20Horizontall%20cbd8d8a9999d4a3b8aec86f3261aeac8/Untitled%2027.png)

Con esto echo podemos hacer un reenvió de puertos

**ssh -i id_rsa strapi@10.10.10.10 -L 8000:localhost:8000**

Desglosare este comando en dos partes

**-i** : Indica que usaremos nuestra llave privada para que se “conecta” con la llave publica

**strapi@10.10.10.10** : Es la dirección a la cual accederemos, por nombre de usuario e ip o dirección de dominio

La forma de esta primera parte se puede ver así 

**ssh [usuario]@[ip-destino]**

La segunda parte del comando puede verse así

**-L [Puerto-local]:[ip-destino]:[puerto-destino]**

**-L** : Para indicar que lo vamos hacerlo de forma local, es decir queremos traer esto a nuestro host de ataque 

![Untitled](WriteUp%20Horizontall%20cbd8d8a9999d4a3b8aec86f3261aeac8/Untitled%2028.png)

Ahora que tenemos la shell y el servicio expuesto de nuestro lado podemos ver este mismo en nuestro localhost

![Untitled](WriteUp%20Horizontall%20cbd8d8a9999d4a3b8aec86f3261aeac8/Untitled%2029.png)

Con este servicio expuesto de nuestro lado podemos hacer uso del exploit antes mencionado, ya que ahora si lo podemos alcanzar.

Su uso es muy trivial

![Untitled](WriteUp%20Horizontall%20cbd8d8a9999d4a3b8aec86f3261aeac8/Untitled%2030.png)

Por lo que ahora podemos ejecutar comando como el usuario root.

Otras formas de encontrar el ROOT

Si bien la manera en que llegamos al root es una buena forma de hacer las cosas pues tuvimos que investigar y preparar todos los escenarios, hay otra forma mas rápida de llegar al root 

Para esto usaremos el **CVE-2021-4034** que es una vulnerabilidad de escalada de privilegios locales en pkexec de polkit, el exploit lo puedes encontrar [aquí](https://raw.githubusercontent.com/arthepsy/CVE-2021-4034/main/cve-2021-4034-poc.c) 

La forma de hacer que esto funcione es la siguiente 

descargue el exploit en mi maquina persona 

**wget https://raw.githubusercontent.com/arthepsy/CVE-2021-4034/main/cve-2021-4034-poc.c**

Lo renombre 

**mv cve-2021-4034-poc.c poc.c** 

Lo trasladamos a la maquina  victima levantando un servidor de pyhon en el directorio /tmp pues es el único que nos dejaría poder ejecutar archivos binarios, seguido de esto ejecute los siguientes comandos 

**gcc 'poc.c' -o exploit**

**gcc 'poc.c' -o exploit**

**./exploit**

![Untitled](WriteUp%20Horizontall%20cbd8d8a9999d4a3b8aec86f3261aeac8/Untitled%2031.png)

¡Obtuvimos una shell con todos los privilegios de administrador!

Muy bien, espero que te gustara, si te fue de ayuda para aprender nuevas cosas, por favor comparte. 

Happy Hacking!
