阿里云服务器极速安装
------------------------

生产环境建议使用 `1.4.8 版本 <http://docs.jumpserver.org/zh/1.4.8/setup_by_alicloud.html>`_

**说明**

- 全新安装的 Centos7 系统
- 保持服务器网络畅通

**开始安装**

以下命令均在一个终端里面执行

.. code-block:: shell

    $ echo -e "\033[31m 1. 防火墙 Selinux 设置 \033[0m" \
      && if [ "$(systemctl status firewalld | grep running)" != "" ]; then firewall-cmd --zone=public --add-port=80/tcp --permanent; firewall-cmd --zone=public --add-port=2222/tcp --permanent; firewall-cmd --permanent --add-rich-rule="rule family="ipv4" source address="172.17.0.0/16" port protocol="tcp" port="8080" accept"; firewall-cmd --reload; fi \
      && if [ "$(getenforce)" != "Disabled" ]; then setsebool -P httpd_can_network_connect 1; fi

.. code-block:: shell

    $ echo -e "\033[31m 2. 部署环境 \033[0m" \
      && yum update -y \
      && yum -y install wget gcc epel-release git \
      && wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo \
      && yum clean all \
      && yum makecache \
      && yum install -y yum-utils device-mapper-persistent-data lvm2 \
      && yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo \
      && yum makecache fast \
      && rpm --import https://mirrors.aliyun.com/docker-ce/linux/centos/gpg \
      && echo -e "[nginx-stable]\nname=nginx stable repo\nbaseurl=http://nginx.org/packages/centos/\$releasever/\$basearch/\ngpgcheck=1\nenabled=1\ngpgkey=https://nginx.org/keys/nginx_signing.key" > /etc/yum.repos.d/nginx.repo \
      && rpm --import https://nginx.org/keys/nginx_signing.key \
      && yum -y install redis mariadb mariadb-devel mariadb-server MariaDB-shared nginx docker-ce \
      && systemctl enable redis mariadb nginx docker \
      && systemctl start redis mariadb \
      && yum -y install python36 python36-devel \
      && python3.6 -m venv /opt/py3

.. code-block:: shell

    $ echo -e "\033[31m 3. 下载组件 \033[0m" \
      && cd /opt \
      && if [ ! -d "/opt/jumpserver" ]; then git clone --depth=1 https://github.com/jumpserver/jumpserver.git; fi \
      && if [ ! -f "/opt/luna.tar.gz" ]; then wget https://demo.jumpserver.org/download/luna/1.5.3/luna.tar.gz; tar xf luna.tar.gz; chown -R root:root luna; fi \
      && yum -y install $(cat /opt/jumpserver/requirements/rpm_requirements.txt) \
      && echo -e "[easy_install]\nindex_url = https://mirrors.aliyun.com/pypi/simple/" > ~/.pydistutils.cfg \
      && source /opt/py3/bin/activate \
      && pip install --upgrade pip setuptools -i https://mirrors.aliyun.com/pypi/simple/ \
      && pip install -r /opt/jumpserver/requirements/requirements.txt -i https://mirrors.aliyun.com/pypi/simple/ \
      && curl -sSL https://get.daocloud.io/daotools/set_mirror.sh | sh -s http://f1361db2.m.daocloud.io \
      && systemctl restart docker \
      && docker pull jumpserver/jms_koko:1.5.3 \
      && docker pull jumpserver/jms_guacamole:1.5.3 \
      && rm -rf /etc/nginx/conf.d/default.conf \
      && wget -O /etc/nginx/conf.d/jumpserver.conf https://demo.jumpserver.org/download/nginx/conf.d/jumpserver.conf

.. code-block:: shell

    $ echo -e "\033[31m 4. 处理配置文件 \033[0m" \
      && source ~/.bashrc \
      && if [ "$DB_PASSWORD" = "" ]; then DB_PASSWORD=`cat /dev/urandom | tr -dc A-Za-z0-9 | head -c 24`; fi \
      && if [ "$SECRET_KEY" = "" ]; then SECRET_KEY=`cat /dev/urandom | tr -dc A-Za-z0-9 | head -c 50`; echo "SECRET_KEY=$SECRET_KEY" >> ~/.bashrc; fi \
      && if [ "$BOOTSTRAP_TOKEN" = "" ]; then BOOTSTRAP_TOKEN=`cat /dev/urandom | tr -dc A-Za-z0-9 | head -c 16`; echo "BOOTSTRAP_TOKEN=$BOOTSTRAP_TOKEN" >> ~/.bashrc; fi \
      && if [ "$Server_IP" = "" ]; then Server_IP=`ip addr | grep inet | egrep -v '(127.0.0.1|inet6|docker)' | awk '{print $2}' | tr -d "addr:" | head -n 1 | cut -d / -f1`; fi \
      && if [ ! -d "/var/lib/mysql/jumpserver" ]; then mysql -uroot -e "create database jumpserver default charset 'utf8';grant all on jumpserver.* to 'jumpserver'@'127.0.0.1' identified by '$DB_PASSWORD';flush privileges;"; fi \
      && if [ ! -f "/opt/jumpserver/config.yml" ]; then cp /opt/jumpserver/config_example.yml /opt/jumpserver/config.yml; sed -i "s/SECRET_KEY:/SECRET_KEY: $SECRET_KEY/g" /opt/jumpserver/config.yml; sed -i "s/BOOTSTRAP_TOKEN:/BOOTSTRAP_TOKEN: $BOOTSTRAP_TOKEN/g" /opt/jumpserver/config.yml; sed -i "s/# DEBUG: true/DEBUG: false/g" /opt/jumpserver/config.yml; sed -i "s/# LOG_LEVEL: DEBUG/LOG_LEVEL: ERROR/g" /opt/jumpserver/config.yml; sed -i "s/# SESSION_EXPIRE_AT_BROWSER_CLOSE: false/SESSION_EXPIRE_AT_BROWSER_CLOSE: true/g" /opt/jumpserver/config.yml; sed -i "s/DB_PASSWORD: /DB_PASSWORD: $DB_PASSWORD/g" /opt/jumpserver/config.yml; fi

.. code-block:: shell

    $ echo -e "\033[31m 5. 启动 Jumpserver \033[0m" \
      && systemctl start nginx \
      && cd /opt/jumpserver \
      && ./jms start -d \
      && docker run --name jms_koko -d -p 2222:2222 -p 127.0.0.1:5000:5000 -e CORE_HOST=http://$Server_IP:8080 -e BOOTSTRAP_TOKEN=$BOOTSTRAP_TOKEN --restart=always jumpserver/jms_koko:1.5.3 \
      && docker run --name jms_guacamole -d -p 127.0.0.1:8081:8080 -e JUMPSERVER_SERVER=http://$Server_IP:8080 -e BOOTSTRAP_TOKEN=$BOOTSTRAP_TOKEN --restart=always jumpserver/jms_guacamole:1.5.3 \
      && echo -e "\033[31m 你的数据库密码是 $DB_PASSWORD \033[0m" \
      && echo -e "\033[31m 你的SECRET_KEY是 $SECRET_KEY \033[0m" \
      && echo -e "\033[31m 你的BOOTSTRAP_TOKEN是 $BOOTSTRAP_TOKEN \033[0m" \
      && echo -e "\033[31m 你的服务器IP是 $Server_IP \033[0m" \
      && echo -e "\033[31m 请在安全组里面放行80和2222端口 \033[0m" \
      && echo -e "\033[31m 请打开浏览器访问公网80端口 用户名:admin 密码:admin \033[0m"

.. code-block:: shell

    $ echo -e "\033[31m 6. 配置自启 \033[0m" \
      && if [ ! -f "/usr/lib/systemd/system/jms.service" ]; then wget -O /usr/lib/systemd/system/jms.service https://demo.jumpserver.org/download/shell/centos/jms.service; chmod 755 /usr/lib/systemd/system/jms.service; systemctl enable jms; fi
