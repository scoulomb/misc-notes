
# NAS setup

## Configure a VPN to LAN via NAS

VPN connection with VPN app
- can use qvpn 
- or open vpn 

need to open dedicated DNAT port. qvpn android app can be plugged wiht music station android apps and other apps.

## Open DNAT 

Open port for different service
https://www.qnap.com/en/how-to/faq/article/what-are-the-network-ports-used-by-qnap-qts-qutscloud-and-quts-hero-system

However if use VPN just need to open VPN port.

## Can use myQNAP cloud

Actually QNAP cloud offers upnp DNAT auto port based on service we activate.
it also provides a [dynDNS as also seen here](../lab-env/README.md#dyndns) + SSL cert

music station android app also access content that way.

## For music

music station can stream to denon, google nest hub
for streaming flac does not work
but can use file application

flac can work with music station if
add video station from multimedia station + cayin from video station

## kube

See [lab env](../lab-env/others.md#use-nas).