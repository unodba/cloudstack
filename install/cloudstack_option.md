# 安装Usage服务（可选）
管理服务器配置完毕后，你可以选择安装Usage服务器。Usage服务器从系统事件中提取数据，便于对账户进行使用计费。

当存在多台管理服务器时，可以选择在它们上面安装任意数量的Usage服务器。Usage服务器会协调处理。考虑到高可用性，所以最少在两台管理服务器中安装Usage服务器。

## 安装Usage服务器的要求
- Usage服务器安装时，管理服务器必须在运行状态。
- Usage服务器必须与管理服务器安装在同一台服务器中。
## 安装Usage服务器的步骤

- RHEL/CentOS系统:

```bash
# yum install cloudstack-usage
```
- Debian/Ubuntu系统：

```bash
# apt-get install cloudstack-usage
```
安装成功后，使用如下命令启动Usage服务器。

```bash
# service cloudstack-usage start
```

## 开机自启动

- RHEL/CentOS系统:

```bash
# chkconfig cloudstack-usage on
```
- Debian/Ubuntu系统：

```bash
# update-rc.d cloudstack-usage defaults
```


# SSL (可选)
CloudStack默认提供HTTP的访问方式。有许多的技术和网站选择使用SSL。因此，我们先抛下CloudStack所使用的HTTP，假设站点使用SSL是一个典型实践。

CloudStack使用Tomcat作为服务容器。由于CloudStack网站中断SSL会话，可能会开启Tomcatde的SSL访问。Tomcat的SSL配置描述请查阅： http://tomcat.apache.org/tomcat-6.0-doc/ssl-howto.html.
