sudo rsync -aHAX /mnt/part2/var/lib/docker/ /var/lib/docker/

sudo systemctl stop docker.service docker.socket containerd.service 2>/dev/null || true
sudo systemctl stop docker 2>/dev/null || true

ps aux | grep dockerd

sudo mv /var/lib/docker /var/lib/docker.NEW_EMPTY_BACKUP.$(date +%Y%m%d-%H%M%S)
sudo mkdir -p /var/lib/docker

sudo usermod -aG docker "$USER"

sudo systemctl start docker
sudo systemctl status docker

docker ps -a
docker images
docker volume ls
docker network ls

tar does --index-file

