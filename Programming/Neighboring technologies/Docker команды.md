> [!info] **Основная идея**  
> Docker CLI — незаменимый инструмент в повседневной разработке и отладке. Знание ключевых команд помогает быстро управлять контейнерами, образами и сетями без лишних тулов.

## Подробно

### 🧱 Работа с образами

**`docker images`** — список локальных образов.  
**`docker pull`** — загрузка образа с Docker Hub или другого реестра.  
**`docker build`** — сборка образа из текущей директории.  
**`docker rmi`** — удаление образа.  

---

### 🚀 Работа с контейнерами

**`docker run`** — запуск нового контейнера.  
**`docker ps`** — список активных контейнеров.  
**`docker ps -a`** — все контейнеры, включая остановленные.  
**`docker stop` / `start`** — остановка или запуск контейнера по ID или имени.  
**`docker rm`** — удаление остановленного контейнера.  
**`docker exec`** — выполнение команды внутри работающего контейнера.  
**`docker logs`** — просмотр stdout/stderr контейнера.  

---

### ⚙️ Docker Compose

**`docker compose up`** — запуск всех сервисов из `docker-compose.yml`.  
**`docker compose down`** — остановка и удаление контейнеров и сети.

---

### 🌐 Работа с сетью и томами

**`docker network ls`** — просмотр всех сетей. 
**`docker volume ls`** — просмотр всех томов.  

---

## Нюансы и подводные камни

> [!warning]
> - `docker rm` и `docker rmi` **не сработают**, если контейнер или образ используется.  
> - После `docker compose down` удаляются **все связанные ресурсы**, включая сети.  
> - Не забудь флаг `-it` при `docker exec`, иначе не получишь интерактивный shell.  
> - `docker build` без `.` в конце ничего не соберёт — это путь к `Dockerfile`.

---

## Полезные ссылки

- [CLI Cheat Sheet (PDF)](https://docs.docker.com/get-started/docker_cheatsheet.pdf?utm_source=chatgpt.com)
- [Docker CLI Reference](https://docs.docker.com/reference/cli/docker/?utm_source=chatgpt.com)
- [The Ultimate Docker Cheat Sheet](https://dockerlabs.collabnix.com/docker/cheatsheet/?utm_source=chatgpt.com)
