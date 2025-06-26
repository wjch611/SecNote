[TOC]

### 思路：

vulhub可以看作一个漏洞专属版的dockerhub。

1. 先去看dockerhub有没有环境，有则Dockerfile不需要写，也就是不需要编译镜像了
2. 自己从源码开始构建，成功后，再构建docker镜像

### 步骤：

###### 参考dockerhub:

1. 去dockerhub找到一个xxx/xxx:xxx的镜像
2. Dockerfile直接写FROM xxx/xxx:xxx
3. 然后docker-compose.yml看dockerhub上有没有，有就copy，里面的imgae还是使用原来的不要使用vulhub/xxx:xxx

###### 自己构建：

1. 搞清除这个环境需要什么：什么运行时、中间件、源码...，版本分别是什么，如果需要数据库那么可以直接在docker-compose.yml直接image对应版本的

2. 根据运行时的版本选择一个系统，参考：

   ```
   # 使用 Ubuntu 18.04 作为基础镜像（提供 PHP 7.2）
   FROM ubuntu:18.04
   
   # 设置非交互式环境和时区
   ARG DEBIAN_FRONTEND=noninteractive
   ENV TZ=Asia/Shanghai
   RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && \
       echo $TZ > /etc/timezone
   
   # 安装基础服务和PHP 7.2
   RUN apt-get update && \
       apt-get install -y --no-install-recommends \
           apache2 \
           ca-certificates \
           curl \
           unzip \
           libapache2-mod-php7.2 \
           php7.2 \
           php7.2-cli \
           php7.2-common \
           php7.2-mbstring \
           php7.2-mysql \
           php7.2-xml \
           php7.2-zip \
           && \
       rm -rf /var/lib/apt/lists/*
   
   # 配置Apache
   RUN a2dismod mpm_event && \
       a2enmod mpm_prefork rewrite
   
   # 设置工作目录和权限
   WORKDIR /var/www/html
   RUN chown -R www-data:www-data /var/www/html
   
   # 安装Emlog 2.1.9（使用官方旧版地址）
   ARG EMLOG_VERSION=2.1.9
   ARG EMLOG_URL=https://gitee.com/snowsun/emlog/releases/download/pro-${EMLOG_VERSION}/emlog_pro_${EMLOG_VERSION}.zip
   
   RUN curl -L ${EMLOG_URL} -o emlog.zip && \
       unzip -q emlog.zip -d /var/www/html/ && \
       rm emlog.zip && \
       chown -R www-data:www-data /var/www/html
   
   # Apache虚拟主机配置
   COPY apache-emlog.conf /etc/apache2/sites-available/000-default.conf
   
   # 暴露端口和启动命令
   EXPOSE 80
   CMD ["apache2ctl", "-D", "FOREGROUND"]
   ```

3. 如果需要apache，那么需要额外一个文件apache-emlog.conf

   ```
   <VirtualHost *:80>
       DocumentRoot /var/www/html
       <Directory /var/www/html>
           Options FollowSymLinks
           AllowOverride All
           Require all granted
       </Directory>
       ErrorLog ${APACHE_LOG_DIR}/error.log
       CustomLog ${APACHE_LOG_DIR}/access.log combined
   </VirtualHost>
   ```

4. 不断利用AI解决BUG，成功编译

   ```
   docker build -t xxx:xxx .
   ```

5. 编写docker-compose.yml

   ```
   version: '3'
   services:
     mysql:
       image: mysql/mysql-server:5.6
       container_name: mysql56
       command:
         - --default_authentication_plugin=mysql_native_password
         - --character-set-server=utf8mb4
         - --collation-server=utf8mb4_unicode_ci
       volumes:
         - ./db_data/mysql:/var/lib/mysql
       ports:
         - "3306:3306"
       restart: always
       environment:
         MYSQL_DATABASE: emlog
         MYSQL_USER: emlog
         MYSQL_PASSWORD: emlog
       networks:
         - emlog_network
     emlog:
       image: xxx:xxx
       container_name: emlog-pro
       restart: always
       environment:
         - EMLOG_DB_HOST=mysql
         - EMLOG_DB_NAME=emlog
         - EMLOG_DB_USER=emlog
         - EMLOG_DB_PASSWORD=emlog
         - EMLOG_DOMAIN_NAME=localhost
         - MAX_POST_BODY=50m
         - MAX_EXECUTION_TIME=300
       ports:
         - 80:80
       networks:
         - emlog_network
       volumes:
         - ./data:/app
       labels:
         createdBy: "Apps"
   networks:
     emlog_network:
       external: true
   ```

6. 成功启动，并且复现漏洞成功之后，可以上传PR了

   ```
   docker-compose up -d
   ```

### 参考：

参考：

36、github贡献项目流程

https://github.com/vulhub/vulhub/wiki/%E8%B4%A1%E7%8C%AE%E6%8C%87%E5%8D%97

### 细节：

1. 需要在base中创建Dockerfile（如果是从dockerhub中获取的就直接FROM它）
2. 需要在根目录创建：docker-compose.yaml（如果是从dockerhub中获取的就直接copy）
3. 需要修改environments.toml
4. 如果一个PR还没合并，又需要提交新的PR
   1. 退回master
   2. 然后重新创建新的分支，进入

### 贡献历史：

https://github.com/vulhub/vulhub/pull/703

https://github.com/vulhub/vulhub/pull/704
