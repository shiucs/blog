---
title: 使用Docker搭建Mastodon
reprint: false
date: 2021-08-28 19:50:13
categories:
  - Code
tags:
  - Docker
cover:
coverWidth:
coverHeight:
---

## 为什么开始使用Docker搭建实例

开始使用Docker纯属一个手残意外导致的阴差阳错：我不小心在升级时按了DigitalOcean面板里的“关闭服务器”按钮，相当于跑着程序突然拔了主机电源，导致整个服务器出现大型罢工。最后由兔子帮（代）助（劳）我迁移了站点，方便起见部署在了Docker上。于是在此也提醒各位，想要关闭Mastodon服务，一定要使用`systemctl stop 具体服务（mastodon-sidekiq、mastodon-web以及mastodon-streaming）`的方法，千万不要硬关。

在使用了一段时间的Docker之后，我总结出以下Docker搭建的优缺点：

### 优点

1. 搭建、升级方便，命令简单，不用自行配置环境，比起用官方文档命令行搭建而言，更适合新手。
2. 对内存较小的服务器，可以免除每次升级需要的编译（precompile）步骤给小机器带来的负担。
3. Mastodon站点运行在一个隔离的小环境（容器）中，安全性较高，不怕新手操作把整个系统搞崩，出现问题之后只要重启即可，会按镜像自动复原。

### 缺点

1. 可用教程较少，许多命令需要重新学起。
2. 魔改字数等相对而言不太方便，需要增加一个步骤，在之后的博文会详细列出。
3. 需要学习docker相关命令。

总体而言，对于一个新手，docker维护起来相对还是比较方便（皮实耐操）的。大家可以自行决定。但搜索互联网，官方并没有给出docker的搭建指南，而民间指南要么过时、要么有冗余步骤会占用大量时间。因此，希望这篇教程能给大家带来一些帮助。

## 如何在Docker上从头搭建Mastodon

### 购买域名、购买服务器、配置SMTP服务

这三步为建站基础，请大家参考本站[最早教程](https://pullopen.github.io/基础搭建/2020/07/19/How-to-build-a-mastodon-instance.html)的前3步逐一完成。

其中，服务器可在任何一家服务器提供商购买，不必局限于（也不推荐）DigitalOcean。其他如国人常用的Vultur、Digital Ocean，或者位于德国的Contabo、Hetzner等都可以。你可以参考[主机测评网站](https://www.zhujiceping.com/)蹲服务器折扣。此外，[O3O站长搭站指南](https://www.notion.so/68b0e4bb0b9e4fa5a9dd74d03b71a328)有更详细的指导意见。

购买时选择操作系统为**Ubuntu或Debian**即可。本文教程均以此系统为准。

无论你选择哪家服务器，请都**不要使用位于国内的服务器**，最好**不要选择国内的服务器提供商**，否则你可能会面临无法与其他站点互联互通、甚至站点下线的风险。

Mastodon系统较为庞大，如果运行一个单人实例，请保证内存至少在1G以上，储存至少在25G以上。如果预计用户数量增加，请相应增加服务器配置。

### 配置系统

- 配置ssh-key：

  ```bash
  mkdir -p ~/.ssh
  nano ~/.ssh/authorized_keys
  ```

  将通过各种方法（如Xshell、PuTTy等软件）生成的ssh-rsa公钥粘贴入其中。随后通过ssh-key密钥方式登录。

  为了安全，官方推荐将ssh密码登录方式关闭（不影响通过VNC、DigitalOcean Console等方式登录，**请确保此时你的SSH是依靠密钥而不是密码登录，否则你设置完毕后会被踢出去**）：

  ```bash
  nano /etc/ssh/sshd_config
  ```

  找到`PasswordAuthentication`一行，将其前面的#删掉（取消注释），在后面将yes改成no。

  重启sshd：

  ```bash
  systemctl restart sshd
  ```

- 安装常用命令：

  ```bash
  apt update && apt install wget rsync python git curl vim git ufw -y
  ```

- 配置SWAP，具体请参考[配置SWAP教程](https://www.digitalocean.com/community/tutorials/how-to-add-swap-space-on-ubuntu-16-04)。

  请让你的内存+SWAP至少达到4G以上。可以在root用户下通过`free -h`查看。

- 配置防火墙

  ```bash
  sudo ufw allow OpenSSH
  sudo ufw enable
  ```

  打开防火墙，随后打开80和443端口：

  ```bash
  sudo ufw allow http
  sudo ufw allow https
  ```

  然后可以通过`sudo ufw status`检查防火墙状态，你应该会看到80和443端口的显示。　　

### 安装docker和docker-compose

注意：这里第一步使用了官方提供的一键脚本安装docker。如果你对此感到不放心，请通过[官网步骤](https://docs.docker.com/engine/install/ubuntu/)自行安装，同样也是复制粘贴命令行。

```bash
bash <(curl -L https://get.docker.com/)
sudo curl -L "https://github.com/docker/compose/releases/download/1.27.4/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

### 拉取Mastodon镜像（2022-04-25修改)

- 拉取镜像

  ```bash
  mkdir -p /home/mastodon/mastodon
  cd /home/mastodon/mastodon
  docker pull tootsuite/mastodon:latest     #如果需要升级到某稳定版本，请将latest改成v3.5.1等版本号。
  wget https://raw.githubusercontent.com/tootsuite/mastodon/master/docker-compose.yml
  ```

- 修改`docker-compose.yml`配置文件

  ```bash
  nano docker-compose.yml
  ```

  依次找到`web`、`streaming`、`sidekiq`分类，在每一类的`image: tootsuite/mastodon`后添加`:latest`或者你刚才拉取的版本号，变成`image: tootsuite/mastodon:latest`或`image: tootsuite/mastodon:v3.5.1`等等。

  此外，官方的`docker-compose.yml`文件里写的是7.17.4，但目前elasticsearch的最高版本为7.10.2，需要修改。

  `ctrl+X`退出保存。

### 初始化PostgreSQL（2022-04-25修改）

>（在v3.5.0以后，请注意Postgres文件夹所在位置有所修改，从原本的postgres改为了postgres14。本教程已更新。）

刚才`docker-compose.yml`文件中，数据库（db）部分的地址为`./postgres14:/var/lib/postgresql/data`，因此你的数据库绝对地址为`/home/mastodon/mastodon/postgres14`。

运行：

```bash
docker run --name postgres14 -v /home/mastodon/mastodon/postgres14:/var/lib/postgresql/data -e   POSTGRES_PASSWORD=设置数据库管理员密码 --rm -d postgres:14-alpine
```

执行完后，检查`/home/mastodon/mastodon/postgres14`，应该出现postgres相关的多个文件，不是空文件夹。

然后执行：

```bash
docker exec -it postgres14 psql -U postgres
```

输入：

```sql
CREATE USER mastodon WITH PASSWORD '数据库密码（最好和数据库管理员密码不一样）' CREATEDB;
```

创建mastodon用户。

```sql
\q
```

退出数据库。

最后停止docker：

```bash
docker stop postgres14
```

> 附：如果你参考了2020-12-12之前的教程，那个教程未对postgres设置密码，有一定的安全隐患，请参考本文新增附录看如何设置密码。　　

### 配置Mastodon（2022-11-21修改)

- 配置文件

  在`/home/mastodon/mastodon`文件夹中创建空白`.env.production`文件：

  ```bash
  touch .env.production
  ```

  root用户内，运行

  ```bash
  docker-compose run --rm web bundle exec rake mastodon:setup
  ```

  随后会出现下列问题（此处参考[此方更新版教程](https://tech.konata.co/2022-02-10-build-a-mastodon/)）：

  Domain name: 你的域名

  Single user mode disables registrations and redirects the landing page to your public profile.

  Do you want to enable single user mode? No

  Are you using Docker to run Mastodon? Yes

  PostgreSQL host: mastodon_db_1

  PostgreSQL port: 5432

  Name of PostgreSQL database: mastodon

  Name of PostgreSQL user: mastodon

  Password of PostgreSQL user: （这里写上面你给mastodon设置的数据库密码）

  Database configuration works! 🎆

  Redis host: mastodon_redis_1

  Redis port: 6379

  Redis password: （这里是直接回车，没有密码）

  Redis configuration works! 🎆

  Do you want to store uploaded files on the cloud? 这个我们先填No，未来再参考[上云教程](https://pullopen.github.io/站点维护/2020/07/22/Move-mastodon-media-to-Scaleway.html)配置。

  Do you want to send e-mails from localhost? No，然后根据刚才配置的邮件服务填写（下文为举例）。

  SMTP server: smtp.zoho.eu

  SMTP port: 587

  SMTP username: 你的zoho管理员邮箱地址

  SMTP password: 你的zoho管理员密码

  SMTP authentication: plain

  SMTP OpenSSL verify mode: none

  E-mail address to send e-mails “from”: 你的zoho管理员邮箱地址

  Send a test e-mail with this configuration right now? no

  This configuration will be written to .env.production

  Save configuration? Yes

  Below is your configuration, save it to an .env.production file outside Docker:

  然后会出现.env.production配置，**请务必复制下来，先存到电脑里，等会儿要用。**

  之后会要你建立数据库和编译，都选是。

  **更新：在4.0.2版本，因为一些bug，请先不要建立管理员帐号，稍后再使用tootctl工具建立！**

  一切成功之后，记得**立刻马上：**

  ```bash
  nano .env.production
  ```

  把你刚才复制下来的配置保存进去。

  **更新：在4.0.2版本，需要在`.env.production`中增加一行：**

  ```ini
  REDIS_URL=redis://@mastodon_redis_1:6379
  ```

- 启动Mastodon

  ```bash
  docker-compose down
  docker-compose up -d
  ```

- 为相应文件夹赋权

  ```bash
  chown 991:991 -R ./public
  chown -R 70:70 ./postgres14
  docker-compose down
  docker-compose up -d
  ```

### 安装并配置nginx（2022-11-21修改）

在这一步之前，请记得到您购买域名的网站（如NameCheap），在DNS设置中添加一个**A Record**，Host填写@（如果没有子域名需求），Value填写你服务器的IP地址，将你设定的域名指向你的服务器。

**请注意：此时你的DNS Setting里除了你刚才在邮箱配置和本步骤中亲自设置的内容之外，别的任何由域名商自动生成的内容请都删光。**

- 安装nginx

  ```bash
  sudo apt install nginx -y
  ```

- 配置nginx

  ```bash
  nano /etc/nginx/sites-available/你的域名
  ```

  网页打开[nginx模板](https://raw.githubusercontent.com/mastodon/mastodon/main/dist/nginx.conf)，将其中的example.com替换成自己域名，将20和43行的`/home/mastodon/live/public`改成`/home/mastodon/mastodon/public`，**更新：**并将`try_files $uri =404;`修改为`try_files $uri @proxy;`，复制到服务器中保存。

  随后配置镜像文件

  ```bash
  ln -s /etc/nginx/sites-available/你的域名 /etc/nginx/sites-enabled/
  ```

  `nginx -t`检查，无误后重启（如果提示ssl证书问题，请在`listen 443 ssl http2;`、`listen [::]:443 ssl http2;`等包含ssl的行前加#号先行注释掉，配置完ssl证书后再改回来）：

  ```bash
  systemctl reload nginx
  ```

- 配置SSL证书

  ```bash
  apt install snapd
  sudo snap install core; sudo snap refresh core
  sudo snap install --classic certbot
  sudo ln -s /snap/bin/certbot /usr/bin/certbot
  sudo certbot certonly --nginx -d 你的域名
  ```

- 重启nginx

  重新打开nginx配置文件，将`ssl_certificate`和`ssl_certificate_key`两行前的#号（以及你刚才添加的#号）删除。

  `nginx -t`检查是否有错误。

  如果没有错误，则可重启nginx：

  ```bash
  systemctl reload nginx
  ```

- 检查证书更新

  最后，可通过

  ```bash
  certbot renew --dry-run
  ```

  检查证书是否能自动更新。

如果不放心，可以再至`/home/mastodon/mastodon`文件夹，运行`docker-compose up -d`重启mastodon。静静等待几分钟后，点开你的域名，你的站点就上线啦！

> 更新：此时可通过tootctl工具建立管理员帐户

```bash
docker exec mastodon_web_1 tootctl accounts create USERNAME --email EMAIL --confirmed --role Owner　　
```

## 站点上线之后

在站点上线之后，你可以：

### 开启全文搜索

Docker的全文搜索开启十分方便，只需要：

```bash
cd /home/mastodon/mastodon
nano docker-compose.yml
```

编辑`docker-compose.yml`，去掉`es`部分前所有的#号，并且去掉`web`部分中`es`前面的#号。

`nano .env.production`编辑`.env.production`文件，加上

```ini
ES_ENABLED=true
ES_HOST=es
ES_PORT=9200
```

三行，重启：

```bash
docker-compose down
docker-compose up -d
```

待文件夹中出现elasticsearch文件夹后，赋权：

```bash
chown 1000:1000 -R elasticsearch
```

再次重启：

```bash
docker-compose down
docker-compose up -d
```

全文搜索即搭建完成。

然后

```bash
docker-compose run --rm web bin/tootctl search deploy
```

建立之前嘟文的搜索索引即可。

### 修改配置文件

如果在之后需要再对.env.production配置进行修改，只需：

```bash
cd /home/mastodon/mastodon
nano .env.production
```

进行相应修改，然后

```bash
docker-compose down
docker-compose up -d
```

重启即可。

### 使用管理命令行

在docker中使用[tootctl管理命令行](https://docs.joinmastodon.org/admin/tootctl/)的方式有三种：

- 进入docker系统后操作

  `docker ps`查看你的容器名字，如果你按照刚才设置，那你的容器名字一般为mastodon_web_1。

  ```bash
  cd /home/mastodon/mastodon
  docker exec -it mastodon_web_1 /bin/bash    #或者将“mastodon_web_1”替换为你的容器名。
  ```

  进入docker系统mastodon用户，然后在其中进行相应的tootctl操作。

  注：如果需要进入docker系统的root用户进行一些软件安装，则需输入`docker exec --user root -it mastodon_web_1 /bin/bash`。

- 在/home/mastodon/mastodon文件夹操作

  首先进入/home/mastodon/mastodon，然后

  ```bash
  docker-compose run --rm web bin/tootctl 具体命令
  ```

  进行操作。

- 在任意位置操作

  在任意位置：

  ```bash
  docker exec mastodon_web_1 tootctl 具体命令
  ```

  需要注意的是，这则具体命令需要包括所有必须的参数，并且如果命令本身会要求你进行后续输入，则无法完成（比如self-distruct命令无法通过该步骤完成。）

- 使用脚本简化命令

  可参考[这篇文章](https://blog.tantalum.life/posts/how-to-run-your-mastodon-by-docker/#使用alias脚本缩写tootctl命令)：

  ```bash
  cd /home/mastodon/mastodon
  nano tootctl.sh
  ```

  编辑脚本内容为：

  ```bash
  #!/bin/bash
  lpwd=$PWD
  mypath=`dirname $0`
  cd $mypath
  if [ $# -ge 1 ]
  then
      case $1 in 
      	"restart"
      		docker-compose restart
      	;;
      	"reload")
      		docker-compose down && docker-compose up -d
      	;;
  		"stop")
      		docker-compose down
      	;;
  		"start")
      		docker-compose up -d
      	;;
      	"psql")
      		docker-compose exec db $*
      	;;
      	*)
      		docker-compose run --rm web bin/tootctl $*
      	;;
  	esac
  else
  	echo "please use tootctl help for help"
  fi
  cd $lpwd
  ```

  随后执行：

  ```bash
  chmod +x /home/mastodon/mastodon/tootctl.sh
  echo "alias tootctl='/home/mastodon/mastodon/tootctl.sh' " >> ~/.bashrc 
  source ~/.bashrc
  ```

  之后tootctl相关命令均可在进入`/home/mastodon/mastodon/`后缩写为`tootctl xxx`，且数据库相关命令也可通过`tootctl psql`进入。

### 定时清理媒体文件

```bash
crontab -e
```

选择编辑器为nano或者你习惯的编辑器，随后输入：

```shell
20 03    * * 1   docker exec mastodon_web_1 tootctl media remove-orphans
20 04    * * 2   docker exec mastodon_web_1 tootctl media remove --days=7
20 05    * * 3   docker exec mastodon_web_1 tootctl preview_cards remove
```

具体清理时间可以自己设置（参考[Crontab教程](https://www.runoob.com/w3cnote/linux-crontab-tasks.html)，上述表示在**服务器时间**周一/二/三相应时间执行媒体清理命令。

### 升级

如果你要升级到最新版本，只需要：

```bash
cd /home/mastodon/mastodon
docker pull tootsuite/mastodon:latest     #或者将latest改成版本号如v3.2.1
```

如果你升级的是特定版本，则需要编辑docker-compose.yml，将web、streaming、sidekiq三部分的版本号改成相应版本。如果是latest则无需改动。

然后

```bash
docker-compose up -d
```

启动。

如果官方升级提示中包括其他步骤如`docker-compose run --rm web rails db:migrate`，则可在启动后进行。

在确认升级没问题之后，运行

```bash
docker system prune -a
```

清除旧的docker镜像文件。　　

### 如果在操作过程中出现了任何问题……

如果没有对站点进行过魔改，只要在docker系统外，通过

```bash
docker-compose down
docker-compose up -d
```

让系统通过docker镜像重新搭建容器即可。

### 利用Scaleway备份数据库

本步脚本由兔子写就，感谢ta！

首先，请注册Scaleway，申请token，创建Bucket，此三步可参考[Scaleway上云教程](https://pullopen.github.io/站点维护/2020/07/22/Move-mastodon-media-to-Scaleway.html)。

注意：下文提到的备份脚本会自动删除7天以前的文件，因此**请为备份数据库单独建立一个bucket，**不要和媒体文件使用同一个bucket！

在服务器中安装rclone和zip：

```bash
curl https://rclone.org/install.sh | sudo bash
apt install zip -y
```

创建rclone配置文件夹

```bash
mkdir -p  ~/.config/rclone/
```

新建配置文件：

```bash
nano ~/.config/rclone/rclone.conf
```

填入下列内容：

```ini
[scaleway]
type = s3
provider = Scaleway
access_key_id = 你的ACCESS KEY
secret_access_key = 你的SECRET KEY
region = nl-ams（根据你bucket选择的地区，法国fr-par，荷兰nl-ams，波兰pl-waw）
endpoint = s3.nl-ams.scw.cloud  （同上）
acl = private
```

保存。

然后创建脚本

```bash
nano /backup.sh
```

输入

```bash
#!/bin/bash
source /etc/profile
now=$(date "+%Y%m%d-%H%M%S")
origin="/home/mastodon/mastodon"
target="scaleway:你的bucket名字"
echo `date +"%Y-%m-%d %H:%M:%S"` " now starting export"
/usr/bin/docker exec pg容器名 pg_dump -U postgres -Fc mastodon_production > ${origin}/backup.dump &&
echo `date +"%Y-%m-%d %H:%M:%S"` " succeed and upload to s3 now"
/usr/bin/zip -P 密码 ${origin}/backup_${now}.zip ${origin}/backup.dump &&
/usr/bin/rclone copy ${origin}/backup_${now}.zip ${target} &&
echo `date +"%Y-%m-%d %H:%M:%S"` " ok all done"
rm -f ${origin}/backup.dump ${origin}/backup_${now}.zip
/usr/bin/rclone --min-age 7d  delete ${target}
```

pg容器名一般为mastodon_db_1（通过docker ps查看），密码为你设立的解压密码。mastodon_production为你的数据库名，可至`.env.production`查看。

保存。

赋权：

```bash
chmod 751 /backup.sh
```

然后，

```bash
/backup.sh
```

试运行一下，看看Scaleway中有没有zip文件生成。如果出现zip文件且大小以mb计算，则成功。

之后如果想通过备份文件恢复，则可参考[迁移教程](https://pullopen.github.io/站点维护/2020/10/21/migrate-Mastodon-to-Docker.html)。

随后，设置定时任务：

```bash
crontab -e
```

选择nano编辑器，

```shell
3 22    * * *   /backup.sh >> /backup.log
```

具体时间自己设置，建议设置在半夜，注意服务器时区（通过`date`查看服务器时间）。　　

### 利用Cloudflare提升网站速度

请注意，Cloudflare目前在大陆也时常被阻拦，未必能起到加速作用。是否能提高速度，需要根据服务器本身的在大陆的连接速度判断。

步骤如下：

到[Cloudflare官网](https://www.cloudflare.com/)注册账号，选择Add Website输入你自己的域名，Cloudflare会自动读取你的DNS记录并且要求你修改域名服务器（NameServer）。如果你是NameCheap上购买的域名，则点开你的域名，在Domain那一栏下面选择Custom DNS，填入Cloudflare提供的两个NameServer，静静等待生效即可。如果是其他地方购买的域名，Cloudflare也提供了相应[教程](https://support.cloudflare.com/hc/zh-cn/articles/205195708-将您的域名服务器更改为-Cloudflare)，通常非常简单就能完成。

![NameCheap设置](https://s1.ax1x.com/2020/07/22/U7oZrR.png)

生效之后会有邮件通知，这时打开Cloudflare，进入你的域名。

点开“速度/Speed - 优化”，下图Auto Minify的三个勾勾请勿选上。

![AutoMinify](https://s1.ax1x.com/2020/07/22/U7TYpF.png)

> 注意（2020-12-01更新）：Auto Minify在3.3.0版本之后会影响网页打开，请勿勾选！

同时**请注意**，**Rocket Loader™**这个开关，请务必确保它**关闭**，否则你的Mastodon会变成白屏。

![RocketLoader](https://s1.ax1x.com/2020/07/22/U7T79S.png)

打开**SSL设置**，将模式改成Full/完全：

![SSLSetting](https://s1.ax1x.com/2020/07/23/UXkujS.png)

现在请打开测试，您的网站速度有没有变快？每个人每个地区情况可能不同，对我而言确实有肉眼可见的速度提升，对于各位可能需要测试。如果速度反而变慢，可以直接在Cloudflare的DNS设定中将橘色的云朵按灰，即可取消代理。

更详细的设置可以参见 **[O3O搭站指南](https://guide.mastodon.im/cloudflare)**。

### 附：如何修改数据库密码（2022-04-25更新）

如果在2020-12-12前参考本教程，当时本教程未要求大家设置数据库密码，这样做有一定的安全风险。因此，如果你**已经建站完毕且未设置数据库密码**，请参考本段添加数据库密码。

关闭mastodon服务：

```bash
docker-compose down
```

`nano docker-compose.yml`修改，将之前教程要求你设置的

```yaml
environment:
  - POSTGRES_HOST_AUTH_METHOD=trust
```

删掉，保存。

`nano .env.production`修改，添加

```ini
DB_PASS=数据库密码
```

保存。

启动数据库并进入psql模式：

```bash
docker run --name postgres14 -v /home/mastodon/mastodon/postgres14:/var/lib/postgresql/data --rm  -d  postgres:14-alpine
docker exec -it postgres14 psql -U postgres
```

填入：

```sql
alter user mastodon with password '数据库密码';
alter user postgres with password '数据库管理员密码（两者最好不要相同）';
\q
```

停止数据库运行并启动mastodon：

```bash
docker stop postgres14
docker-compose up -d
```

数据库管理员密码和数据库密码即添加完毕。　　

## 总结

以上就是从头通过Docker搭建Mastodon的方法，通过单纯复制粘贴很快就能搭建出来。在升级过程中也能免去编译过程对小机器造成的负担。（当然，SWAP还是要开的！）如果使用官方分支，升级只要几分钟。

目前网络上一些其他的docker搭建指南都提到需要docker-compose build这一步骤，并无必要，且耗时长、对服务器压力大。

然后有朋友就要问了：我之前是按照DigitalOcean一键镜像搭建的/用官方文档命令行搭建的，怎么迁移到docker上去呢？如果我想对代码进行一点改动要怎么做呢？这些问题，我们将在后续的文章中谈到。
