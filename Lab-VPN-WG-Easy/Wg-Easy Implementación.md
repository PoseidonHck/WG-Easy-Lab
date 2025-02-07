- **Próposito:**
	El siguiente documento tiene el fin de poner en practica y aprender el como configurar y aplicar un servidor VPN de WireGuard, para simular el acceso remoto a una LAN empresarial para acceder a recursos internos.
	En esta ocasión haremos uso de [wg-easy](https://github.com/wg-easy/wg-easy), una herramienta especializada en facilitar la implementación y administración del servidor VPN a través de un aplicativo WEB mediante el uso de contenedores **Docker**.
# Objetivos del Laboratorio:

- **Configurar un servidor VPN WireGuard** en una máquina (virtual) que actuará como puerta de enlace (gateway) para la red interna.
- **Configurar clientes VPN** que se conecten al servidor, simulando usuarios que se conectan desde fuera de la LAN corporativa.
- **Verificar la conectividad y el enrutamiento:** acceso a recursos internos (por ejemplo, un servidor web o un servidor de archivos) a través de la VPN.
- **Explorar conceptos de seguridad**, como la generación y el intercambio de claves, y la gestión de reglas de firewall.

# Preparar el Entorno

Para este entorno utilizaremos máquinas virtuales en **VMWare Workstation** para 3 máquinas virtuales:

- Una VM que actuará como el **servidor VPN**. Una máquina Ubuntu Server 24.01
	- Esta máquina contará con el usuario: **`vpn-server:vpn1`**
	- Dirección IP Interna: **`10.0.0.1`** de la red **`10.0.0.0/24`**
	- Además contará con una dirección IP la cual será la que se conectará a internet para poder acceder a ella desde una red externa.
	  
- Una VM que simulará ser un **cliente remoto**.
	- Esta máquina será una máquina ubuntu personal configurada anteriormente.
	  
- Otra VM que represente un recurso interno (por ejemplo, un servidor web).
	- Esta máquina contará con el usuario: **vpn-internal:vpn1**
	- Dirección IP interna: **`10.0.0.2`** de la red **`10.0.0.0/24`**.


# Configuración del Entorno

## Servidor

Nos conectaremos vía **SSH** al servidor para una administración más práctica y sencilla. El servidor debe de tener conexión a internet inicialmente para poder realizar las actualizaciones e instalaciones necesarias, por ello estará conectada la red LAN con acceso a internet a demás de la interfaz de la red interna.

Además esto sirve para simular la dirección IP pública a donde deberíamos de apuntar en una situación real donde un usuario desea conectarse a la red interna empresarial desde casa.

En una situación real, se debe de exponer el puerto del **router** que va a internet y hacer que el tráfico de dicho puerto pase al puerto del servidor del Servidor VPN. 

Con esto claro procedemos a la configuración del servidor:

- **Actualización de Repositorios**
```bash
sudo apt update && sudo apt upgrade -y
```

- **Instalación de Docker**
```bash
sudo apt install -y docker.io docker-compose
sudo systemctl enable --now docker
```

- **Creamos un Directorio wg-easy:**
```bash
mkdir ~/wg-easy && cd ~/wg-easy
```

- **Ejecución del Contenedor de WireGuard Easy WG-Easy:**
	- Crearemos un archivo *docker-compose.yml* con el siguiente contenido:
```yaml
version: "3.8"

services:
  wireguard:
    image: weejewel/wg-easy
    container_name: wg-easy
    restart: unless-stopped
    ports:
      - "51820:51820/udp"
      - "51821:51821/tcp"
    environment:
      - WG_HOST=<IP_PUBLICA_O_LOCAL>
      - PASSWORD=admin
      - WG_PORT=51820
      - WG_DEFAULT_ADDRESS=10.8.0.x
      - WG_ALLOWED_IPS=10.8.0.0/24
    volumes:
      - ./config:/etc/wireguard
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    sysctls:
      - net.ipv4.conf.all.src_valid_mark=1
```

Reemplezando `<IP_PUBLICA_O_LOCAL>` con la dirección IP de la interfaz que utilizará el cliente para conectarse. En este caso utilizaremos una dirección IP de mi LAN **`192.168.1.0/24`**, pero teniendo en cuenta que el servidor interno estará en una red Interna AISLADA, debemos de aplicar una reglas **NAT**.

Esto con la finalidad de redirigir el tráfico que llega por la interfaz en la red 10.10.10.0/24 hacia el servicio WireGuard que está escuchando en la IP de la interfaz 192.168.1.X. Teniendo como resultado que:
- El tráfico que llega a la IP **`10.0.0.X`** en el puerto de *WireGuard*, usualmente **`51820`** UDP, se redirija **(DNAT)** a la IP **`192.168.1.X`** donde realmente está escuchando *wg-easy*.
- La respuesta se reescriba (SNAT o MASQUERADE) para que el servidor interno vea que la conexión proviene de la IP **`10.0.0.X`**, la cual sí conoce.

### Configuración NAT del Servidor para la Conexión del Servidor Interno a la Red VPN

Ahora suponiendo que la dirección IP del servidor que comparte los recursos internos es la **`10.0.0.2`** y que la interfaz es la **`ens34`** podemos emplear las siguientes reglas:

- **Regla DNAT - Redireccionamiento de Destino:**
```bash
sudo iptables -t nat -A PREROUTING -i ens34 -d 10.0.0.2 -p udp --dport 51820 -j DNAT --to-destination 192.168.1.92:51820
```

Esta regla intercepta el tráfico que llegue a 10.0.0.2:51820, servidor interno, y lo redirige a 192.168.1.92:51820.

- **Regla SNAT (IP estática) - Reescritura de Origen**
```bash
sudo iptables -t nat -A POSTROUTING -o ens34 -s 192.168.1.92 -p udp --sport 51820 -j SNAT --to-source 10.0.0.2
```

Para que cuando la respuesta salga desde **WireGuard**, desde **`192.168.1.92`** al servidor de recursos internos, la IP de origen se traduzca a **`10.0.0.2`**, la que el servidor de recursos conoce.

- **Permitir el Reenvío entre Interfaces:**
```bash
sudo iptables -A FORWARD -p udp -d 192.168.1.92 --dport 51820 -m state --state NEW,ESTABLISHED,RELATED -j ACCEPT 
sudo iptables -A FORWARD -p udp -s 192.168.1.92 --sport 51820 -m state --state ESTABLISHED,RELATED -j ACCEPT
```

Luego de esto habilitamos en el servidor el IP forwariding en el kernel, esto se puede realizar con el siguiente comando:

```bash
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

Y con esto ahora podemos ejecutar el contenedor a partir del archivo **docker-compose.yml** que creamos anteriormente.

![[wg-easy-contenedor.png]]

## Servidor Interno

Para la configuración del servidor necesitaremos que este conectado a internet al inicio para poder actualizar los repositorios e instalar las herramientas necesarias:

- **Actualización de Repositorios:**
```bash
sudo apt update && sudo apt upgrade -y
```

- **Instalación de Apache2 y Wireguard**
```bash
sudo apt install apache2 wireguard
```

La herramienta **Apache2** será utilizada para acceder a un aplicativo web que solo podrá ser accesible por el cliente externo a través de la conexión VPN.

Por lo que luego actualizar en instalar los archivos necesarios, debemos de transferir el archivo de configuración para la conexión al servidor de WireGuard, por lo que primero creamos el cliente para luego descargar la configuración del mismo a través del aplicativo web en el servidor VPN.

![[aplicativo-web-easy-wg.png]]

Como podemos ver tenemos la opción de crear clientes y asignar el nombre que queramos, luego podemos utilizar el botón de descargas para descargar el archivo de configuración que deberá usar el cliente para conectarse al red VPN:

![[interanl.conf.png|500]]

Antes de transferir el archivo debemos de recordad que debemos modificarlo, cambiando el **`EndPoint`**:

```yml
EndPoint = 10.0.0.1:51820
```

Dado que es en esta dirección donde si tienen conexión los servidores. Y que gracias a las reglas que aplicamos en el servidor anteriormente a la hora que el servidor interno intente conectarse al servidor VPN, este realizará el reenvío correspondiente para que vaya hacia **wg-easy**.

Ahora este archivo debe de ser transferido a la máquina de servicios internos. Como nuestra máquina cliente, con la cual administramos los servidores, ya no tiene acceso a la máquina de servicio internos, primero compartimos el archivo desde un servidor web simple con python:

![[compartiendo-internal.conf.png]]

Luego desde el servidor VPN, el cual si tiene conexión con la máquina de servicios internos en la interfaz **ens34** **10.0.0.0/24**, y con la máquina cliente, descargamos el archivo con **wget**, luego con **scp** transferimos el archivo a dicho servidor.

![[trans-internal.conf.png]]

Ya con el archivo en el servidor interno, primero cambiaremos su nombre a **wg0.conf** y será movido a **`/etc/wireguard/conf`** para utilizarlo como archivo de conexión:

![[modificación-wg0.conf.png|550]]

Ahora si realizaremos al conexión y si las reglas están bien aplicadas el, deberíamos de poder conectarnos al servidor VPN.

```bash
sudo wg-quick up wg0
```

![[conexión-exitosa.png]]

Y como vemos ahora contamos con la interfaz **wg0** con la cual tenemos una dirección IP **`10.8.0.2`** del servidor VPN. Como último paso debemos de configurar el firewall para que el cliente externo, pueda comunicarse al servidor y acceder al aplicativo web:

```bash
sudo ufw allow from 10.8.0.0/24
```

## Conexión con el Cliente Externo

Ahora desde nuestra máquina local que hará de cliente interno, intentaremos conectarnos a la red VPN y ver si podemos alcanzar el aplicativo web en el servidor interno **`10.8.0.2`**. 

Por lo que ahora debemos crear un cliente nuevo **`External`** en el aplicativo web de Wireguard:

![[creando-external.png]]

Ahora descargamos el archivo de configuración y lo utilizamos para conectarnos al servidor VPN, antes de esto realizamos el mismo cambio de nombre y ubicamos el archivo en **`/etc/wireguard/wg0.conf`**.

![[wg0.cofn-external.png]]

Y ahora ejecutamos la conexión al servidor:

```bash
sudo wg-quick up wg0
```

![[conexión-external.png]]

Y ahora si todo esta configurado de manera correcta, si en el navegador intentamos acceder al aplicativo web del servidor interno en la **`10.8.0.2`**, significa que hemos realizado la conexión VPN de manera correcta y exitosa.

Verificamos conexión con un **ping**:

![[ping-cliente-servidor.png]]

Y tenemos conexión, por lo que ahora desde el navegador podemos acceder al aplicativo web:

![[aplicativoweb.png]]

# Conclusión

Gracias al laboratorio logramos la implementación de un servidor VPN con **WireGuard** una de las soluciones más utilizadas del momento dado a su rapidez y seguridad. Utilizando **WG-Easy**, logramos una implementación y administración fácil y segura del servidor y sus clientes. 

Utilizando la implementación de reglas de Firewall, logramos la comunicación de los servidores internos para simular conexiones de una red interna, logrando que el servidor del aplicativo logrará entrar en la red VPN y así ofrecer dicho recursos a los clientes que se conectaban a la red.