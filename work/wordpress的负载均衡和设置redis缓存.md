
# 一、wordpress的负载均衡和配置redis缓存

## 1.1 环境准备

|IP 地址|主机名|环境|
| :----: | :----: | :----: |
|  192.168.12.51    | wp-1 | lnmp，worpress4.91， |
| 192.168.12.52     | wp-2 | lnp，worpress4.91 |
| 192.168.12.53     | proxy | nginx1.24,redis5 |


## 


---

## 1.2 安装nginx

---

在 /etc/yum.repos.d/ 下创建 nginx.repo ⽂件，添加**nginx**的软件库

```bash
[root@localhost ~]# vi /etc/yum.repos.d/nginx.repo
```

编辑**nginx.repo**加入以下内容

```ABAP
[nginx]
name = nginx repo
baseurl = https://nginx.org/packages/mainline/centos/7/$basearch/
gpgcheck = 0
enabled = 1
```

接下来安装**nginx**

```bash
[root@localhost ~]# yum install -y nginx 
```

打开默认配置文件进行编辑

```bash
[root@localhost ~]# vi /etc/nginx/conf.d/default.conf
```

找到**server**，将  “ { } ”括号里面的内容更换为下面的内容

```bash
server {
 listen 80;
 root /usr/share/nginx/html;
 server_name localhost;
 #charset koi8-r;
 #access_log /var/log/nginx/log/host.access.log main;
 #
 location / {
 index index.php index.html index.htm;
 }
 #error_page 404 /404.html;
 #redirect server error pages to the static page /50x.html
 #
 error_page 500 502 503 504 /50x.html;
 location = /50x.html {
 root /usr/share/nginx/html;
 }
 #pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
 #
 location ~ .php$ {
 fastcgi_pass 127.0.0.1:9000;
 fastcgi_index index.php;
 fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
 include fastcgi_params;
 }
}
```

启动**nginx**并设置为开机自启

```bash
[root@localhost ~]# systemctl start nginx
[root@localhost ~]# systemctl enable nginx
```

显示下面这个图片 说明Nginx服务配置成功



## 1.3 安装数据库

第二台机器不用安装 两台机共用一个
使用rpm查询是否已安装**Mariadb**，如果安装了用**yum**卸载掉，没有则不用卸载

```bash
[root@localhost ~]# rpm -qa | grep -i mariadb
[root@localhost ~]# yum -y remove 包名
```

  然后在**/etc/yum.repos.d/**    创建**Mariadb**

```bash
[root@localhost ~]# vi /etc/yum.repos.d/MariaDB.repo
```

打开编辑并写入以下内容，添加**Mariadb**的软件库

```bash
[mariadb]
name = MariaDB
baseurl = https://mirrors.cloud.tencent.com/mariadb/yum/10.4/centos7-amd64
gpgkey=https://mirrors.cloud.tencent.com/mariadb/yum/RPM-GPG-KEY-MariaDB
gpgcheck=1
```

安装，启动并设置为开机自启

```bash
[root@localhost ~]#  yum -y install MariaDB-client MariaDB-server
[root@localhost ~]#  systemctl start mariadb
[root@localhost ~]#  systemctl enable mariadb
```

好了之后在终端输入**mysql**  ，看看安装成功没 ，如果成功安装会进入到数据库，ctrl+c退出数据库

---



## 1.4 安装PHP

更新**yum**中**PHP**的软件源

```bash
rpm -Uvh https://mirrors.cloud.tencent.com/epel/epel-release-latest-7.noarch.rpm

rpm -Uvh https://mirror.webtatic.com/yum/el7/webtatic-release.rpm
```

安装**php7.2**所需要的包

```bash
yum -y install mod_php72w.x86_64 php72w-cli.x86_64 php72w-common.x86_64 php72w-mysqlnd php72w-fpm.x86_64
```

启动并设置为开机自启

```bash
[root@localhost ~]# systemctl start php-fpm
[root@localhost ~]# systemctl enable php-fpm
```

创建一个**PHP**安装测试的文件

```bash
[root@localhost ~]# echo "<?php phpinfo(); ?>" >> /usr/share/nginx/html/index.php
```


重启**Nginx**

```
systemctl restart nginx
```

---



## 1.5 搭建站点

---

在mysql服务器内创建⼀个名为**wordpress**的数据库并授权

进入数据库

```mysql
[root@localhost ~]# mysql -uroot -p000000
```

创建wordpress数据表；

```mysql
create database wordpress;
```



```mysql
grant all privileges on *.* to "wordpress"@"%" identified by 'Admin@123';
```

 

```mysql
 flush privileges;
```

完成之后输入 **exit;**  退出**mysql**



重启**mysql**服务

```mysql
[root@localhost ~]# systemctl restart mariadb
```

删除**nginx**的默认⽂件

```mysql
[root@localhost ~]# rm -rf /usr/share/nginx/html/*
```

下载并解压**wordpress**的安装包

```mysql
wget  https://cn.wordpress.org/wordpress-4.9.1-zh_CN.tar.gz
tar  -zxvf  wordpress-4.9.1-zh_CN.tar.gz

```

 把**wordpress**的文件全部复制到**nginx**的**html**下

```mysql
cp -rf wordpress/* /usr/share/nginx/html/
cd /usr/share/nginx/html/
```

复制⼀个配置⽂件，根据**mysql**授权填写配置⽂件

```bash
cp wp-config-sample.php wp-config.php
vim wp-config.php
```

**DB_NAME**是数据表 **DE_USER**是用户名 **DB_PASSWORD**是密码 **DB_HOST**是主机 **DB_ CHARSET **是文字编码 

```bash
# 两台机填写一样的
define('DB_NAME', 'wordpress');

/** MySQL数据库用户名 */
define('DB_USER', 'wordpress');

/** MySQL数据库密码 */
define('DB_PASSWORD', 'Admin@123');

/** MySQL主机 */
define('DB_HOST', '192.168.12.51');

/** 创建数据表时默认的文字编码 */
define('DB_CHARSET', 'utf8');

```

给予所有文件权限

```bash
[root@localhost html]# chmod -R 777 /usr/share/nginx/html/
```

<br/>



## 1.6 配置负载均衡 



请确保 wordpress配置成功 然后再来配置负载均衡
在三号机 下载 nginx     然后打开配置文件


```bash
cd /etc/nginx/conf.d/default.conf   
# 加入以下内容
 
upstream  blogs  {
         server  192.168.12.51;
         server  192.168.12.52;
        # ip_hash;
        # hash $request_uri consistent;
}
server {
     listen       80;
     location / {
      proxy_pass http://blogs;
                    }

}
```


重启 nginx并设定开机自启动

```bash
systemctl restart nginx && systemctl enable nginx
```


访问网页，http://192.168.12.51   进入后台 输入初始化设置的账号密码

> 点击设置-常规
>  修改URL 为负载均衡的IP 地址

![](https://img.oratun.cn/blog_wz/wp-1.png)


然后点击保存更改





测试负载均衡状态 









## 1.7 wordpress 配置redis缓存



### 1.7.1 安装redis

在第一台机安装 redis


```bash
yum  -y install  https://mirrors.tuna.tsinghua.edu.cn/remi/enterprise/remi-release-7.rpm
# 修改yum源
vim /etc/yum.repos.d/remi.repo
# 修改如下：
[remi]
name=Remi's RPM repository for Enterprise Linux 7 - $basearch
#baseurl=http://rpms.remirepo.net/enterprise/7/remi/$basearch/
#mirrorlist=https://rpms.remirepo.net/enterprise/7/remi/httpsmirror
mirrorlist=http://cdn.remirepo.net/enterprise/7/remi/mirror
enabled=1        # 这里原本默认是0的修改为1开启
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-remi

# 安装redis
yum  -y install redis-5*
```

### 1.7.2 安装php-redis连接驱动

现在redis安装好了，但是还是没法缓存数据，因为php还没有于redis建立联系

```bash
# 三台都要安装
yum -y install php-redis
# 出现依赖问题
yum -y install  --skip-broken  php-redis
yum -y install  --skip-broken  php72w*
```

### 1.7.3  安装redis插件


在wordpress后台，安装Redis Object Cache插件，并且启用，看到如果已连接，表示成功：

![](https://img.oratun.cn/blog_wz/wp-3.png)

![](https://img.oratun.cn/blog_wz/wp-2.png)


如果安装插件出现权限问题  做以下步骤

```bash
# 开启所有权限
cd /usr/share/nginx/
chmod 777 *
chown -R 777 *

cd html
chmod 777 *
chown -R 777 *

cd wp-content
chmod 777 *
chown -R 777 *

cd plugins
chmod 777 *
chown -R 777 *
```

```bash
# 修改 wp-config.php 配置文件 底部添加如下 两台都添加

define("FS_METHOD", "direct");
define("FS_CHMOD_DIR", 0777);
define("FS_CHMOD_FILE", 0777);


# 然后重启
systemctl restart nginx

define("FS_METHOD", "direct"); 这一行代码设置了文件系统的访问方式。"direct"表示直接访问文件系统，而不是使用 FTP 或 SSH 等方式。
define("FS_CHMOD_DIR", 0777); 这一行设置了文件夹权限。0777 表示文件夹权限为读、写、执行，对所有用户都可用。
define("FS_CHMOD_FILE", 0777); 这一行设置了文件权限。与文件夹权限相同，0777 表示文件权限为读、写、执行，对所有用户都可用。
```

> 修改完记得重启
> 这时就会发现可以下载安装并启用了

![](https://img.oratun.cn/blog_wz/wp-2.png)

启用之后如果显示插件文件不存在，把第一台机器 插件 scp 到第二台机器传输过去即可，设置权限的操作在第二台机器再执行一次。


```bash
cd /usr/share/nginx/html/wp-content
 scp -r redis-cache/ 192.168.12.52:/usr/share/nginx/html/wp-content/plugins/

systecmtl restart nginx
```

这时刷新网页就发现可以进去了


### 1.7.4 在web服务上指定redis服务器IP地址

```bash
vim /usr/share/nginx/html/wordpress/wp-content/plugins/redis-cache/includes/object-cache.php

$parameters = array(
     'scheme' => 'tcp',
     'host' => '192.168.12.51',   这里填的时redis的IP地址
     'port' => 6379,
     'timeout' => 5,
     'read_timeout' => 5,
     'retry_interval' => null
   );
```

### 1.7.5 查看缓存信息
```bash
redis-cli
127.0.0.1:6379> SELECT 0
OK
127.0.0.1:6379> keys *
 1) "wp:transient:plugin_slugs"
 2) "wp:options:notoptions"
 3) "wp:userlogins:admin"
 4) "wp:site-transient:available_translations"
 5) "wp:site-transient:update_plugins"
 6) "wp:options:nonce_key"
 7) "wp:default:is_blog_installed"
 8) "wp:redis-cache:metrics"
 9) "wp:site-transient:update_core"
10) "wp:options:nonce_salt"
11) "wp:users:1"
12) "wp:useremail:123123@qq.com"
13) "wp:options:logged_in_salt"
14) "wp:options:auth_key"
15) "wp:options:auth_salt"
16) "wp:options:can_compress_scripts"
17) "wp:userslugs:admin"
18) "wp:site-transient:update_themes"
19) "wp:user_meta:1"
20) "wp:options:alloptions"
21) "wp:options:logged_in_key"
```
![](https://img.oratun.cn/blog_wz/wp-4.png)

![](https://img.oratun.cn/blog_wz/wp-5.png)



