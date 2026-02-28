# Trilium note on docker compose 

### `note.domain.com` указан как пример
### Up memos server note on docker
### После установки докера и докер compose 

### 1. Создадим папку для проекта
```
mkdir -p /etc/docker/note
cd /etc/docker/note
```
### 2. Создаем `docker-compose.yml` и поднимаем контейнер 

```
nano docker-compose.yml
```

#### Cодержимое `docker-compose.yml`
```yml
# Running `docker-compose up` will create/use the "trilium-data" directory in the user home
# Run `TRILIUM_DATA_DIR=/path/of/your/choice docker-compose up` to set a different directory
# To run in the background, use `docker-compose up -d`
services:
  trilium:
    # Optionally, replace `latest` with a version tag like `v0.90.3`
    # Using `latest` may cause unintended updates to the container
    image: triliumnext/notes:latest
    # Restart the container unless it was stopped by the user
    restart: unless-stopped
    environment:
      - TRILIUM_DATA_DIR=/home/node/trilium-data
    ports:
      # By default, Trilium will be available at http://localhost:8080
      # It will also be accessible at http://<host-ip>:8080
      # You might want to limit this with something like Docker Networks, reverse proxies, or firewall rules,
      # however be aware that using UFW is known to not work with default Docker installations, see:
      # https://docs.docker.com/engine/network/packet-filtering-firewalls/#docker-and-ufw
      - '127.0.0.1:8996:8080'
    volumes:
      # Unless TRILIUM_DATA_DIR is set, the data will be stored in the "trilium-data" directory in the home directory.
      # This can also be changed with by replacing the line below with `- /path/of/your/choice:/home/node/trilium-data
      - ${TRILIUM_DATA_DIR:-~/trilium-data}:/home/node/trilium-data
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro                      
```
#### Поднимаем контейнер через команду `docker compose up -d` так же смотрим статус контейнера ниже пример того чего мы должны увидеть
```bash
root@server-gw-git:/etc/docker/memos# docker ps
CONTAINER ID   IMAGE                       COMMAND                  CREATED        STATUS                    PORTS                                                               NAMES
fe6dd6a498c0   triliumnext/notes:latest    "docker-entrypoint.s…"   24 hours ago   Up 24 hours (healthy)     127.0.0.1:8996->8080/tcp                                            notes-test-trilium-1
root@server-gw-git:/etc/docker/memos#
```
### 3. Настраиваем nginx т.к безопаснее будет заходить на сайт по домен
##### Если у вас не установлен `nginx` и `certbot` то устанавливаем и включаем 
```
apt install certbot python3-certbot-nginx nginx
systemctl enable --now nginx
```
##### Настройка самого nginx
##### Создаем файл 
```
nano /etc/nginx/sites-available/note.domain.com
```
#### Содержимое файла `note.domain.com`
```
server {
    server_name note.domain.com;

    location / {
        client_max_body_size 512M;
        proxy_pass http://127.0.0.1:8996;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        add_header Strict-Transport-Security "max-age=15552000; includeSubDomains" always;
    }
}
```
### 4. Обязательно делаем ссылку без нее не будет работать и перезапускаем nginx
```bash
ln -s /etc/nginx/sites-available/note.domain.com /etc/nginx/sites-enabled/
systemctl restart nginx
```

### 5. Получение сертификата
Пишем команду `certbot --nginx -d note.domain.com` после этого вы можете переходить на доменное имя `note.domain.com`
И перезапускаем nginx
```bash
systemctl restart nginx
```

