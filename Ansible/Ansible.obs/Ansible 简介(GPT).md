
# 好，来，咱们把 **Ansible** 这玩意儿从头到尾讲透一遍。  
我会按「你以后真要用它」的角度讲，而不是那种教科书式介绍。

---

# 一、Ansible 是干嘛的？

一句话：

> **Ansible 是一个“远程批量自动化运维工具”。**

它能干的事情包括：

- 批量安装软件
    
- 批量改配置文件
    
- 批量创建用户
    
- 批量重启服务
    
- 批量部署项目
    
- 批量上线集群
    
- 自动化发布
    
- 自动化巡检
    

你可以理解为：

> 它是一个“高级版 SSH 批量执行器”。

---

# 二、Ansible 的核心思想

它和传统自动化工具最大的区别是：

## ✅ 1️⃣ 无 Agent

它：

- 不需要在被控端安装客户端
    
- 不需要跑后台服务
    
- 不需要开额外端口
    

只需要：

```bash
ssh + python
```

就能工作。

这是它爆火的原因。

---

## ✅ 2️⃣ 基于 SSH 工作

控制端通过 SSH 连接目标机器，然后：

- 临时上传一个模块
    
- 在目标机执行
    
- 删除临时文件
    
- 返回结果
    

就像远程执行一样。

---

## ✅ 3️⃣ 声明式

你写的不是“怎么做”，而是“我要什么状态”。

例如：

```yaml
- name: 安装 nginx
  yum:
    name: nginx
    state: present
```

你没有写：

> 执行 yum install

你写的是：

> 保证 nginx 存在

这叫：

> 声明式自动化

---

# 三、Ansible 架构

Ansible 非常简单：

```
控制端
   |
   |  SSH
   v
被控端（多个）
```

核心组成：

| 组件        | 作用    |
| --------- | ----- |
| Inventory | 主机清单  |
| Module    | 功能模块  |
| Playbook  | 自动化脚本 |
| Role      | 模块化结构 |
| Plugin    | 扩展插件  |

---

# 四、Ansible 工作流程

当你执行：

```bash
ansible web -m ping
```

它会：

1. 读取 Inventory
    
2. 找到 web 组的机器
    
3. SSH 登录
    
4. 上传 ping 模块
    
5. 执行
    
6. 返回结果
    

---

# 五、Ansible 核心组成（重点）

## 1️⃣ Inventory（主机清单）

告诉 Ansible：

> 我要控制谁？

例子：

```ini
[web]
192.168.1.10
192.168.1.11

[db]
192.168.1.20
```

---

## 2️⃣ Module（模块）

Ansible 的灵魂。

常见模块：

|模块|功能|
|---|---|
|ping|测试连通|
|yum / apt|安装软件|
|copy|传文件|
|service|管理服务|
|user|管理用户|
|file|改权限|
|shell|执行命令|

你可以理解：

> 每个模块 = 一个功能插件

---

## 3️⃣ Playbook（剧本）

这是 Ansible 的核心玩法。

用 YAML 写。

例子：

```yaml
- hosts: web
  tasks:
    - name: 安装 nginx
      yum:
        name: nginx
        state: present

    - name: 启动 nginx
      service:
        name: nginx
        state: started
```

运行：

```bash
ansible-playbook test.yml
```

它就自动完成所有操作。

---

# 六、Ansible 的高级概念

---

## 🔥 1️⃣ 变量（Variables）

```yaml
vars:
  pkg_name: nginx
```

提高复用性。

---

## 🔥 2️⃣ 条件判断

```yaml
when: ansible_os_family == "RedHat"
```

---

## 🔥 3️⃣ 循环

```yaml
loop:
  - nginx
  - httpd
```

---

## 🔥 4️⃣ Handler

当配置改变时才重启服务：

```yaml
notify: restart nginx
```

---

## 🔥 5️⃣ Role（大型项目用）

标准目录结构：

```
roles/
  nginx/
    tasks/
    handlers/
    templates/
    vars/
```

企业级自动化必须用 Role。

---

# 七、Ansible 的优势

✔ 简单  
✔ 无 agent  
✔ 学习成本低  
✔ YAML 可读性强  
✔ 社区强大  
✔ 模块丰富

---

# 八、Ansible 的劣势

✖ 大规模机器时性能一般  
✖ 不适合秒级实时任务  
✖ YAML 容易缩进出错  
✖ 复杂逻辑写起来有点绕

---

# 九、Ansible 适用场景

非常适合：

- 批量服务器管理
    
- 自动化部署
    
- CI/CD
    
- 云服务器初始化
    
- K8S 节点批量部署
    

---

# 十、Ansible 和其他工具对比

|工具|特点|
|---|---|
|Ansible|无 agent，简单|
|SaltStack|实时性强|
|Puppet|重型，企业级|
|Chef|Ruby 编写|

Ansible 是最“亲民”的。

---

# 十一、Ansible 在云时代的地位

在：

- 云计算
    
- DevOps
    
- 自动化运维
    

中，Ansible 是：

> 标准技能

RHCE 甚至专门考 Ansible。

---

# 十二、学习路线建议（给你）

如果你真的想掌握它：

1. 学基础命令
    
2. 熟悉模块
    
3. 写简单 Playbook
    
4. 学变量和条件
    
5. 学 Role
    
6. 做一个完整部署项目
    

---

# 十三、总结一句话

Ansible 本质是：

> 用 YAML 描述服务器应该是什么样，然后让它自动变成那个样子。

---

如果你愿意，我可以：

- 给你画一个 Ansible 底层执行流程图
    
- 或者带你从 0 搭一个小实验环境
    
- 或者讲讲它和 SSH 原理结合的底层细节
    
- 或者讲讲它和 Kubernetes 的关系
    

你想往哪个方向深挖？ 😄