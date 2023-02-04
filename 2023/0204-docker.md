---
theme: seriph
background: https://minio.momobako.com:9000/nanahira/koishi-talk/20211218-thirdeye.jpg
class: text-center
highlighter: shiki
---

# Koishi Talk - External

<div class="opacity-80">
33.5th / 2023-02-04
</div>

---

# Docker & Docker-Compose

```yaml
version: '2.4'
services:
  mysql:
    restart: always
    image: mariadb:10
    volumes:
      - ./db:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: koishi
  koishi:
    image: koishijs/koishi
    restart: always
    volumes:
      - .:/koishi # /koishi would be empty
    ports:
      - 8080:5140
```

---

# Koishi Bootstrap

```yaml
# koishi.yml
plugins:
  adapter-onebot:
    $install: true
  novelai:
    $install: '1.0.0'

# docker-compose.yml
version: '2.4'
services:
  koishi:
    image: git-registry.mycard.moe:3rdeye/koishi-bootstrap
    restart: always
    volumes:
      - ./koishi.yml:/app/config.yml # that's enough
    ports:
      - 5140:5140
```

---

# Versioning in koishijs/koishi

- 镜像构建时，把 Koishi 本体放置在 /usr/src/koishi.tar.gz。
- 容器启动时 (entrypoint)：
  - /koishi/package.json 不存在，则从 /usr/src/koishi.tar.gz 解压。
  - /koishi/package.json 存在，则不进行处理。

```yaml
version: '2.4'
services:
  koishi:
    image: koishijs/koishi
    restart: always
    volumes:
      - ./koishi:/koishi
    ports:
      - 5140:5140
```

---

# END
