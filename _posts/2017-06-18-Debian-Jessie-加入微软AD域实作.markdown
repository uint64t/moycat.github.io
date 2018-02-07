---
layout: post
title: Debian Jessie(8.x) 加入微软AD域实作
date: 2017-06-18 20:27
categories: IT杂记
tags: [Debian, Linux, 域, 教程]
---
以前 Linux 虚拟机总是各自独立的管理账户，用起来麻烦至极。现在既然有了域也有了时间折腾，不如把手上的 Linux 服务器加进域里，使用 AD 域来统一认证。

参考了网上的一些资料，也踩了一些坑（还好有快照），终于配置成功了。在这里记上一笔。

我手上的 Linux 服务器都是 Debian Jessie 版本。**如果你要参照此教程，请善用快照，域那里不要紧，删了计算机账户就行，而 Linux 机器可能出现没有任何用户能够登录的情况。**

## 零、目标

 - 将 Debian Jessie 系统的机器加入微软的 AD 域中。
 - 实现 Debian 机器使用 AD 认证账号。
 - 共享 AD 域中用户的用户组信息，并能用于权限控制。

## 一、安装必要的依赖包

在 Shell 中以 <code>root</code> 权限执行以下命令：

```shell
aptitude install krb5-user krb5-config libpam-krb5 winbind libpam-winbind libnss-winbind
```

这将会安装加入域、用域认证所需的各种软件包。

 > **如果安装时就要求输入域和服务器，则域以全大写输入，服务器视情况输入。**

## 二、修改配置文件

在加入域之前，还需要对配置文件进行适当的修改。这里假设你要加入的域为 <code>domain.com</code>。

### 1.  修改 krb5 相关配置并测试

<code>krb5</code> 用于支持微软的 Kerberos 身份认证。

编辑 <code>/etc/krb5.conf</code> 文件，在其中对应的字段修改/加入以下内容（按实际情况修改域名）：

```
[libdefaults]
dns_lookup_realm = false
ticket_lifetime = 24h
renew_lifetime = 7d
forwardable = true
rdns = false
default_realm = DOMAIN.COM
dns_lookup_kdc = false

[realms]
DOMAIN.COM = {
kdc = dc.domain.com
admin_server = dc.domain.com
}

[domain_realm]
.domain.com = DOMAIN.COM
domain.com = DOMAIN.COM
```

 > **注意：文本框中大写部分必须保持全大写，否则将会出现不可预料的后果！**

使用 <code>kinit</code> 与 <code>klist</code> 来验证是否可以成功获得 ticket ：

```shell
root@Proxmox:~# kinit Administrator@DOMAIN.COM
Password for Administrator@DOMAIN.COM: 
root@Proxmox:~# klist
Ticket cache: FILE:/tmp/krb5cc_0
Default principal: Administrator@DOMAIN.COM

Valid starting     Expires            Service principal
mm/dd/yy hh:mm:ss  mm/dd/yy hh:mm:ss  krbtgt/DOMAIN.COM@DOMAIN.COM
    renew until mm/dd/yy hh:mm:ss
root@Proxmox:~#
```

 > **注意：文本框中大写部分还是必须保持全大写。**

如果正常，输入密码后 <code>kinit</code> 不应该返回任何结果，然后输入 <code>klist</code> 可以查看 ticket 信息。

如果这一步有问题，还是先去 debug 吧，如果再继续，多半最后**会导致没有任何用户能登录**……（这种情况建议进入**单用户模式**撤销操作）

### 2. 修改 Samba 配置并创建家目录

编辑 <code>/etc/samba/smb.conf</code> 文件，在其中对应的字段修改/加入以下内容（按实际情况修改域名）：

```
workgroup = DOMAIN
realm = DOMAIN.COM
security = ads
idmap config * : range = 16777216-33554431
template homedir = /home/DOMAIN/%U
template shell = /bin/bash
kerberos method = secrets only
winbind use default domain = true
winbind offline logon = true
```

 > **注意：大写部分保持，你懂的。**

然后，运行 <code>mkdir /home/DOMAIN</code> 命令建立所有域用户登录后的家目录根目录。注意，这里的目录需要与上文中 smb.conf 里 template homedir 的字段一致。

## 三、加入域并配置认证

### 1.  大胆加域

该加域了，是不是很一颗赛艇？

在 Shell 中运行以下加入域，并进行加域测试，其中 Administrator 可以为任意有加域权限的域用户：

```bash
net ads join DOMAIN.COM -U Administrator
net ads testjoin
```

 > **大写，你懂……**

正常情况下，回应应该如图所示。

```bash
root@Proxmox:~# net ads join DOMAIN.COM -U Administrator
Enter Administrator's password:
Using short domain name -- DOMAIN
Joined 'PROXMOX' to dns domain 'domain.com'
root@Proxmox:~# net ads testjoin
Join is OK
root@Proxmox:~#
```

重新启动服务：<code>service smbd restart &amp;&amp; service winbind restart</code>

### 2. 配置认证

PAM 可插拔认证模块，是一个看上去很叼，似乎也确凿很叼的东西。

编辑 <code>/etc/pam.d/common-account</code> 文件，在头部加入以下内容：

```
account sufficient      pam_winbind.so
account required        pam_unix.so
```

编辑 <code>/etc/pam.d/common-auth</code> 文件，在头部加入以下内容：

```
auth sufficient         pam_winbind.so
auth required           pam_deny.so
```

编辑 <code>/etc/pam.d/common-session</code> 文件，在头部加入以下内容：

```
session required        pam_unix.so
session required        pam_mkhomedir.so umask=0022 skel=/etc/skel
```

nssswitch 用于兼容一些程序。

编辑 <code>/etc/nsswitch.conf</code> 文件，将相应字段修改为以下内容：

```
passwd:         compat winbind
group:          compat winbind
shadow:         compat winbind
```

SSH 登录也需要配置，首先在 shell 中执行以下命令来建立白名单。

```
mkdir -p /etc/sshd
vim /etc/sshd/sshd.allow
```

在其中输入允许登录的 AD 域用户组即可，一行一个。

然后在 <code>/etc/pam.d/sshd</code> 文件头部加入以下内容：

```
auth required pam_listfile.so item=group sense=allow file=/etc/sshd/sshd.allow onerr=succeed
```

最后配置一下有权限执行 <code>sudo</code> 的用户组。编辑 <code>/etc/sudoers</code> 文件，在其中加入一行：

```
%sudoers   ALL=(ALL:ALL) ALL
```

其中 <code>sudoers</code> 是我专门在 AD 域中建立的一个用户组，用以给用户执行 <code>sudo</code> 的权限（其实我就是把 <code>Domain Admins</code> 甩进去了）。

你也可以加入：

```
%sudoers   ALL=(ALL) NOPASSWD: ALL
```

让用户无需输入密码即可使用 <code>sudo</code>。

至此，配置已经完成。你既可以用 <code>su Administrator</code> 来切换到域用户，也可以使用域账户本地登录、登录 SSH 了。

## 四、更复杂的访问控制

然而目前的情况，会不会让域中任意用户都能登录到服务器呢？不知道，没试过，其实我这是个新域没有非管理员账户也懒得试。不如直接加个访问控制，白名单制。

### 1. 修改 access.conf

修改 <code>/etc/security/access.conf</code> 文件，加入以下内容：

```
+ : sudoers : ALL
- : ALL : ALL
```

 > **注意：这将会禁止除了 <code>sudoers</code> 用户组以外所有用户登录！在下一步之前，请确认你已经配置好了用户组的相关认证，并将 <code>sudoers</code> 修改为你自己的管理员组。**

### 2. 在各个登录点启用这个模块。

access 模块默认不启用，所以需要手动配置。

在 <code>/etc/pam.d</code> 目录里，在对应的配置文件中，取消下列一行的注释，如果不存在则加入：

```
account  required     pam_access.so
```

我启用的配置文件为 <code>sshd</code>、<code>login</code>。

至此，一切都应该正常了。再也不用为 Windows 主机和 Linux 主机账户不统一犯强迫症了……

------------------------

 > **参考资料**
 >
 > - [Join Debian 8 to Active Directory][1]
 > - [Authenticating Linux With Active Directory][2]
 > - [access.conf(5) - Linux man page][3]

  [1]: http://www.installvirtual.com/join-debian-8-to-active-directory/
  [2]: https://wiki.debian.org/AuthenticatingLinuxWithActiveDirectory
  [3]: https://linux.die.net/man/5/access.conf