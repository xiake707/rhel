
# 一、任务规划

1. 准备一台虚拟机作为`dns`服务器
2. 挂载并编辑本地仓库
3. 安装`bind`域名管理安装包
4. 修改配置文件，将域名设置为`gqian.com`
5. 放行防火墙，重启服务
6. 测试

# 二、准备一台虚拟机做实验

## 被控机器的`ip`
```shell
[root@dnsserver ~]# ifconfig
enp1s0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.122.101  netmask 255.255.255.0  broadcast 192.168.122.255
        inet6 fe80::5054:ff:fee8:1ac6  prefixlen 64  scopeid 0x20<link>
        ether 52:54:00:e8:1a:c6  txqueuelen 1000  (Ethernet)
        RX packets 7153  bytes 408370 (398.7 KiB)
        RX errors 0  dropped 6152  overruns 0  frame 0
        TX packets 820  bytes 66910 (65.3 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

# 三、安装`bind`包

## 3.1. 控制节点`ansible`的结构

###  3.1.1. 项目文件结构

```shell
[root@server demo0]# tree
.
├── ansible.cfg
├── copy_yy.yml
├── file_yy.yml
└── inventory
    └── hosts

1 directory, 4 files

```
### 3.1.2. 主机文件

```shell
[root@server inventory]# cat hosts 
[client0]
192.168.122.18
[dnsserver]
192.168.122.101
[web:children]
client0
dnsserver
```
### 3.1.3. 配置文件 `ansible.cfg`

```shell
[root@server demo0]# cat ansible.cfg 
[defaults]
inventory=./inventory/hosts
```

## 3.2. 编写剧本文件

### 3.2.1. 挂载被控节点的仓库

>要用到mount模块

```yml
- name: 挂在/dev/sr0设备
      mount:
        src: /dev/sr0
        path: /mnt
        fstype: iso9660
        opts: defaults
        state: mounted

```

### 3.2.2. 配置仓库文件

>使用copy模块将本地local.repo文件copy到被控节点

```shell
[root@server yum.repos.d]# cat /etc/yum.repos.d/local.repo 
[baseos]
name=baseos
baseurl=/mnt/BaseOS
gpgcheck=0
enable=1
[appstream]
name=baseos
baseurl=/mnt/AppStream
gpgcheck=0
enable=1

```

```yml
- name: 配置仓库配置文件
      copy:
        src: /etc/yum.repos.d/local.repo
        dest: /etc/yum.repos.d/local.repo
        owner: root
        group: root
        mode: '0644'

```
### 3.2.3. 在`dnsserver`上安装`bind`包。

>要用到dnf模块，

**注意：** 这里使用ansible-doc  dnf 查询dnf模块的用法，详情请见[[Ansible#三、`ansible`中模块及其使用方法|ansible-doc 输出信息解读]]
```shell
EXAMPLES:

- name: Install the latest version of Apache
  ansible.builtin.dnf:
    name: httpd
    state: latest

```

> 稍作修改一下

```yml
- name: 安装bind软件
      dnf:
        name: bind
        state: latest
```

>写到这里时，如果使用`ansible-playbook dns.yml --check`永远回报错，报错的地方是dnf模块，因为前面的几个没有先决条件，而dnf必须要远程主机仓库配置好才行。但是ansible有幂等性，我们可以直接执行，但这在生产环境是不安全的，一定要三思。


```shell
[root@server demo0]# cat dns.yml 
- name: 远程配置dns服务器
  hosts: dnsserver
  become: yes
    
  tasks:
    - name: 将/dev/sr0设备挂载到/mnt
      mount:
        src: /dev/sr0
        path: /mnt
        fstype: iso9660
        opts: defaults
        state: mounted
    - name: 配置仓库配置文件
      copy:
        src: /etc/yum.repos.d/local.repo
        dest: /etc/yum.repos.d/local.repo
        owner: root
        group: root
        mode: '0644'
    - name: 安装bind软件
      dnf:
        name: bind
        state: latest
[root@server demo0]# ansible-playbook dns.yml 

PLAY [远程配置dns服务器] *************************************************************************************************************************

TASK [Gathering Facts] ***************************************************************************************************************************
ok: [192.168.122.101]

TASK [将/dev/sr0设备挂载到/mnt] ******************************************************************************************************************
changed: [192.168.122.101]

TASK [配置仓库配置文件] **************************************************************************************************************************
ok: [192.168.122.101]

TASK [安装bind软件] ******************************************************************************************************************************
changed: [192.168.122.101]

PLAY RECAP ***************************************************************************************************************************************
192.168.122.101            : ok=4    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

```
# 四、修改`named`的配置文件

#### 一个非常重要的模块
>做下面的实验，主要使用的是`copy`模块，这个模块有参数非常重要的参数`validate`
详情参见[[Ansible/提问日志/提问日志#3.|提问日志]]

**Ansible 的 validate 工作原理：**

1. 先**复制文件**到目标主机的临时位置
    
2. 然后**执行 validate 命令**检查这个临时文件
    
3. **只有 validate 成功**，才会用临时文件替换目标文件
    
4. 如果 validate 失败，不会替换原文件，任务失败
#### 修改目标主机的思路
将编写好的配置文件放到控制节点的file文件夹里
```shell
[root@server demo0]# tree file
file
├── gqian.zone
├── named.conf
├── named.rfc1912.zones
└── zone.gqian
```
让后使用`copy`模块将编写好的配置文件替换掉被控节点上的配置文件，就完成配置了 

## 4.1. 修改主配置文件`/etc/named.conf`

```yml
- name: 配置主配置文件
      copy:
        src: file/named.conf
        dest: /etc/named.conf
        owner: root
        group: named
        mode: '0640'
        validate: '/usr/sbin/named-checkconf %s'

```

## 4.2. 修改`/etc/named.rfc1912.zones`
```yml
- name: 配置区域配置文件
      copy:
        src: file/named.rfc1912.zones
        dest: /etc/named.rfc19212.zones
        owner: root
        group: named
        mode: '0640'
        validate: '/usr/sbin/named-checkconf %s'
```
## 4.3. 在目录`/var/named/`下添加区域解析文件
```yml 
- name: 配置区域文件
      copy:
        src: file/gqian.zone
        dest: /var/named/gqian.zone
        owner: root
        group: named
        mode: '0640'
        validate: '/usr/sbin/named-checkzone %s'

```

---
## 4.4. 最终形态
### 4.4.1. `playbook`的最终形态
```shell
[root@server demo0]# cat dns.yml 
- name: 远程配置dns服务器
  hosts: dnsserver
  become: yes
    
  tasks:
    - name: 将/dev/sr0设备挂载到/mnt
      mount:
        src: /dev/sr0
        path: /mnt
        fstype: iso9660
        opts: defaults
        state: mounted
    - name: 配置仓库配置文件
      copy:
        src: /etc/yum.repos.d/local.repo
        dest: /etc/yum.repos.d/local.repo
        owner: root
        group: root
        mode: '0644'
    - name: 安装bind软件
      dnf:
        name: bind
        state: latest
    - name: 配置主配置文件
      copy:
        src: file/named.conf
        dest: /etc/named.conf
        owner: root
        group: named
        mode: '0640'
        validate: '/usr/sbin/named-checkconf %s'
    - name: 配置区域配置文件
      copy:
        src: file/named.rfc1912.zones
        dest: /etc/named.rfc19212.zones
        owner: root
        group: named
        mode: '0640'
        validate: '/usr/sbin/named-checkconf %s'
    - name: 配置区域文件
      copy:
        src: file/gqian.zone
        dest: /var/named/gqian.zone
        owner: root
        group: named
        mode: '0640'
        validate: '/usr/sbin/named-checkzone gqian.com. %s'
```

### 4.4.2. 执行`ansible-playbook dns.yml`

```shell
[root@server demo0]# ansible-playbook dns.yml

PLAY [远程配置dns服务器] *************************************************************************************************************

TASK [Gathering Facts] ***************************************************************************************************************
ok: [192.168.122.101]

TASK [将/dev/sr0设备挂载到/mnt] ******************************************************************************************************
ok: [192.168.122.101]

TASK [配置仓库配置文件] **************************************************************************************************************
ok: [192.168.122.101]

TASK [安装bind软件] ******************************************************************************************************************
ok: [192.168.122.101]

TASK [配置主配置文件] ****************************************************************************************************************
changed: [192.168.122.101]

TASK [配置区域配置文件] **************************************************************************************************************
changed: [192.168.122.101]

TASK [配置区域文件] ******************************************************************************************************************
changed: [192.168.122.101]

PLAY RECAP ***************************************************************************************************************************
192.168.122.101            : ok=7    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```
# 五、启动服务，测试服务是否成功

## 5.1. 放行防火墙
> **dns服务需要放行的流量清单**
> 1. dns服务

```shell
 - name: 防火墙放行服务
      firewalld:
        service: dns
        permanent: yes
        state: enabled
        immediate: yes
```

## 5.2. 重新启动dns服务

```shell
 - name: 重新启动dns服务
      systemd:
        name: named
        state: restarted
        daemon_reload: yes
```

# 六、验证实验是否成功

## 6.1. 完整的`dns.yml`playbook内容
```yml
- name: 远程配置dns服务器
  hosts: dnsserver
  become: yes

  tasks:
    - name: 将/dev/sr0设备挂载到/mnt
      mount:
        src: /dev/sr0
        path: /mnt
        fstype: iso9660
        opts: defaults
        state: mounted
    - name: 配置仓库配置文件
      copy:
        src: /etc/yum.repos.d/local.repo
        dest: /etc/yum.repos.d/local.repo
        owner: root
        group: root
        mode: '0644'
    - name: 安装bind软件
      dnf:
        name: bind
        state: latest
    - name: 配置主配置文件
      copy:
        src: file/named.conf
        dest: /etc/named.conf
        owner: root
        group: named
        mode: '0640'
        validate: '/usr/sbin/named-checkconf %s'
    - name: 配置区域配置文件
      copy:
        src: file/named.rfc1912.zones
        dest: /etc/named.rfc1912.zones
        owner: root
        group: named
        mode: '0640'
        validate: '/usr/sbin/named-checkconf %s'
    - name: 配置区域文件
      copy:
        src: file/gqian.zone
        dest: /var/named/gqian.zone
        owner: root
        group: named
        mode: '0640'
        validate: '/usr/sbin/named-checkzone gqian.com. %s'
      copy:
        src: file/zone.gqian
        dest: /var/named/zone.gqian
        owner: root
        group: named
        mode: '0640'
        validate: '/usr/sbin/named-checkzone 122.168.192.in-addr.arpa %s'
    - name: 防火墙放行服务
      firewalld:
        service: dns
        permanent: yes
        state: enabled
        immediate: yes
    - name: 重新启动dns服务
      systemd:
        name: named
        state: restarted
        daemon_reload: yes
```
## 6.2. 执行剧本
```shell
[root@server demo0]# ansible-playbook dns.yml 
[WARNING]: While constructing a mapping from /root/demo0/dns.yml, line 40, column 7, found a duplicate dict key (copy). Using last
defined value only.

PLAY [远程配置dns服务器] *************************************************************************************************************

TASK [Gathering Facts] ***************************************************************************************************************
ok: [192.168.122.101]

TASK [将/dev/sr0设备挂载到/mnt] ******************************************************************************************************
ok: [192.168.122.101]

TASK [配置仓库配置文件] **************************************************************************************************************
ok: [192.168.122.101]

TASK [安装bind软件] ******************************************************************************************************************
ok: [192.168.122.101]

TASK [配置主配置文件] ****************************************************************************************************************
ok: [192.168.122.101]

TASK [配置区域配置文件] **************************************************************************************************************
ok: [192.168.122.101]

TASK [配置区域文件] ******************************************************************************************************************
changed: [192.168.122.101]

TASK [防火墙放行服务] ****************************************************************************************************************
ok: [192.168.122.101]

TASK [重新启动dns服务] ***************************************************************************************************************
changed: [192.168.122.101]

PLAY RECAP ***************************************************************************************************************************
192.168.122.101            : ok=9    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0 
```

## 6.3. 使用`dig`命令测试dns服务是否成功。

```shell
[root@server demo0]# dig @192.168.122.101 www.gqian.com A

; <<>> DiG 9.16.23-RH <<>> @192.168.122.101 www.gqian.com A
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 35119
;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: 30082c134184840001000000699ab89cf8e1c8cf736e54d9 (good)
;; QUESTION SECTION:
;www.gqian.com.			IN	A

;; ANSWER SECTION:
www.gqian.com.		86400	IN	A	192.168.122.18

;; Query time: 3 msec
;; SERVER: 192.168.122.101#53(192.168.122.101)
;; WHEN: Sun Feb 22 16:04:44 CST 2026
;; MSG SIZE  rcvd: 86
```


# 总结
### 1. 核心流程：四步循环法

在实验过程中，我们反复践行了高效的 Playbook 开发逻辑：

- **分析需求**：明确 DNS 部署涉及的挂载、配置仓库、安装软件、分发配置、防火墙放行及服务启动。
    
- **锁定模块**：通过功能匹配查找到 `mount`、`copy`、`dnf`、`firewalld` 和 `systemd` 模块。
    
- **自学文档**：利用 `ansible-doc` 快速获取参数（如 `dnf` 的 `state`，`firewalld` 的 `immediate`）。
    
- **编写剧本**：将零散的模块组合成具备逻辑先后顺序的 YAML 流程。
    

### 2. 技术亮点：安全与稳健性

- **配置校验 (Validate)**：这是本实验最大的亮点。通过 `copy` 模块的 `validate` 参数（如 `/usr/sbin/named-checkconf %s`），实现了“先校验、后替换”。这保证了即便配置文件有语法错误，也不会破坏远程主机的现有服务，是生产环境的**金标准**。
    
- **环境自愈**：剧本包含了从挂载光盘到配置本地 YUM 源的全过程，确保了在没有互联网环境的 RHEL 主机上也能实现自动化安装。
    
- **状态闭环**：通过 `firewalld` 放行流量和 `systemd` 重启服务，确保了部署完成后服务是立即对外可用的，而非死配置。
    

### 3. 经验总结与注意事项（避坑指南）

- **语法陷阱（重复键）**：在执行过程中出现的 `WARNING: found a duplicate dict key (copy)` 提示你：在同一个 Task 块中不能有两个并列的 `copy` 关键字。如果要拷贝多个文件，应分为两个 Task 或使用 `loop` 循环。
    
- **前置依赖**：实验中发现 `dnf` 模块在 `--check` 模式下会报错。这提醒我们：Ansible 剧本中的任务往往存在**强上下文依赖**（不配仓库就装不了包）。在复杂实验中，有时需要先确保基础环境任务执行成功。
    
- **权限管理**：在拷贝 DNS 区域文件时，你注意到了 `group: named` 和 `mode: '0640'`，这体现了对 RHEL 安全权限体系的深入理解，防止了 DNS 信息泄露。