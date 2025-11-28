
Jellyfish

```

version: "3.9"

services:
  jellyfin:
    image: jellyfin/jellyfin:latest
    container_name: jellyfin
    network_mode: host
    volumes:
      - /srv/jellyfin/config:/config
      - /srv/jellyfin/cache:/cache
      - /media:/media
    restart: unless-stopped



mkdir -p /srv/jellyfin/{config,cache}



```

Use yaght not portainer

tidy volume spam

```

docker volume ls -f dangling=true --format '{{.Name}}' \
| grep -E '^[0-9a-f]{20,64}$' \
| xargs -r docker volume rm


```

ssh webui

```

version: "3.9"

services:
  sshwifty:
    image: niruix/sshwifty:latest
    container_name: sshwifty
    restart: unless-stopped
    ports:
      - "8182:8182"
    environment:
      # Basic config
      SSHWIFTY_HOST: "0.0.0.0"
      SSHWIFTY_PORT: "8182"

      # Optional: set a simple login (NOT for exposure to the internet directly)
      SSHWIFTY_AUTH: "basic"
      SSHWIFTY_BASIC_USERNAME: "admin"
      SSHWIFTY_BASIC_PASSWORD: "changeme"

      # Example default target(s) – user fills in host/port in UI anyway
      SSHWIFTY_PRESETS: |
        [
          {
            "Name": "My NAS",
            "Type": "SSH",
            "Host": "192.168.1.10",
            "Port": 22
          },
          {
            "Name": "RPi",
            "Type": "SSH",
            "Host": "192.168.1.20",
            "Port": 22
          }
        ]

    # Optional: if you want to bind only to LAN or localhost, adjust this
    networks:
      - sshlab

networks:
  sshlab:
    driver: bridge

```

