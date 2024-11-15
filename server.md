# Базовая настройка сервера:

## 1. создаем юзера

1. логин на сервер под рутом:
  `ssh root@<your_server_ip_address>`
2. добавить юзера, далее везде для примера с юзернеймом `vpsadmin`: `adduser vpsadmin` 
3. добавить юзера в судо группу `usermod -aG sudo vpsadmin`
4. тестим юзера:
  - `su - vpsadmin`
  - `sudo ls -la /root` - должно показать директорию
  
## 2. Заход по ключам, запрет по паролю рутом

1. локально на машине сгенерировать ключ `ssh-keygen -t ed25519 -C "название ключа"`
2. скопировать **публичный** ключ на сервер `ssh-copy-id -i ~/.ssh/filename.pub vpsadmin@ip`
3. для удобства добавить ~/.ssh/config:
	```
	Host <server_name>
		Hostname <IP>
		User vpsadmin
		Port 22
		IdentityFile ~/.ssh/filename
	```
    IdentityFile - путь к **приватному** ключу. 
    Далее на сервер можно будет заходить командой `ssh servername`
  
4. запрет на логин по паролям:
  - редактируем `sudo nano /etc/ssh/sshd_config`. Ставим `PasswordAuthentication` - `no`
  - `PermitRootLogin` - `no` - запрет логин рутом по SSH
  - рестартим ssh сервис     `sudo systemctl restart ssh`

## 3. настройка firewall
Базовые команды: 
 - `sudo ufw status` - статус файрволла
 - `sudo ufw allow 22` - открыть порт 22
 - `sudo ufw enable`  - включить, `sudo ufw disable` - выключить
 - `sudo ufw app list` - список разрешенных приложений
 - `sudo ufw allow OpenSSH` добавить ssh
 
 ## 4. установка панели.
 
 1. от рута 
    ```shell 
    bash <(curl -Ls https://raw.githubusercontent.com/mhsanaei/3x-ui/master/install.sh)
    ```
    Придумываем юзера, пароль, секретный путь до панели, порт. Не забываем сохранить
 2. Чтобы не заходить по http, пока не настроен https, прокидываем ssh тоннель:
 
    ```shell
    ssh -f -N -L localport:localhost:panelport <servername>
    ``` 
    servername берем из п3. выше. после этого панель будет доступна через 127.0.0.1:localport/secret_path

## 5. Установка докера

полный гайд [тут](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository)

1. удаление старых версий `for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get remove $pkg; done`
2. добавить репозитарий:
	```bash
	# Add Docker's official GPG key:
	sudo apt-get update
	sudo apt-get install ca-certificates curl
	sudo install -m 0755 -d /etc/apt/keyrings
	sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
	sudo chmod a+r /etc/apt/keyrings/docker.asc

	# Add the repository to Apt sources:
	echo \
	"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
	$(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
	
	sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
	sudo apt-get update
	```
3. установка: `sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin`
4. проверка `sudo docker run hello-world`

### Докер без рута:

1. `sudo groupadd docker`
2. `sudo usermod -aG docker $USER`
3. `newgrp docker`
4. проверка `docker run hello-world`
----
### Настройка автозапуска докера:
```bash
sudo systemctl enable docker.service
sudo systemctl enable containerd.service
```

### Логгирование докера:

Редактируем `sudo nano /etc/docker/daemon.json`
1. добавляем:
    ```json
    {
      "log-driver": "local",
      "log-opts": {
        "max-size": "10m"
      }
    }
    ```
2. перезапускаем
    ```bash
    sudo systemctl restart docker.service
    sudo systemctl restart containerd.service
    ```

3. проверяем `docker info --format '{{.LoggingDriver}}'` должно быть local

## Установка nginx
1. `sudo apt install nginx-full`

## Сертификаты через acme.sh используя cloudflare, т.к наш nginx будет слушать локалхост

1. Клонируем, устанавливаем, не забываем поправить емейл
    ```shell
    git clone https://github.com/acmesh-official/acme.sh.git
    cd ./acme.sh
    ./acme.sh --install -m my@example.com
    ```
2. Настроить ACME на let's encrypt сервер:
    ```shell
    acme.sh --set-default-ca --server letsencrypt
    ```
3. Получение токенов.

    Вам необходимо создать API токен, который:  
   1. имеет разрешения на редактирование одной конкретной DNS зоны; или
   2. имеет разрешения на редактирование нескольких DNS зон.

       Вы можете сделать это на странице вашего профиля Cloudflare, в разделе API Tokens. 
       При создании токена, в разделе Permissions выберите Zone > DNS > Edit, а в разделе Zone Resources укажите только те DNS зоны, в которых вам нужно выполнять задачи ACME DNS.

       API токен представляет собой строку длиной 40 символов, которая может содержать заглавные и строчные буквы, цифры и символы подчеркивания. Вам нужно передать этот токен acme.sh, установив переменную окружения CF_Token с его значением, например, выполните команду: 
       ```shell
       export CF_Token="3RglOjayEU6wM3qZ8xz_RcAB9rMuOWEcNkgQWxhA".
       ```
4. (i) Одна DNS зона.  
    Вы должны предоставить acme.sh идентификатор зоны (zone ID) той DNS зоны, которую необходимо редактировать.  
    Это 32-значная строка в шестнадцатеричном формате (например, 763eac4f1bcebd8b5c95e9fc50d010b4) и не должна путаться с именем зоны (например, example.com). Идентификатор зоны можно найти в панели управления Cloudflare на странице Overview (Обзор) зоны, в правой боковой панели.
    
    Чтобы передать эту информацию, установите переменную окружения CF_Zone_ID с идентификатором зоны, например:
    ```shell
    export CF_Zone_ID="763eac4f1bcebd8b5c95e9fc50d010b4".
    ```
   
5. выпустить сертификат. Далее в качестве примера будет домен `example.com`
    ```shell
   ./acme.sh --issue --dns dns_cf -d example.com
   ```
   
6. разрешить текущему юзеру ребутать nginx без рута и без запроса пароля sudo:
    ```shell
   sudo visudo sudo visudo -f /etc/sudoers.d/myusername
   ```
   добавить в конец файла:  
   `myusername ALL=(ALL) NOPASSWD: /usr/sbin/service nginx start,/usr/sbin/service nginx stop,/usr/sbin/service nginx restart,/usr/sbin/service nginx reload`  
    или `myusername ALL=(ALL) NOPASSWD: /usr/sbin/service nginx *` что даст выполнять все операции  nginx без пароля
7. создаем директорию для сертификатов: 
    ```shell
    mkdir -p ~/certificates/example.com
    ```
8. Устанавливаем сертификаты:
    ```shell
    acme.sh --install-cert -d example.com \
    --key-file ~/certificates/example.com/key.pem  \
    --fullchain-file ~/certificates/example.com/cert.pem \
    --reloadcmd "sudo service nginx force-reload"
    ```
9. Далее эти сертификаты пропишем в конфиг nginx и в панель. Для панели в panel settings:
 - public path `/home/vpsadmin/certificates/example.com/cert.pem`
 - Private Key Path `/home/vpsadmin/certificates/example.com/key.pem`

пример конфига nginx смотри в default.conf.  Подразумевается что уже есть контейнер с nginx в докере на котором запущен любой сайт, например твои любимые котики
   