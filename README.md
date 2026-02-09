                              
 
 
VPN WireGuard amb Docker 
 
Curs 25/26 Grup IFC31C Data 
lliurament 
04/Febrer 
Mòdul SISAD 
Títol VPN WireGuard amb Docker 
 
 
Tipus de treball grup 
Pautes de realització (indicar en primer lloc si el treball és individual o en grup): 
1. Crear dues imatges Docker pròpies (server i client) amb Ubuntu base 
 
2. Instal·lar WireGuard manualment dins dels contenidors 
 
3. Configurar una VPN punt-a-punt segura entre els contenidors 
 
4. Practicar resolució d’errors i troubleshooting 
 
5. Verificar la comunicació i fer diagnòstic de la xarxa 
 
 
 
 
 
 
 
 
 
 
 
 
Contenidors construïts correctament  
WireGuard instal·lat manualment 
Configuració wg0.conf correcta (server i client) 
Connexió VPN funcionant (ping, acces a servei) 
 
 
 
 
  
MDO20303 - Pàg. 1/6 
1. Topologia de xarxa 
+-------------------+        
+-------------------+ 
| Contenidor Client | <----> | Contenidor Server | 
|  wg0: 10.9.0.2    
|        
+-------------------+        
|  wg0: 10.9.0.1    
| 
+-------------------+ 
\____________________ VPN WireGuard 
____________________/ 
● Xarxa VPN: 10.9.0.0/24 
● Port UDP exposat: 51820 
Observació: L’endpoint del client és el nom del contenidor servidor 
(wireguard-server) dins de la xarxa Docker. 
2. Estructura del projecte 
wireguard-2containers-vpn/ 
├── server/ 
│   
├── Dockerfile 
│   
└── start.sh 
├── client/ 
│   
├── Dockerfile 
│   
└── start.sh 
└── keys/   # generades al host 
3. Generació de claus (host) 
mkdir keys && cd keys 
# Servidor 
wg genkey | tee server.key | wg pubkey > server.pub 
# Client 
wg genkey | tee client.key | wg pubkey > client.pub 
MDO20303 - Pàg. 2/6 
4. Dockerfiles 
Server Dockerfile 
FROM ubuntu:22.04 
WORKDIR /root 
COPY start.sh /start.sh 
RUN chmod +x /start.sh 
CMD ["/bin/bash"] 
Client Dockerfile 
FROM ubuntu:22.04 
WORKDIR /root 
COPY start.sh /start.sh 
RUN chmod +x /start.sh 
CMD ["/bin/bash"] 
Observació: L’alumne farà apt install wireguard iproute2 
iptables manualment dins dels contenidors. 
5. Scripts d’arrencada buits 
server/start.sh 
#!/bin/bash 
echo "Contenidor servidor iniciat" 
bash 
client/start.sh 
#!/bin/bash 
echo "Contenidor client iniciat" 
bash 
6. Creació de la xarxa Docker 
docker network create wg-net 
MDO20303 - Pàg. 3/6 
7. Construcció de les imatges 
docker build -t wg-server ./server 
docker build -t wg-client ./client 
8. Execució dels contenidors 
Servidor 
docker run -it --name wireguard-server \ --network wg-net \ --cap-add=NET_ADMIN \ --cap-add=SYS_MODULE \ -p 51820:51820/udp \ 
wg-server 
Client (altre terminal) 
docker run -it --name wireguard-client \ --network wg-net \ --cap-add=NET_ADMIN \ 
wg-client 
NOTA: --cap-add=NET_ADMIN 
● Què és: És una “capabilitat” del kernel que permet al contenidor executar operacions de xarxa 
privilegiades. 
● Per què és necessari a WireGuard: 
WireGuard necessita: 
○ Crear interfícies de xarxa virtuals (wg0) 
○ Configurar rutes dins del contenidor 
○ Modificar regles d’iptables (si s’utilitza NAT o forwarding) 
Sense NET_ADMIN: wg-quick up wg0 
generaria errors com: RTNETLINK answers: Operation not permitted 
● Resum: Dona permisos de “administració de xarxa” dins del contenidor, sense donar-li permisos 
root complets al host. 
MDO20303 - Pàg. 4/6 
--cap-add=SYS_MODULE 
● Què és: Permet al contenidor carregar o descarregar mòduls del kernel. 
● Per què és necessari a WireGuard: 
○ WireGuard pot utilitzar un mòdul del kernel (wireguard.ko) per accelerar el tràfic. 
○ Quan fas modprobe wireguard dins del contenidor, necessites aquesta capabilitat. 
Sense SYS_MODULE: modprobe wireguard 
pot fallar amb: modprobe: FATAL: Module wireguard not found in directory 
/lib/modules/... 
● Resum: Dona permisos per carregar mòduls del kernel, útil quan el kernel del host té WireGuard 
com a mòdul. 
9. Instal·lació manual dins dels contenidors 
Servidor 
apt update 
apt install wireguard iproute2 iptables -y 
sysctl -w net.ipv4.ip_forward=1 
mkdir /etc/wireguard 
nano /etc/wireguard/wg0.conf 
wg0.conf (server) 
[Interface] 
Address = 10.9.0.1/24 
ListenPort = 51820 
PrivateKey = <CONTINGUT DE server.key> 
[Peer] 
PublicKey = <CONTINGUT DE client.pub> 
AllowedIPs = 10.9.0.2/32 
Client 
apt update 
apt install wireguard iproute2 iputils-ping -y 
mkdir /etc/wireguard 
MDO20303 - Pàg. 5/6 
nano /etc/wireguard/wg0.conf 
wg0.conf (client) 
[Interface] 
Address = 10.9.0.2/24 
PrivateKey = <CONTINGUT DE client.key> 
[Peer] 
PublicKey = <CONTINGUT DE server.pub> 
Endpoint = wireguard-server:51820 
AllowedIPs = 10.9.0.0/24 
PersistentKeepalive = 25 
10. Arrancar la VPN 
Servidor 
wg-quick up /etc/wireguard/wg0.conf 
Client 
wg-quick up /etc/wireguard/wg0.conf 
11.Verificació 
● Comprovar interfícies i estat de WireGuard 
● Ping del client al servidor 
● Verificar que el client pot llegir informació generada per el servidor 
accedint a través de la VPN 
Si funciona: VPN correcta. 
MDO20303 - Pàg. 6/6 
