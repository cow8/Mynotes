# Deploy ShareLaTeX on local server

<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [Deploy ShareLaTeX on local server](#deploy-sharelatex-on-local-server)
    - [deployment procedure](#deployment-procedure)
        - [install docker](#install-docker)
        - [database](#database)
            - [install mongodb and redis in the system env](#install-mongodb-and-redis-in-the-system-env)
            - [check nginx ip inside the docker](#check-nginx-ip-inside-the-docker)
            - [config bind ip of database](#config-bind-ip-of-database)
        - [create docker container to load sharelatex image](#create-docker-container-to-load-sharelatex-image)
        - [create the first user](#create-the-first-user)
        - [start after reboot](#start-after-reboot)
    - [Tested functions](#tested-functions)
    - [Some useful link](#some-useful-link)

<!-- /code_chunk_output -->

## deployment procedure

### install docker

```{bash}
sudo apt install apt-transport-https ca-certificates software-properties-common curl
curl -fsSL https://mirrors.ustc.edu.cn/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://mirrors.ustc.edu.cn/docker-ce/linux/ubuntu \
$(lsb_release -cs) stable"
sudo apt update
sudo apt install docker-ce
sudo systemctl enable docker
sudo systemctl start docker
sudo docker run hello-world
```

### database

#### install mongodb and redis in the system env

```{bash}
sudo apt-get update
sudo apt-get install -y redis-server
sudo apt-get install -y mongodb
```

it can also be setuped through docker, but there will be more port mapping stuff.

#### check nginx ip inside the docker

`sudo docker run -d -p 80:80 nginx --name nginx` usually it will output that the inside ip is . And then we can `sudo docker stop nginx`

#### config bind ip of database

- `sudo vi /etc/mongodb.conf` and set to `bind_ip = 172.17.0.1`
- same as mongodb.conf: `sudo vi /etc/redis/redis.conf` and set `bind 172.17.0.1`
- restart service to use
    ```{bash}
    sudo service mongodb restart
    sudo service redis-server restart
    ```

### create docker container to load sharelatex image

```{bash}
sudo docker run -d \ 
    -v ~/sharelatex_data:/var/lib/sharelatex \ 
    -p 80:80 \ 
    --name=sharelatex \ 
    --env SHARELATEX_SITE_URL=$domain \ 
    --env SHARELATEX_APP_NAME=LaTeX \
    xuio/sharelatex-docker-image-full
```
The code above is the config i used, and there are some other options as follow:

---
- to use original image: replace `xuio/sharelatex-docker-image-full` with `sharelatex/sharelatex`
- to setup email: set flag as: https://github.com/sharelatex/sharelatex/wiki/Configuring-SMTP-Email
- 
---
### create the first user

```
$MY_EMAIL=test@test.test
sudo docker exec sharelatex /bin/bash -c "cd /var/www/sharelatex/web; grunt create-admin --email $MY_EMAIL"
```

### start after reboot

`vim startup.sh`
and input:

```{bash}
sleep 10s
service mongodb start
sleep 10s
service redis-server start
sleep 20s
docker start sharelatex
```

## Tested functions

- add normal user
- a user invite other collaborators
- visit through user-defined domain

## Some useful link

- https://gist.github.com/jbeyerstedt/25aa90cd9a8cfd7476618daca81c9c58
- https://github.com/sharelatex/sharelatex/wiki/Quick-Start-Guide
- https://github.com/sharelatex/sharelatex-docker-image/issues/90