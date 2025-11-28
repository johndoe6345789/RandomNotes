
Existing config
```

**pi@wardpi**:**/mnt/part2**$ docker ps

docker volume ls

docker images

CONTAINER ID   IMAGE                                       COMMAND                  CREATED       STATUS                   PORTS                                                                                                                                                                                            NAMES

149a9abaf706   itzg/minecraft-server                       "/image/scripts/start"   4 weeks ago   Up 3 minutes (healthy)   0.0.0.0:42069->25565/tcp, [::]:42069->25565/tcp                                                                                                                                                  experimental2-mc-1

5389f36712f8   lscr.io/linuxserver/openssh-server:latest   "/init"                  4 weeks ago   Up 3 minutes             0.0.0.0:2225->2222/tcp, [::]:2225->2222/tcp                                                                                                                                                      experimental2-openssh-server-two-1

0d37b4f9f451   993e0f5c60d6                                "/image/scripts/start"   5 weeks ago   Up 3 minutes (healthy)   0.0.0.0:25566->25565/tcp, [::]:25566->25565/tcp, 0.0.0.0:25576->25575/tcp, [::]:25576->25575/tcp                                                                                                 experimental-mc-1

5d1cdf5022e9   993e0f5c60d6                                "/image/scripts/start"   5 weeks ago   Up 3 minutes (healthy)   0.0.0.0:8123->8123/tcp, [::]:8123->8123/tcp, 0.0.0.0:25565->25565/tcp, [::]:25565->25565/tcp, 0.0.0.0:25575->25575/tcp, [::]:25575->25575/tcp, 0.0.0.0:19132->19132/udp, [::]:19132->19132/udp   linux-mc-1

9e75f9a29b6d   portainer/portainer-ce:latest               "/portainer"             5 weeks ago   Up 3 minutes             0.0.0.0:8000->8000/tcp, [::]:8000->8000/tcp, 0.0.0.0:9443->9443/tcp, [::]:9443->9443/tcp, 9000/tcp                                                                                               portainer

DRIVER    VOLUME NAME

local     3a8daa19bfa359e86baf12778abcfb319db6d7aec775cc2d0b3235b1ff001d71

local     3defc461cf2edda486d03f32ad32df41adc3f5058b3a65b33fe5bf2d581694f7

local     3eeecd3bca7ff742359d356ee57700459d56042df9af4aad740f2d6c2a93ee55

local     5c73ee521bdf5ffbd30ad1b5f279d0a9f22e7dbbe4194c0e8410c7ca1da46b6f

local     5f71e89f25f816b7a37edbeecf26e8196c049fb38c60c8c3844bca01f538bb2f

local     5f499ad3fa6adc804fbdf5f691dc6119065a656efbf92f5405584cdc35019e4f

local     6b5a614e056031771c31c05a5af2f2708dad8ddb7e312956fa063a68854fe9c0

local     7c6ec928fe0183c792b869260d6f7ac296bbf62a5d01915e9eec4cdf00522834

local     8cc6fc00f598333db06acde5953f3b0f62e32788249162271511f61f48dc43b2

local     09ac27d3b99bc651650f2ef6aa679d3236ce20f7d9041d1992a09c979aebb37c

local     10d21fb0debdbf18e401e00fbb145bfbe4bfb65fccba5ae4878857151bd31c16

local     28a54a2609a3831f96b6369833adc977d565cbcf408d9aeb874cdf45f87e94e5

local     079a5dd35bd5300207bcf1e0258e1b62ed7183f450a0e18664322ae2faa4bc42

local     557ad7d58689a8e34193f9a213bea4582b364559d5eb799465a179cd1f19343d

local     780f2b1e741939dcc0e474000f3263fe3c6c591d6f1bac44b9920cd2e88c8bf0

local     2471ac6274d5dca66ca802f12f4398366b8984fa7e151f68f297a7c58cfdeff9

local     04502c0b67bee66c9cad39a57d198dedf0761f545c73db35ec238342d7f3f4b6

local     32507c06247f7d5ed781b44b648bb45a2f863f0bc6a7418a7ac4090eb7064b5b

local     79616858aac3b7b75a6ae771c07e54f8921091ec768b555826bb1bc2d5807c1b

local     b4d8f1c8d8bccfc8003c2001cae7f7ce0a2bcb0fd7fcc7d7686e0fe9c75c37e0

local     ba02f52e1263e17abead70be933403c4173de23ec535d0b94feb03eec80cfd21

local     c99d191bef51e6fc8032b9f96f2699c497325bb631292a422870a1eab0e0de0b

local     d66ec95b11d33681d556e3298a9b128c10123dedf0ccdb6042007008d922f8a1

local     d603a82f1679711acdfdf7847e168663efb303832056224c3027cdef8676b2ad

local     dcac1bd8898db9c5fb9e55e1ba24057ee057c169f73bb0d9938057fce16f97be

local     e430734811f71379873ee9a56b1cb0bf99fcce98cde7ff50fa9014edfb1d4c9d

local     experimental2_minecraft

local     experimental2_mods

local     experimental2_plugins

local     experimental_minecraft

local     experimental_mods

local     experimental_plugins

local     f1ee8bcda22ba616f9724150e0d6d654381f3abfc344fb6b2ffd1037ba85cfe2

local     f50c70131a680e3d11a6adbe0cf031351f3f4304f27adeeb204740458bba8bfb

local     fc88e4df8820032bb5ecbecc2cf2c7287c038dc31840dd33137a1cf7541f14dd

local     homeserver2_acme

local     homeserver2_certs

local     homeserver2_conf

local     homeserver2_dhparam

local     homeserver2_html

local     homeserver2_phpapache2

local     homeserver2_phpini

local     homeserver2_phplib

local     homeserver2_phpvolume

local     homeserver2_richwdb

local     homeserver2_storagefilesconf

local     homeserver2_storagefilesvolume

local     homeserver2_vhost

local     homeserver2_webrootapache2

local     homeserver2_webrootini

local     homeserver2_webrootlib

local     homeserver2_webrootvolume

local     linux_minecraft

local     linux_mods

local     linux_plugins

local     portainer_data

                                                                                                    **i** Info →   U  In Use

**IMAGE**                                       **ID**             **DISK USAGE**   **CONTENT SIZE**   **EXTRA**

**alpine:latest**                               171e65262c80       8.51MB             0B        

**arm64v8/php:8.1-apache**                      034b63e1ea39        512MB             0B        

**httpd:latest**                                5b8390563802        147MB             0B        

**itzg/minecraft-server:latest**                39c231eb7047        861MB             0B    U   

**lscr.io/linuxserver/openssh-server:latest**   8f7e156dc73b       49.1MB             0B    U   

**mysql:latest**                                90ec22e21be2        939MB             0B        

**nginxproxy/acme-companion:2.4**               7b55348294c6       52.7MB             0B        

**nginxproxy/acme-companion:latest**            a927bc1b0478       51.5MB             0B        

**nginxproxy/nginx-proxy:alpine**               ba6a2076e6f3       76.6MB             0B        

**phpmyadmin:latest**                           52ad6a8d0b7c        589MB             0B        

**portainer/helper-reset-password:latest**      7ff0f698d2ad       42.4MB             0B        

**portainer/portainer-ce:2.20.3**               27ec4f5cca9c        280MB             0B        

**portainer/portainer-ce:latest**               f8f6fc245e62        180MB             0B    U   

**pi@wardpi**:**/mnt/part2**$

```

New ones:
- Vaultwarden
- Nextcloud - laptop
- Jellyfin/Emby - laptop?
- Some kind of docker IRC
- Some kind of docker XMPP / Jabber
- Dozzle + Portainer
- [Container | Jellyfin](https://jellyfin.org/docs/general/installation/container/)
- 
