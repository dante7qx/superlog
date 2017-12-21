## Ansible

### 一. 前言

​	自动化运维。随着信息时代的持续发展，IT运维已经成为IT服务内涵中重要的组成部分。面对越来越复杂的业务，面对越来越多样化的用户需求，不断扩展的IT应用需要越来越合理的模式来保障IT服务能灵活便捷、安全稳定地持续保障，这种模式中的保障因素就是IT运维（其他因素是更加优越的IT架构等）。从初期的几台服务器发展到庞大的数据中心，单靠人工已经无法满足在技术、业务、管理等方面的要求，那么标准化、自动化、架构优化、过程优化等降低IT服务成本的因素越来越被人们所重视。其中，自动化最开始作为代替人工操作为出发点的诉求被广泛研究和应用。自动化作为其重要属性之一已经不仅仅只是代替人工操作，更重要的是深层探知和全局分析，关注的是在当前条件下如何实现性能与服务最优化，同时保障投资收益最大化。

​	自动化运维工具：puppet、cfengine、chef、func、fabric、**ansible**。

### 二. 概念

​	ansible是新出现的自动化运维工具，基于Python开发，集合了众多运维工具（puppet、cfengine、chef、func、fabric）的优点，实现了批量系统配置、批量程序部署、批量运行命令等功能。ansible是基于**模块**工作的，本身没有批量部署的能力。真正具有批量部署的是ansible所运行的模块，ansible只是提供一种框架。主要包括：

(1)、连接插件connection plugins：负责和被监控端实现通信（**ssh**）；

(2)、host inventory：指定操作的主机，是一个配置文件里面定义监控的主机；

(3)、各种模块核心模块、command模块、自定义模块；

(4)、借助于插件完成记录日志邮件等功能；

(5)、**playbook**：剧本执行多个任务时，非必需可以让节点一次性运行多个任务。**（include、role）**

**架构图：**

![架构](/Users/dante/Documents/Technique/且行且记/Ansible/架构图.jpg)

```properties
Connection Plugins: 连接插件，Ansible和Host通信使用，Hosts 定义在 Host Inventory 中
Host Invertory: 记录了每一个由Ansible管理的主机信息，信息包括ssh端口，root帐号密码，ip地址等等。可以通过file来加载，可以通过CMDB (https://github.com/roncoo/roncoo-cmdb) 加载
Core Modules: Ansible管理主机之前，先调用core Modules中的模块，然后指明管理Host Lnventory中的主机，就可以完成管理主机。
Custom Modules: 自定义模块，完成Ansible核心模块无法完成的功能，此模块支持任何语言编写。
Playbooks: YAML格式文件，多个任务定义在一个文件中，使用时可以统一调用，“剧本”用来定义那些主机需要调用那些模块来完成的功能.
```

**执行过程：**

 暖色调的代表已经模块化

![执行过程](/Users/dante/Documents/Technique/且行且记/Ansible/执行过程.jpg)

### 三. 安装

参考：https://docs.ansible.com/ansible/latest/intro_installation.html

### 四. 实战操作

#### 1. SSH免密登录

##### a、方式1 

- 生成密钥 `ssh-keygen -t rsa -N 1@wq#ew -f ~/.ssh/id_rsa`
  - authorized_keys  存放远程免密登录的公钥,主要通过这个文件记录多台机器的公钥
  - id_rsa  私钥文件
  - id_rsa.pub  公钥文件
  - know_hosts  已知的主机公钥清单
  - 满足条件
    - .ssh目录的权限必须是700
    - .ssh/authorized_keys文件权限必须是600
- 复制公钥到远程主机 `ssh-copy-id -i ~/.ssh/id_rsa.pub 10.71.202.121 `

##### b、方式2

- 编辑 /etc/ansible/hosts

  ```ini
  [no-pass-login]
  10.71.202.121
  10.71.202.122
  ```

- 执行命令

  ```shell
  ansible no-pass-login -m authorized_key -a "user=root key='{{ lookup('file','/root/.ssh/id_rsa.pub') }}'" -k
  ```

#### 2. Inventory

默认配置在 /etc/ansible/hosts，指定源 `ansible -i /etc/ansible/test_host <host-pattern_1:[host-pattern_2]> -m setup`，多个 host-pattern 之间用 “,” 分隔。

```ini
##
# 组的变量，定义在文件中
#
# /etc/ansible/group_vars/<组名>/变量文件，组下的主机都能使用
# /etc/ansible/host_vars/<主机名>/变量文件
# 
##

[local]
127.0.0.1 	ansible_connection=local

[amp-server:children]
amp-web
amp-db

[amp-web:children]
amp-sysmgr
amp-airline-design

[amp-db]
10.71.88.214

[amp-sysmgr]
10.71.202.121

[amp-airline-design]
10.71.202.122
```

参数说明

```shell
ansible helloworld -m command -a 'echo {{ansible_ssh_host}} - {{ansible_ssh_user}} -  {{ansible_ssh_private_key_file}}'

10.71.202.121 | SUCCESS | rc=0 >>
10.71.202.121 - root - /Users/dante/.ssh/id_rsa

10.71.202.122 | SUCCESS | rc=0 >>
10.71.202.122 - root - /Users/dante/.ssh/id_rsa
```



#### 3. 模块（module）

通过   `ansible-doc -s <command_name>` 获取模块信息，

语法：`ansible <host_pattern> -m <module_name> -a <arguments>`

- ##### **setup**：`ansible helloworld -m setup`

- **ping**：`ansible helloworld -m ping`


- **command**：不能通过 shell 执行，`"<"`, `">"`, `"|"`, `";"` and `"&"` 不能使用。
- **shell**：同 command

  ```shell
  ## chdir: 进入目录
  ## creates：文件存在时，不执行命令
  ## removes：文件不存在时，不执行命令
  ansible helloworld -m command -a 'chdir=/home creates=spirit  ls -l'
  ansible helloworld -m shell -a 'chdir=/home creates=spirit  ls -l > /tmp/a.txt'
  ```

- **file**：操作文件、链接、目录的创建和删除，相同功能的模块如下

   http://docs.ansible.com/ansible/latest/file_module.html

- **copy**： http://docs.ansible.com/ansible/latest/copy_module.html#copy

- **template**：

  - **state**
    - file：文件不存在，不会创建
    - touch：创建、更新
    - directory:  目录不存在时创建
    - link：创建软链接，需要 src 和 dest
    - hard：hardlinks 创建、更新
    - absent：删除目录、文件或者取消链接文件

```powershell
## file
ansible helloworld -m file -a 'path=/tmp/xx mode=0666 state=touch'
ansible helloworld -m file -a 'src=/tmp/xx dest=/tmp/xx.lnk state=link owner=root group=root'
ansible helloworld -m file -a 'path=/tmp/xx state=absent'
ansible helloworld -m file -a 'path=/tmp/xx.lnk state=absent'

## copy
ansible helloworld -m copy -a 'src=/tmp/xx dest=/tmp/yy backup=yes'
```

- **stat**：获取远程主机文件状态信息

```powershell
ansible helloworld -m stat -a 'path=/tmp/file1'
```

- **get_url**：在远程主机下载指定url到远程主机上

```powershell
ansible helloworld -m get_url -a 'url=http://www.baidu.com dest=/tmp/index.html'
```

- **cron**：远程主机定时任务
  - http://docs.ansible.com/ansible/latest/cron_module.html
  - http://www.cnblogs.com/juandx/archive/2015/11/24/4992465.html

```powershell
ansible helloworld -m cron -a 'name="test ansible cron" minute=*/1 job="ls /home > /home/cc"'

ansible helloworld -m cron -a 'name="test ansible cron" state=absent'
```

- **service**：管理远程主机的服务

```powershell
ansible helloworld -m service -a 'name=httpd state=started'
ansible helloworld -m service -a 'name=httpd state=stopped'
ansible helloworld -m service -a 'name=httpd state=restarted'
ansible helloworld -m service -a 'name=httpd state=reloaded'
```

- **firewalled**：管理远程主机的防火墙端口

  - http://docs.ansible.com/ansible/latest/firewalld_module.html


  - http://jingyan.baidu.com/article/92255446580010851648f42e.html

```powershell
ansible helloworld -m firewalld -a 'port=8082/tcp permanent=true state=enabled immediate=yes'
```

- **yum**：使用yum包管理器对远程主机进行安装、卸载软件包
  - http://docs.ansible.com/ansible/latest/yum_module.html

```powershell
ansible helloworld -m yum -a 'name=jq state=latest'
```

#### 4. 变量 variable

定义方法：vars、vars_files、vars_prompt

- vars

```yaml
## 定义变量
foo:
  field1: one
  field2: two
## 使用，推荐使用[]
foo['field1']
foo.field1
```

- vars_files

```yaml
- hosts: all
  remote_user: root
  vars:
    favcolor: blue
  vars_files:
    - /vars/external_vars.yml
    - /vars/nginx_vars.yml
```

- 内置变量

  - **hostvars**： 获取某台指定的主机的相关变量

    ```json
    {{ hostvars['db.example.com'].ansible_eth0.ipv4.address }}
    // db.example.com不能使用ip地址来取代，只能使用主机名或别名
    ```

  - **inventory_hostname与inventory_hostname_short**

    inventory_hostname是Ansible所识别的当前正在运行task的主机的主机名

    ```shell
    ansible all -m shell -a 'echo {{inventory_hostname}}'

    localhost | SUCCESS | rc=0 >>
    localhost

    10.71.202.121 | SUCCESS | rc=0 >>
    10.71.202.121

    10.71.202.122 | SUCCESS | rc=0 >>
    10.71.202.122

    ansible all -m shell -a 'echo {{inventory_hostname_short}}'
    localhost | SUCCESS | rc=0 >>
    localhost

    10.71.202.121 | SUCCESS | rc=0 >>
    10

    10.71.202.122 | SUCCESS | rc=0 >>
    10
    ```

  - group_names：用于标识当前正在执行task的目标主机位于的主机组

  - groups：当你想要访问一组主机的变量时，groups变量会很有用

    ```yaml
    ### 在所有的dbservers组的服务器上创建一个数据库用户kate
    - name: Create a user for all db servers
      mysql_user: name=kate password=test host={{ hostvars.[item].ansible_eth0.ipv4.address }} state=present
      with_items: groups['dbservers'] 
    ```

    ​

- Jinja2：

- 优先级

  ```properties
  1、extra vars(命令中-e)最优先
  2、inventory 主机清单中连接变量(ansible_ssh_user 等)
  3、play 中 vars、vars_files 等
  4、剩余的在 inventory 中定义的变量
  5、系统的 facts 变量
  6、角色定义的默认变量(roles/rolesname/defaults/main.yml)

  子组会覆盖父组，主机总是覆盖组定义的变量
  ```

#### 5. playbook

​	由一个或多个 “play” 组成列表。play的主要功能在于将事先归并为一组的主机装扮成事先通过ansible中的task定义好的角色。从根本上来讲所谓task无非是调用ansible的一个module。将多个play组织在一个playbook中即可以让它们联同起来按事先编排的机制同唱一台大戏。

- **Target section**： 定义将要执行 playbook 的远程主机组
- **Variable section**：定义 playbook 运行时需要使用的变量
- **Task section**： 定义将要在远程主机上执行的任务列表
- **Handler section**：定义 task 执行完成以后需要调用的任务

基本命令：

```powershell
# 检查palybook语法
ansible-playbook -i hosts httpd.yml --syntax-check

# 列出要执行的主机
ansible-playbook -i hosts httpd.yml --list-hosts

# 列出要执行的任务
ansible-playbook -i hosts httpd.yml --list-tasks

## 参数说明
## become_user不是root时，执行模块会被写入到/tmp下的一个随机文件中，当命令执行后会被立即删除
become: yes
become_method: sudo
become_user: root

# 执行命令前，先执行setup命令来获取远程主机信息
gather_facts: yes 
```

```yaml
### Apache Ansible Install
---
- hosts: apache
  vars: 
    http_port: 80
    max_clients: 200
  remote_user: root
  tasks:
  - name: Ansible install apache
    yum: name=httpd state=latest
    notify: 
    - restart apache
  - name: ensure apache is running (and enable it at boot)
    service: 
      name: httpd 
      state: started 
      enabled: yes
  - name: open firewall for port 80
    firewalld: port=80/tcp permanent=true state=enabled immediate=yes
  handlers:
    - name: restart apache
      service: 
        name: httpd 
        state: stopped
```

##### Include

将一个大的playbook文件拆分成小的文件，通过 include 来包含引用。

```yaml
---
- hosts: helloworld
  vars_files: 
    - var1.yml
    - var2.yml
  remote_user: root
  tasks: 
    - include: task1.yml
    - include: task2.yml
  handlers:
    - include: handler.yml
- include: playbook1.yml
```

##### Role

基于已知文件结构自动加载 vars_files、tasks、handlers。按角色分组内容还允许轻松与其他用户共享角色。

![role](/Users/dante/Documents/Technique/且行且记/Ansible/role.png)

- site.yml：主要的playbook
- hosts：主机 Inventory
- group_vars
  - all：其中列出的变量将被添加到所有的主机组中
  - groupname1：列出的变量将被添加到groupname1主机组中
- host_vars
  - hostname1：列出的变量将被添加到hostname1主机组中
- roles/*/**/main.yml：tasks、handles、variables 都将被添加到play中
- roles/x/files/：copy tasks不需要指明文件的路径，可以直接引用
- roles/x/templates/：template tasks不需要指明文件的路径，可以直接引用
- roles/x/tasks/：include tasks 不需要指明文件的路径，可以直接引用
- 如果 roles 目录下有文件不存在，这些文件将被忽略

**加载顺序**

1. meta/main.yml
2. tasks/main.yml
3. handlers/main.yml
4. vars/main.yml
5. defaults/main.yml

**执行角色任务前后执行的动作**

```yaml
---
- hosts: webservers

  pre_tasks:
    - shell: echo 'hello'

  roles:
    - { role: some_role }

  tasks:
    - shell: echo 'still busy'

  post_tasks:
    - shell: echo 'goodbye'
```

**角色依赖**

角色依赖关系存储在角色目录中包含的meta/main.yml文件中

```yaml
### 角色依赖性始终在包含角色的角色之前执行，并且是递归的。
### 按照common，apache，postgres，'/path/to/common/roles/foo' 顺序依次执行，再执行此角色

---
dependencies:
  - { role: common, some_parameter: 3 }
  - { role: apache, apache_port: 80 }
  - { role: postgres, dbname: blarg, other_parameter: 12 }
  - { role: '/path/to/common/roles/foo', x: 1 }
```

**示例**：[microservice-spirit](/Users/dante/Documents/Technique/Ansible/microservice-spirit)

#### 6. 判断和循环

##### when

```yaml
hosts: all
  tasks:
    - name: when and loop test
      command: echo {{item}}
      with_items: [0, 2, 4, 6, 8, 10]
      when: item > 5
```

##### loop

```yaml
---
- name: Loop Hash
  hosts: local
  vars_files: 
    - user_hash.yml

  tasks: 
    - name: print phone records
      debug:
        msg: "User {{ item.key }} is {{ item.value.name }} ({{ item.value.telephone }})"
      with_dict: "{{ users }}"
      
    - name: loop file
      debug: msg="{{ item }}"
      with_file: 
        - file1
        - file2
        
---
users:
  alice:
    name: Alice Appleworth
    telephone: 123-456-7890
  bob:
    name: Bob Bananarama
    telephone: 987-654-3210
```



### 七. 名词解释

```properties
ad-hoc: ansible 命令行
```





### 八. 参考资料

- http://docs.ansible.com/


- http://sofar.blog.51cto.com/353572/1579894
- http://www.jianshu.com/p/8b17779febf3?utm_campaign=maleskine&utm_content=note&utm_medium=pc_all_hots&utm_source=recommendation
- http://www.jianshu.com/p/f0cf027225df