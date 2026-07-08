# Linux 运维详解 - Java 开发者必备

> 本文档面向 Java 后端开发者，系统梳理 Linux 运维核心知识，涵盖基础命令、性能分析、故障排查、Shell 脚本、系统调优、容器化等内容。适合面试备战与线上问题排查。

---

## 目录

- [Part 1: Linux 基础](#part-1-linux-基础)
- [Part 2: 常用命令大全](#part-2-常用命令大全)
- [Part 3: 进程管理](#part-3-进程管理)
- [Part 4: 内存管理](#part-4-内存管理)
- [Part 5: CPU 分析](#part-5-cpu-分析)
- [Part 6: 磁盘与 IO](#part-6-磁盘与-io)
- [Part 7: 网络管理](#part-7-网络管理)
- [Part 8: 系统日志](#part-8-系统日志)
- [Part 9: 性能分析实战](#part-9-性能分析实战)
- [Part 10: Shell 脚本](#part-10-shell-脚本)
- [Part 11: 系统调优](#part-11-系统调优)
- [Part 12: 容器化运维（Docker）](#part-12-容器化运维docker)
- [Part 13: 定时任务与服务管理](#part-13-定时任务与服务管理)
- [Part 14: 安全加固](#part-14-安全加固)
- [Part 15: 常见面试题 FAQ](#part-15-常见面试题-faq)

---

# Part 1: Linux 基础

## 1.1 Linux 发行版选型

Linux 发行版众多，作为 Java 开发者和运维人员，最常接触的是以下几个：

### CentOS / RHEL

**Red Hat Enterprise Linux（RHEL）** 是企业级商业发行版，提供官方支持和安全补丁，常用于金融、电信等对稳定性要求极高的场景。

**CentOS** 曾是 RHEL 的免费克隆版，与 RHEL 二进制兼容，是国内互联网公司使用最广泛的服务器操作系统之一。

> **注意**：CentOS 8 已于 2021 年 12 月 31 日停止维护，CentOS Stream 转变为 RHEL 的上游测试版。建议迁移到 **Rocky Linux** 或 **AlmaLinux**（均为 RHEL 的社区替代品）。

**特点：**
- 包管理：`yum`（CentOS 7）/ `dnf`（CentOS 8+）
- 防火墙：`firewalld`
- 服务管理：`systemd`
- 内核版本较保守，稳定性优先

```bash
# 查看系统版本
cat /etc/redhat-release
# 输出：CentOS Linux release 7.9.2009 (Core)

cat /etc/os-release
# 输出更详细的信息
```

### Ubuntu / Debian

**Ubuntu** 基于 Debian，以易用性著称，桌面版和服务器版均有广泛使用。**Ubuntu LTS（Long Term Support）** 版本提供 5 年官方支持，是云端部署的热门选择。

**特点：**
- 包管理：`apt` / `apt-get`
- 防火墙：`ufw`（底层仍是 iptables）
- 更新的软件包版本
- 社区活跃，文档丰富

```bash
# 查看 Ubuntu 版本
lsb_release -a
# 输出：
# Distributor ID: Ubuntu
# Description:    Ubuntu 22.04.3 LTS
# Release:        22.04
# Codename:       jammy

cat /etc/debian_version
```

### 发行版对比

| 特性 | CentOS/RHEL | Ubuntu/Debian |
|------|-------------|---------------|
| 包管理 | yum/dnf/rpm | apt/dpkg |
| 默认 Shell | bash | bash/dash |
| 防火墙 | firewalld | ufw |
| 服务路径 | /etc/sysconfig | /etc/default |
| 日志系统 | /var/log/messages | /var/log/syslog |
| 企业使用 | 国内广泛 | 云服务常见 |

### 选型建议

- **企业内网服务器**：Rocky Linux 8/9 或 AlmaLinux（CentOS 替代）
- **云服务器**：Ubuntu 22.04 LTS（AWS/GCP/阿里云均有官方镜像）
- **开发环境**：Ubuntu Desktop 或 WSL2（Ubuntu）
- **容器基础镜像**：Alpine（极小）/ Debian Slim / Ubuntu

---

## 1.2 Linux 目录结构详解

Linux 遵循 **FHS（Filesystem Hierarchy Standard，文件系统层次标准）**，所有目录从根目录 `/` 开始。

```
/
├── bin         -> usr/bin      # 基本用户命令二进制文件（ls, cp, mv...）
├── sbin        -> usr/sbin     # 系统管理员命令（fdisk, ifconfig...）
├── etc                         # 系统配置文件
├── var                         # 可变数据（日志、缓存、数据库）
├── usr                         # 用户级应用程序
│   ├── bin                     # 用户命令
│   ├── sbin                    # 系统命令
│   ├── lib                     # 库文件
│   ├── local                   # 本地安装的软件
│   └── share                   # 架构无关数据
├── tmp                         # 临时文件（重启后清空）
├── proc                        # 内核和进程的虚拟文件系统
├── sys                         # 内核设备和驱动信息（sysfs）
├── home                        # 普通用户家目录
│   └── alice                   # 用户 alice 的家目录
├── root                        # root 用户家目录
├── lib         -> usr/lib      # 共享库文件
├── lib64       -> usr/lib64    # 64位共享库
├── dev                         # 设备文件（硬盘/终端/随机数）
├── mnt                         # 临时挂载点
├── media                       # 可移动媒体挂载（U盘/光盘）
├── opt                         # 可选第三方软件（如 /opt/jdk）
├── srv                         # 服务数据（如 /srv/www）
├── run                         # 运行时数据（PID文件/socket）
└── boot                        # 引导程序文件（内核/initramfs）
```

### 关键目录详解

**`/etc` - 系统配置中心**

```bash
/etc/passwd          # 用户账户信息（用户名:密码:UID:GID:备注:家目录:Shell）
/etc/shadow          # 加密密码（只有root可读）
/etc/group           # 用户组信息
/etc/hosts           # 静态主机名解析
/etc/resolv.conf     # DNS 配置
/etc/fstab           # 文件系统挂载配置
/etc/crontab         # 系统级定时任务
/etc/sysctl.conf     # 内核参数配置
/etc/profile         # 全局环境变量（所有用户登录时执行）
/etc/bashrc          # 全局 bash 配置
/etc/sudoers         # sudo 权限配置（用 visudo 编辑）
/etc/ssh/sshd_config # SSH 服务配置
/etc/nginx/          # Nginx 配置目录
/etc/systemd/system/ # systemd 服务单元文件
```

**`/var` - 动态变化数据**

```bash
/var/log/            # 系统和应用日志
/var/log/messages    # 系统消息（CentOS）
/var/log/syslog      # 系统日志（Ubuntu）
/var/log/auth.log    # 认证日志（登录/sudo）
/var/log/cron        # 定时任务日志
/var/log/nginx/      # Nginx 日志
/var/run/            # 运行时 PID 文件
/var/spool/cron/     # 用户 crontab 文件
/var/cache/          # 应用缓存
/var/lib/            # 应用状态数据（MySQL数据文件等）
```

**`/proc` - 进程与内核信息**

```bash
/proc/cpuinfo        # CPU 信息
/proc/meminfo        # 内存信息
/proc/loadavg        # 系统负载
/proc/net/tcp        # TCP 连接状态
/proc/net/dev        # 网络接口统计
/proc/sys/           # 内核参数（sysctl 对应此目录）
/proc/[PID]/         # 每个进程的目录
/proc/[PID]/status   # 进程状态
/proc/[PID]/cmdline  # 进程启动命令
/proc/[PID]/fd/      # 进程打开的文件描述符
/proc/[PID]/maps     # 内存映射
/proc/[PID]/environ  # 环境变量
/proc/[PID]/limits   # 资源限制
```

**`/sys` - 内核设备信息（sysfs）**

```bash
/sys/block/          # 块设备
/sys/class/net/      # 网络设备
/sys/kernel/mm/      # 内存管理（透明大页等）
/sys/kernel/mm/transparent_hugepage/enabled  # 透明大页状态
```

---

## 1.3 文件权限

### 权限基础

Linux 文件权限分三组：**所有者（User）**、**所属组（Group）**、**其他人（Others）**，每组有读（r=4）、写（w=2）、执行（x=1）三种权限。

```bash
ls -l /etc/passwd
# -rw-r--r-- 1 root root 1234 Jan  1 00:00 /etc/passwd
#  ^          ^ ^    ^
#  |          | |    |
#  |          | |    +-- 其他人: r-- = 4 (只读)
#  |          | +------- 所属组: r-- = 4 (只读)
#  |          +--------- 所有者: rw- = 6 (读写)
#  +-------------------- 文件类型: - 普通文件 d 目录 l 软链接 c 字符设备 b 块设备
```

### chmod - 修改权限

```bash
# 数字方式
chmod 755 script.sh     # rwxr-xr-x：所有者可读写执行，组/其他人可读执行
chmod 644 config.txt    # rw-r--r--：所有者可读写，其他人只读
chmod 600 id_rsa        # rw-------：只有所有者可读写（私钥必须是600）
chmod 777 /tmp/shared   # rwxrwxrwx：所有人可读写执行（不推荐）

# 符号方式
chmod u+x script.sh     # 给所有者添加执行权限
chmod g-w file.txt      # 去掉组的写权限
chmod o=r file.txt      # 其他人权限设为只读
chmod a+x script.sh     # 所有人添加执行权限（a = all）
chmod u+x,g+x file.sh   # 同时修改多个

# 递归修改
chmod -R 755 /var/www/html   # 递归修改目录及其内容
```

### chown / chgrp - 修改所有者和组

```bash
# 修改所有者
chown alice file.txt          # 将 file.txt 的所有者改为 alice
chown alice:devteam file.txt  # 同时修改所有者和所属组
chown -R www-data:www-data /var/www/  # 递归修改

# 只修改所属组
chgrp devteam project/        # 修改目录所属组
chgrp -R docker /var/run/docker.sock
```

### 特殊权限位

**SUID（Set User ID，4xxx）**

```bash
# 文件执行时以文件所有者的身份运行（而非当前用户）
# 经典例子：/usr/bin/passwd（普通用户修改自己的密码）
ls -l /usr/bin/passwd
# -rwsr-xr-x 1 root root ... /usr/bin/passwd
#     ^ s 表示 SUID（原来是 x 位置，变为 s）

chmod u+s /usr/bin/program   # 设置 SUID
chmod 4755 /usr/bin/program  # 数字方式：4 代表 SUID
```

**SGID（Set Group ID，2xxx）**

```bash
# 对文件：执行时以文件所属组身份运行
# 对目录：目录下新建的文件继承目录的所属组（协作目录常用）
ls -l /usr/bin/wall
# -rwxr-sr-x 1 root tty ... /usr/bin/wall
#        ^ s 表示 SGID

mkdir /shared
chmod g+s /shared       # 设置目录 SGID
chmod 2775 /shared      # 数字方式
```

**Sticky Bit（粘滞位，1xxx）**

```bash
# 对目录：只有文件所有者或 root 才能删除文件（即使其他人有写权限）
# 经典例子：/tmp 目录
ls -ld /tmp
# drwxrwxrwt 1 root root ... /tmp
#          ^ t 表示 Sticky Bit

chmod +t /shared         # 设置粘滞位
chmod 1777 /tmp          # 数字方式
```

---

## 1.4 软链接 vs 硬链接

### 硬链接（Hard Link）

硬链接是文件系统中指向同一个 **inode**（数据块）的多个目录项。

```bash
# 创建硬链接
ln original.txt hardlink.txt

# 验证：两者 inode 相同
ls -li original.txt hardlink.txt
# 123456 -rw-r--r-- 2 user user 100 ... original.txt
# 123456 -rw-r--r-- 2 user user 100 ... hardlink.txt
# 注意：第2列数字 2 = 硬链接计数

# 特点：
# 1. 删除 original.txt，hardlink.txt 仍然有效（inode 引用计数 > 0）
# 2. 不能跨文件系统
# 3. 不能链接目录（避免循环）
# 4. 修改任一文件，另一个也变化（指向同一数据）
```

### 软链接（Symbolic Link / Symlink）

软链接是一个特殊文件，内容是目标路径的字符串，类似 Windows 快捷方式。

```bash
# 创建软链接
ln -s /opt/jdk-17 /usr/local/java       # 绝对路径软链接（推荐）
ln -s ../config/app.conf ./app.conf     # 相对路径软链接

# 查看软链接
ls -l /usr/local/java
# lrwxrwxrwx 1 root root 10 ... /usr/local/java -> /opt/jdk-17
# ^ l 表示软链接

# 修改软链接指向（先删除再创建）
ln -sf /opt/jdk-21 /usr/local/java   # -f 强制覆盖

# 查找所有软链接
find /usr/local -type l -ls

# 特点：
# 1. 可以跨文件系统
# 2. 可以链接目录
# 3. 目标删除后，软链接变为"悬空链接"（dangling link）
# 4. 软链接有自己的 inode
```

### 对比

| 特性 | 硬链接 | 软链接 |
|------|--------|--------|
| 跨文件系统 | 否 | 是 |
| 链接目录 | 否 | 是 |
| 目标删除 | 数据仍存在 | 链接失效 |
| inode | 相同 | 不同 |
| 常用场景 | 文件备份 | 版本切换、路径别名 |

---

## 1.5 文件描述符与重定向

### 标准文件描述符

每个进程启动时默认有三个文件描述符：

| 描述符 | 名称 | 说明 |
|--------|------|------|
| 0 | stdin  | 标准输入（键盘） |
| 1 | stdout | 标准输出（终端） |
| 2 | stderr | 标准错误（终端） |

### 重定向操作

```bash
# 标准输出重定向（覆盖）
command > output.txt        # 等同于 command 1> output.txt

# 标准输出重定向（追加）
command >> output.txt

# 标准错误重定向
command 2> error.txt
command 2>> error.txt       # 追加

# 标准输出和标准错误都重定向到同一文件
command > all.log 2>&1      # 推荐方式（先重定向stdout，再把stderr指向stdout）
command &> all.log           # bash 简写
command >> all.log 2>&1     # 追加方式

# 常见错误写法（顺序错误）
# command 2>&1 > all.log    # 错！stderr 会输出到终端，只有 stdout 进文件

# 丢弃输出
command > /dev/null          # 丢弃标准输出
command > /dev/null 2>&1    # 丢弃所有输出
command &> /dev/null         # 同上

# 从文件读取输入
command < input.txt

# Here Document（内联输入）
cat << EOF
line 1
line 2
EOF

# Here String
grep "pattern" <<< "some string to search"
```

### 实际应用示例

```bash
# Java 应用日志重定向
java -jar app.jar > /var/log/app/stdout.log 2> /var/log/app/stderr.log &

# 同时输出到文件和终端（tee）
java -jar app.jar 2>&1 | tee /var/log/app/app.log

# 丢弃正常输出，只看错误
make build > /dev/null

# 查看命令输出，同时记录
curl http://api.example.com 2>&1 | tee response.log
```

---

## 1.6 管道（|）与 xargs

### 管道

管道将前一个命令的 **stdout** 连接到后一个命令的 **stdin**。

```bash
# 基础用法
ps aux | grep java           # 查找 java 进程
cat /etc/passwd | wc -l      # 统计用户数
ls -l | sort -k5 -rn         # 按文件大小排序

# 多级管道
ps aux | grep java | grep -v grep | awk '{print $2}'  # 获取 java 进程 PID

# 统计 Nginx 访问日志中各状态码出现次数
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -rn

# 查找占用磁盘最多的目录
du -h /var | sort -rh | head -20
```

### xargs

`xargs` 将 stdin 的输入转换为命令的参数（解决参数过多或参数含特殊字符的问题）。

```bash
# 基本用法：将 find 结果传给 rm（不用 xargs 可能报"参数列表过长"）
find /tmp -name "*.log" -mtime +7 | xargs rm -f

# -n：每次传几个参数
echo "a b c d" | xargs -n 2 echo
# a b
# c d

# -I：替换符号（类似 {} in find -exec）
find . -name "*.java" | xargs -I {} wc -l {}
ls *.bak | xargs -I {} mv {} {}.old

# -P：并行执行（加速）
find /data -name "*.gz" | xargs -P 4 -I {} gzip -d {}

# -0/-print0：处理含空格的文件名
find . -name "*.log" -print0 | xargs -0 rm -f

# 与 grep 结合：批量搜索
find . -name "*.java" | xargs grep -l "SQLException"

# 空输入不执行（-r / --no-run-if-empty）
find /empty -name "*.txt" | xargs -r rm   # 找不到文件时不执行 rm
```

---

## 1.7 通配符与正则表达式

### Shell 通配符（Glob）

通配符由 Shell 解释，用于文件名匹配：

```bash
*        # 匹配任意字符（包括空）
?        # 匹配单个任意字符
[abc]    # 匹配 a、b 或 c
[a-z]    # 匹配 a 到 z
[!abc]   # 不匹配 a、b、c
{a,b,c}  # 匹配 a 或 b 或 c（花括号展开，不是通配符）

# 示例
ls *.java             # 列出所有 .java 文件
ls test?.txt          # 匹配 test1.txt test2.txt 但不匹配 test10.txt
ls /var/log/[a-z]*.log
cp file{1,2,3}.txt /backup/   # 复制 file1.txt file2.txt file3.txt
mv app.{jar,jar.bak}          # 快速备份：app.jar -> app.jar.bak
```

### 正则表达式（Regex）

正则表达式由工具（grep/sed/awk）解释，用于文本内容匹配：

```bash
# 基本正则（BRE）
.        # 匹配任意单个字符（除换行）
*        # 前一个字符的 0 次或多次重复
^        # 行首
$        # 行尾
[abc]    # 字符集
[^abc]   # 否定字符集

# 扩展正则（ERE，grep -E 或 egrep）
+        # 前一个字符 1 次或多次
?        # 前一个字符 0 次或 1 次
{n,m}    # 前一个字符 n 到 m 次
(  )     # 分组（ERE 不需要转义）
|        # 或

# grep 示例
grep "^ERROR" app.log                    # 以 ERROR 开头的行
grep "Exception$" app.log               # 以 Exception 结尾的行
grep -E "[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}" access.log  # IP地址
grep -E "^[[:space:]]*$" file.txt       # 空行
grep -E "(ERROR|WARN|FATAL)" app.log    # 多关键字

# 常用模式
# ^$                空行
# ^[[:space:]]*$    空行或只有空白字符的行
# [0-9]+\.[0-9]+    浮点数
# [A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}  邮件地址
```


---

# Part 2: 常用命令大全

## 2.1 文件操作命令

### ls - 列出目录内容

```bash
ls              # 基本列表
ls -l           # 长格式（权限/所有者/大小/时间）
ls -la          # 包含隐藏文件
ls -lh          # 文件大小人性化显示（KB/MB/GB）
ls -lt          # 按修改时间倒序
ls -lS          # 按文件大小倒序
ls -lR          # 递归列出子目录
ls -ld /etc     # 只显示目录本身信息（不显示内容）
ls -li          # 显示 inode 号
ls -1           # 每行一个文件（适合脚本处理）

# 结合过滤
ls *.java
ls -l /var/log/*.log | sort -k5 -rn   # 按大小排序日志文件

# 颜色说明（ls --color=auto 默认）
# 蓝色：目录  绿色：可执行文件  红色：压缩文件  青色：软链接
```

### find - 强大的文件查找工具

```bash
# ---- 按名称查找 ----
find /home -name "*.log"              # 查找所有 .log 文件
find /home -name "*.Log" -o -name "*.log"  # 或条件
find . -iname "*.java"               # 忽略大小写
find / -name "nginx.conf" 2>/dev/null # 全局查找，忽略权限错误

# ---- 按类型查找 ----
find /var -type f        # 只找文件（f）
find /etc -type d        # 只找目录（d）
find /usr -type l        # 只找软链接（l）
find /dev -type c        # 字符设备（c）
find /dev -type b        # 块设备（b）

# ---- 按时间查找 ----
find /tmp -mtime +7      # 修改时间超过 7 天（+7 表示大于7天前，-7 表示7天内）
find /var/log -mtime -1  # 24小时内修改的文件
find . -newer reference.txt  # 比 reference.txt 更新的文件
find /backup -mmin -60   # 60 分钟内修改（mmin 按分钟）
find / -atime -1         # 1天内访问过（atime）
find / -ctime -7         # 7天内状态改变（ctime：权限/所有者变化）

# ---- 按大小查找 ----
find / -size +100M       # 大于 100MB 的文件
find /var -size +10M -size -1G  # 10MB 到 1GB 之间
find . -empty            # 空文件或空目录
find / -size 0           # 大小为 0 的文件

# ---- 按权限查找 ----
find / -perm 777         # 权限恰好是 777
find / -perm -777        # 权限包含 777（每一位都满足）
find / -perm /4000       # 设置了 SUID 的文件（安全审计）
find / -perm /6000       # 设置了 SUID 或 SGID 的文件

# ---- 按用户/组查找 ----
find /home -user alice    # alice 拥有的文件
find /var -group www-data # 属于 www-data 组的文件
find / -nouser           # 没有对应用户的文件（孤立文件）

# ---- 执行操作（-exec）----
# {} 代表找到的文件，\; 表示命令结束（逐个执行）
find /tmp -name "*.tmp" -exec rm -f {} \;

# 使用 + 号（类似 xargs，批量执行，效率更高）
find /log -name "*.log" -mtime +30 -exec gzip {} +

# 删除7天前的日志（生产常用）
find /var/log/app -name "*.log" -mtime +7 -exec rm -f {} \;

# 查找并修改权限
find /var/www -type f -exec chmod 644 {} \;
find /var/www -type d -exec chmod 755 {} \;

# 查找大文件（线上排查磁盘满）
find / -size +500M -type f 2>/dev/null | xargs ls -lh

# 结合多个条件（AND：默认，OR：-o，NOT：!）
find /etc -name "*.conf" -size +1k          # 大于1k 的 .conf 文件
find /etc \( -name "*.conf" -o -name "*.cfg" \) -type f
find /etc ! -name "*.conf" -type f          # 非 .conf 文件
```

### grep - 文本搜索利器

```bash
# ---- 基本用法 ----
grep "pattern" file.txt              # 在文件中搜索
grep "ERROR" /var/log/app.log        # 查找错误日志
grep -r "NullPointerException" /var/log/   # 递归搜索目录

# ---- 常用选项 ----
grep -i "error" app.log      # 忽略大小写（case Insensitive）
grep -v "DEBUG" app.log      # 反向匹配（显示不含 DEBUG 的行）
grep -n "Exception" app.log  # 显示行号（line Number）
grep -l "error" /var/log/*.log   # 只显示匹配的文件名（file List）
grep -c "error" app.log          # 只显示匹配行数（Count）
grep -w "error" app.log          # 全词匹配（Word boundary）
grep -x "exact line" file.txt    # 全行匹配（eXact）

# ---- 上下文显示 ----
grep -A 5 "OutOfMemoryError" app.log   # 匹配行及后 5 行（After）
grep -B 3 "OutOfMemoryError" app.log   # 匹配行及前 3 行（Before）
grep -C 5 "OutOfMemoryError" app.log   # 前后各 5 行（Context）

# ---- 正则表达式 ----
grep -E "(ERROR|FATAL)" app.log       # 扩展正则（等同于 egrep）
grep -P "\d{4}-\d{2}-\d{2}" log.txt  # Perl 正则（需要 -P）
grep "^2024" app.log                  # 以 2024 开头的行
grep "Exception$" app.log            # 以 Exception 结尾的行

# ---- 递归搜索 ----
grep -r "password" /etc/             # 搜索所有文件（慎用）
grep -rl "TODO" /project/src/        # 只显示含 TODO 的文件名
grep -rn "import java.util" --include="*.java" /project/

# ---- 实用场景 ----
# 查找 Java 进程
ps aux | grep "[j]ava"    # 技巧：[j]ava 不会匹配 grep 自身

# 统计 HTTP 状态码
awk '{print $9}' /var/log/nginx/access.log | grep -E "^[0-9]{3}$" | sort | uniq -c

# 查看配置文件（过滤注释和空行）
grep -Ev "^#|^$" /etc/nginx/nginx.conf

# 查找包含 IP 地址的行
grep -E "\b([0-9]{1,3}\.){3}[0-9]{1,3}\b" access.log
```

### sed - 流编辑器

`sed` 逐行处理文本，常用于替换、删除、插入操作。

```bash
# ---- 替换（s命令）----
sed 's/old/new/' file.txt              # 每行替换第一个匹配
sed 's/old/new/g' file.txt            # 全局替换（g = global）
sed 's/old/new/gi' file.txt           # 忽略大小写替换
sed 's/old/new/2' file.txt            # 替换每行第 2 个匹配
sed -i 's/old/new/g' file.txt         # 原地修改文件（in-place）
sed -i.bak 's/old/new/g' file.txt     # 原地修改，同时保留 .bak 备份

# 分隔符可以是任意字符（路径中含/时很有用）
sed 's|/old/path|/new/path|g' file.txt
sed 's#foo#bar#g' file.txt

# 引用匹配组（&=整个匹配，\1=第1组）
sed 's/[0-9]\+/[&]/g' file.txt         # 给所有数字加方括号
sed -E 's/(ERROR|WARN)/[\1]/g' file.txt # 给 ERROR 或 WARN 加方括号

# ---- 删除行（d命令）----
sed '3d' file.txt                  # 删除第 3 行
sed '3,5d' file.txt                # 删除第 3 到 5 行
sed '/^#/d' file.txt               # 删除注释行
sed '/^$/d' file.txt               # 删除空行
sed '/^#/d; /^$/d' file.txt        # 删除注释行和空行
sed -i '/password/d' config.txt    # 原地删除含 password 的行

# ---- 打印行（p命令）----
sed -n '5p' file.txt               # 只打印第 5 行（-n 抑制默认输出）
sed -n '5,10p' file.txt            # 打印第 5 到 10 行
sed -n '/ERROR/p' app.log          # 打印含 ERROR 的行
sed -n '/START/,/END/p' file.txt   # 打印 START 到 END 之间的行

# ---- 插入/追加行（i/a命令）----
sed '3i\inserted line' file.txt    # 在第 3 行前插入
sed '3a\appended line' file.txt    # 在第 3 行后追加
sed '/pattern/a\new line' file.txt # 在匹配行后追加

# ---- 实用场景 ----
# 修改配置文件参数
sed -i 's/^max_connections = .*/max_connections = 500/' /etc/mysql/my.cnf

# 删除 Windows 换行符（\r）
sed -i 's/\r//' script.sh

# 提取日志中特定时间段
sed -n '/2024-01-01 10:00/,/2024-01-01 11:00/p' app.log

# 批量替换目录下所有文件
sed -i 's/old_hostname/new_hostname/g' /etc/hosts /etc/hostname
```

### awk - 文本处理与数据提取

`awk` 以行为单位处理文本，默认按空白字符分字段，$1 $2... 代表各字段，$0 代表整行，NR 是行号，NF 是字段数。

```bash
# ---- 基础用法 ----
awk '{print $1}' file.txt              # 打印第 1 列
awk '{print $1, $3}' file.txt          # 打印第 1 和第 3 列
awk '{print NR, $0}' file.txt          # 打印行号和整行
awk '{print NF}' file.txt              # 打印每行字段数
awk '{print $NF}' file.txt             # 打印最后一列

# 自定义分隔符（-F）
awk -F: '{print $1, $3}' /etc/passwd   # 以 : 分隔，打印用户名和UID
awk -F',' '{print $2}' data.csv         # CSV 处理
awk -F'\t' '{print $1}' data.tsv        # Tab 分隔

# ---- 条件过滤 ----
awk '$3 > 1000' /etc/passwd            # 第3列（UID）大于1000
awk '/ERROR/ {print $0}' app.log       # 包含 ERROR 的行
awk '!/^#/ {print}' config.txt         # 非注释行
awk '$9 == "200"' access.log           # HTTP 状态码为 200 的行
awk 'NR>=10 && NR<=20' file.txt        # 打印第 10 到 20 行

# ---- BEGIN / END ----
# BEGIN 在处理文件前执行，END 在处理完后执行
awk 'BEGIN {print "Start"} {print $1} END {print "End"}' file.txt

# 统计文件总行数
awk 'END {print NR}' file.txt

# 计算第 5 列总和
awk '{sum += $5} END {print "Total:", sum}' data.txt

# 计算平均值
awk '{sum += $1; count++} END {print "Avg:", sum/count}' numbers.txt

# ---- 变量与运算 ----
awk -v threshold=100 '$5 > threshold {print $0}' data.txt  # 传入变量
awk '{count[$1]++} END {for (k in count) print k, count[k]}' log.txt  # 统计频率

# ---- 实用场景 ----
# 统计 Nginx 日志中各 IP 的请求次数（Top 10）
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -10

# 统计每个 HTTP 状态码的数量
awk '{count[$9]++} END {for (s in count) print s, count[s]}' access.log | sort

# 提取进程的 PID 和内存使用
ps aux | awk 'NR>1 {print $2, $4, $11}' | sort -k2 -rn | head -10

# 处理 /etc/passwd：打印 UID > 1000 的用户
awk -F: '$3 > 1000 {printf "User: %-15s UID: %d\n", $1, $3}' /etc/passwd

# 两个文件按字段关联（类似 JOIN）
awk 'NR==FNR{a[$1]=$2; next} $1 in a {print $0, a[$1]}' file1.txt file2.txt
```

### sort / uniq / wc / cut / tr

```bash
# ---- sort ----
sort file.txt                      # 字典序排序
sort -r file.txt                   # 反向排序
sort -n numbers.txt                # 数字排序（-n）
sort -rn numbers.txt               # 数字反向排序
sort -k2 data.txt                  # 按第2列排序
sort -k2 -t',' data.csv            # 指定分隔符，按第2列
sort -u file.txt                   # 排序并去重（等同于 sort | uniq）
sort -h sizes.txt                  # 按人性化大小排序（1K 2M 3G）

# ---- uniq ----
sort file.txt | uniq               # 去重（必须先排序）
sort file.txt | uniq -c            # 统计每行出现次数
sort file.txt | uniq -d            # 只显示重复的行
sort file.txt | uniq -u            # 只显示唯一的行（未重复）
sort file.txt | uniq -c | sort -rn # 频率统计（常用）

# ---- wc ----
wc -l file.txt    # 行数（line count）
wc -w file.txt    # 单词数（word count）
wc -c file.txt    # 字节数（character/byte count）
wc -m file.txt    # 字符数（多字节字符正确计算）
wc file.txt       # 显示行数/单词数/字节数

# 统计 Java 代码行数
find . -name "*.java" | xargs wc -l | tail -1

# ---- cut ----
cut -d: -f1 /etc/passwd            # 以 : 分隔，取第1列
cut -d: -f1,3 /etc/passwd          # 取第1和第3列
cut -c1-10 file.txt                # 取每行前10个字符
cut -c10- file.txt                 # 从第10个字符到行尾
cut -f2 data.tsv                   # Tab分隔文件取第2列

# ---- tr ----
tr 'a-z' 'A-Z' < file.txt         # 小写转大写
tr 'A-Z' 'a-z' < file.txt         # 大写转小写
tr -d '\r' < win.txt > unix.txt    # 删除 \r（Windows换行符）
tr -d ' ' < file.txt               # 删除所有空格
tr -s ' ' < file.txt               # 压缩连续空格为单个空格
tr ':' '\n' <<< "$PATH"           # 将 PATH 中的 : 替换为换行（查看PATH各路径）
echo "hello world" | tr -d aeiou  # 删除元音字母
```

### tar / gzip / zip / unzip

```bash
# ---- tar ----
# 常用选项：c=创建 x=解压 v=详细 f=指定文件 z=gzip j=bzip2 J=xz

# 打包并压缩
tar -czvf backup.tar.gz /var/www/html/      # 创建 gzip 压缩包
tar -cjvf backup.tar.bz2 /var/www/html/    # 创建 bzip2 压缩包（压缩率更高，更慢）
tar -cJvf backup.tar.xz /var/www/html/     # 创建 xz 压缩包（最高压缩率）
tar -cvf backup.tar /var/www/html/          # 只打包，不压缩

# 解压
tar -xzvf backup.tar.gz                    # 解压到当前目录
tar -xzvf backup.tar.gz -C /target/dir/   # 解压到指定目录（-C）
tar -xzvf backup.tar.gz --strip-components=1  # 去掉顶层目录

# 查看内容（不解压）
tar -tzvf backup.tar.gz                    # 列出压缩包内容

# 排除某些文件
tar -czvf backup.tar.gz /var/www/ --exclude="*.log" --exclude="*.tmp"

# 增量备份（配合 --newer）
tar -czvf incr.tar.gz --newer="2024-01-01" /data/

# ---- gzip / gunzip ----
gzip file.txt              # 压缩（生成 file.txt.gz，删除原文件）
gzip -k file.txt           # 压缩但保留原文件（-k = keep）
gzip -d file.txt.gz        # 解压
gunzip file.txt.gz         # 解压（等同于 gzip -d）
gzip -l file.txt.gz        # 查看压缩信息（压缩前后大小）
gzip -9 file.txt           # 最高压缩级别（默认是6）

# ---- zip / unzip ----
zip archive.zip file1.txt file2.txt    # 创建 zip
zip -r archive.zip /directory/         # 递归压缩目录
zip -e archive.zip secret.txt          # 加密压缩（会提示输入密码）
zip -u archive.zip newfile.txt         # 更新/追加到现有 zip

unzip archive.zip                      # 解压到当前目录
unzip archive.zip -d /target/          # 解压到指定目录
unzip -l archive.zip                   # 列出内容（不解压）
unzip -o archive.zip                   # 覆盖已有文件
unzip archive.zip file.txt             # 只解压特定文件
```

### diff / patch

```bash
# ---- diff ----
diff file1.txt file2.txt           # 比较两个文件
diff -u file1.txt file2.txt        # 统一格式（unified，更易读）
diff -y file1.txt file2.txt        # 并排显示
diff -r dir1/ dir2/                # 递归比较目录

# 输出格式说明（-u）：
# --- a/file1.txt  原文件
# +++ b/file2.txt  新文件
# @@ -1,3 +1,4 @@  原文第1行开始3行，新文第1行开始4行
# -old line        被删除的行（前面有 -）
# +new line        新增的行（前面有 +）
#  context line    上下文行（无前缀）

# 生成补丁文件
diff -u original.java modified.java > fix.patch

# ---- patch ----
patch file.txt < fix.patch         # 应用补丁
patch -R file.txt < fix.patch      # 回滚补丁（-R = reverse）
patch -p1 < fix.patch              # 去掉路径中的1个前缀（-p1 常用于 git 导出的 patch）
patch --dry-run file.txt < fix.patch  # 模拟应用（不实际修改）
```

---

## 2.2 文本查看命令

### cat / more / less / head / tail

```bash
# ---- cat ----
cat file.txt                    # 显示文件内容
cat -n file.txt                 # 显示行号
cat -A file.txt                 # 显示不可见字符（^ = Ctrl，M- = 高位字符，$ = 行尾）
cat file1.txt file2.txt > merged.txt  # 合并文件

# ---- more ----
more file.txt       # 分页查看（只能向下翻页，q 退出）
# 空格：下一页；Enter：下一行；q：退出

# ---- less ----
less file.txt       # 分页查看（推荐，可上下翻页）
# 快捷键：
#   空格/PgDn：下一页
#   b/PgUp：上一页
#   G：跳到文件末尾
#   g：跳到文件开头
#   /pattern：向下搜索
#   ?pattern：向上搜索
#   n：下一个匹配
#   N：上一个匹配
#   q：退出
#   F：类似 tail -f，实时监控

less +F app.log          # 打开后直接进入 follow 模式
less +G app.log          # 打开后直接跳到末尾
less -S app.log          # 不折行（长行水平滚动）

# ---- head ----
head file.txt           # 显示前 10 行（默认）
head -n 20 file.txt     # 显示前 20 行
head -20 file.txt       # 同上（简写）
head -c 100 file.txt    # 显示前 100 字节

# ---- tail ----
tail file.txt           # 显示最后 10 行
tail -n 50 app.log      # 显示最后 50 行
tail -100 app.log       # 同上
tail -c 500 file.txt    # 显示最后 500 字节
tail -f app.log         # 实时监控日志（follow）——最常用！
tail -F app.log         # 实时监控，文件轮转后自动重新打开（比 -f 更健壮）

# 组合使用
tail -f app.log | grep "ERROR"             # 实时过滤错误日志
tail -f app.log | grep --line-buffered "Exception"  # --line-buffered 避免缓冲问题
tail -n 1000 app.log | grep "OutOfMemory" # 最后1000行中搜索
head -n 100 file.txt | tail -n 50         # 取第51-100行（head取前100，tail取其中后50）
```

### vim 常用操作

vim 是 Linux 下最强大的文本编辑器，分三种模式：**普通模式（Normal）**、**插入模式（Insert）**、**命令模式（Command）**。

```
启动和退出：
  vim file.txt          打开/创建文件
  :q                    退出（未修改）
  :q!                   强制退出（不保存）
  :w                    保存
  :wq 或 ZZ            保存并退出
  :w newfile.txt        另存为

进入插入模式（从普通模式进入）：
  i                     在光标前插入
  a                     在光标后插入（append）
  I                     在行首插入
  A                     在行尾插入
  o                     在当前行下方新建行
  O                     在当前行上方新建行
  s                     删除当前字符并插入
  cc 或 S              删除当前行并插入
  Esc                   返回普通模式

移动（普通模式）：
  h j k l               左 下 上 右（或方向键）
  w                     下一个单词首
  b                     上一个单词首
  e                     当前/下一个单词尾
  0                     行首
  ^                     行首（非空字符）
  $                     行尾
  gg                    文件首行
  G                     文件末行
  :n 或 nG             跳到第 n 行（如 :25 跳到第25行）
  Ctrl+f                向下翻页
  Ctrl+b                向上翻页
  Ctrl+d                向下半页
  Ctrl+u                向上半页

编辑（普通模式）：
  x                     删除当前字符
  dd                    删除当前行
  ndd                   删除 n 行（如 5dd 删除5行）
  dw                    删除当前单词
  d$                    删除到行尾
  D                     同 d$
  yy                    复制当前行（yank）
  nyy                   复制 n 行
  p                     粘贴到当前行下方
  P                     粘贴到当前行上方
  u                     撤销（undo）
  Ctrl+r                重做（redo）
  .                     重复上次操作
  >>                    右缩进
  <<                    左缩进
  ~                     切换大小写
  r                     替换单个字符

查找（普通模式）：
  /pattern              向下搜索
  ?pattern              向上搜索
  n                     下一个匹配
  N                     上一个匹配
  *                     向下搜索当前光标所在单词
  #                     向上搜索当前光标所在单词

替换（命令模式）：
  :s/old/new/           替换当前行第一个
  :s/old/new/g          替换当前行所有
  :%s/old/new/g         替换全文所有
  :%s/old/new/gc        替换全文，每次询问确认
  :5,10s/old/new/g      替换 5-10 行
  :%s/\s\+$//           删除每行末尾空白（常用整理）

多窗口：
  :sp filename          水平分割窗口（split）
  :vsp filename         垂直分割窗口（vertical split）
  Ctrl+w w              切换窗口
  Ctrl+w h/j/k/l        在窗口间移动（方向）
  :close                关闭当前窗口

实用技巧：
  :set nu               显示行号
  :set nonu             隐藏行号
  :set paste            粘贴模式（防止自动缩进）
  :set nopaste          关闭粘贴模式
  :%d                   删除所有行
  :g/pattern/d          删除所有匹配行
  ggVGy                 全选并复制（gg=行首 V=可视行模式 G=末行 y=复制）
```

---

# Part 3: 进程管理

## 3.1 ps - 进程状态快照

```bash
# 两种常用格式
ps aux           # BSD 风格（无 -）
ps -ef           # POSIX 风格（有 -）

# ps aux 字段含义：
# USER     PID  %CPU %MEM    VSZ   RSS TTY    STAT START   TIME COMMAND
# root       1   0.0  0.0 171820  8908 ?      Ss   Jan01   0:03 /sbin/init

# USER：进程所有者
# PID：进程ID
# %CPU：CPU使用率
# %MEM：内存使用率（占物理内存百分比）
# VSZ：虚拟内存大小（KB）
# RSS：实际驻留物理内存（KB，Resident Set Size）
# TTY：关联终端（? 表示没有终端，为守护进程）
# STAT：进程状态
#   S = 睡眠，D = 不可中断睡眠（IO等待），R = 运行中，Z = 僵尸，T = 停止
#   s = 会话组长，< = 高优先级，l = 多线程
# START：进程启动时间
# TIME：累计 CPU 时间
# COMMAND：命令及参数

# ps -ef 字段含义（多了 PPID：父进程 ID）

# 常用操作
ps aux | grep java                  # 查找 Java 进程
ps aux --sort=-%mem | head -20      # 按内存使用降序
ps aux --sort=-%cpu | head -20      # 按 CPU 使用降序
ps -p 1234 -o pid,ppid,cmd,%mem     # 查看特定进程特定字段
ps -u alice                         # 查看特定用户的进程

# 查看进程的完整命令行（COMMAND可能被截断）
cat /proc/1234/cmdline | tr '\0' ' '

# 查看僵尸进程
ps aux | awk '$8=="Z" {print $0}'
```

## 3.2 top / htop

### top

```bash
top               # 实时显示进程信息
top -p 1234       # 只监控特定 PID
top -u alice      # 只显示特定用户的进程
top -b -n 1       # 批处理模式，输出1次（适合脚本使用）
top -b -n 1 -o %MEM  # 按内存排序输出

# top 交互快捷键：
# P：按 CPU 使用率排序（默认）
# M：按内存使用率排序
# T：按累计 CPU 时间排序
# k：杀进程（输入 PID 和信号）
# r：renice（调整进程优先级）
# q：退出
# 1：展开显示每个 CPU 核心
# H：显示线程（Java 调试时必用！！！）
# f：字段管理（添加/删除显示字段）
# s：修改刷新间隔
# u：过滤特定用户

# top 头部信息解读：
# top - 10:30:00 up 5 days,  3:21,  2 users,  load average: 0.52, 0.48, 0.45
#       当前时间  运行时间          登录用户数   负载均值（1/5/15分钟）
#
# Tasks: 234 total,   1 running, 233 sleeping,   0 stopped,   0 zombie
#        总进程数       运行中      睡眠中           停止          僵尸
#
# %Cpu(s):  5.2 us,  1.3 sy,  0.0 ni, 92.3 id,  0.8 wa,  0.0 hi,  0.4 si,  0.0 st
#           用户态   系统态    nice   空闲        IO等待   硬中断    软中断    虚拟机偷取
#
# MiB Mem :   7834.8 total,   1234.5 free,   4567.8 used,   2032.5 buff/cache
# MiB Swap:   2048.0 total,   2048.0 free,      0.0 used.   2890.3 avail Mem
```

### htop（需要安装）

```bash
# 安装
yum install -y htop    # CentOS
apt install -y htop    # Ubuntu

htop                   # 启动（比 top 更友好，支持鼠标）

# htop 快捷键：
# F2：设置/配置
# F3 或 /：搜索进程
# F4：过滤
# F5：树形视图（显示父子关系）
# F6：选择排序字段
# F9：杀进程（选择信号）
# F10 或 q：退出
# Space：标记进程（可多选）
# t：切换树形/列表视图
```

## 3.3 kill / killall / pkill - 发送信号

```bash
# 常用信号
# 1  SIGHUP    挂起（让进程重读配置，不终止）
# 2  SIGINT    中断（等同于 Ctrl+C）
# 9  SIGKILL   强制杀死（不能被捕获/忽略，立即终止）
# 15 SIGTERM   优雅终止（默认信号，可以被捕获，允许清理）
# 17 SIGCHLD   子进程状态改变
# 18 SIGCONT   继续（恢复被暂停的进程）
# 19 SIGSTOP   暂停（不能被捕获）
# 3  SIGQUIT   退出（生成 core dump）

kill -l              # 列出所有信号

# kill（按 PID）
kill 1234            # 发送 SIGTERM（优雅终止，给进程善后机会）
kill -9 1234         # 发送 SIGKILL（强制终止，立即杀死，丢失未保存数据）
kill -1 1234         # 发送 SIGHUP（重载配置，常用于 Nginx reload）
kill -15 1234        # 明确发送 SIGTERM

# killall（按进程名）
killall java                # 杀死所有名为 java 的进程（SIGTERM）
killall -9 java             # 强制杀死所有 java 进程
killall -u alice java       # 只杀死 alice 用户的 java 进程
killall -v java             # 显示详细信息

# pkill（按模式匹配，更灵活）
pkill java                  # 杀死名字匹配 java 的进程
pkill -9 -f "spring-boot"   # -f 匹配完整命令行（包括参数）
pkill -u alice              # 杀死 alice 的所有进程
pkill -t pts/1              # 杀死 pts/1 终端的所有进程

# 实践：优雅重启 Java 服务
APP_PID=$(cat /var/run/myapp.pid)
kill -15 $APP_PID
sleep 5
kill -0 $APP_PID 2>/dev/null && kill -9 $APP_PID  # kill -0 检查进程是否存在
```

## 3.4 jobs / bg / fg / nohup / &

```bash
# 前后台作业管理
command &           # 在后台运行命令
# Ctrl+Z            # 挂起当前前台进程（发送 SIGSTOP）
jobs                # 列出当前 shell 的作业
jobs -l             # 列出作业（包含 PID）
bg                  # 恢复最近的挂起作业到后台
fg                  # 恢复后台作业到前台（fg %1，%1 是作业号）
bg %2               # 恢复第2个作业到后台
fg %2               # 恢复第2个作业到前台
kill %1             # 杀死第1个作业

# nohup - 不挂断（关闭终端后进程继续运行）
nohup java -jar app.jar &                          # 后台运行，输出到 nohup.out
nohup java -jar app.jar > /var/log/app.log 2>&1 &  # 指定日志文件

# 运行后台进程并记录 PID
nohup ./script.sh > script.log 2>&1 &
echo $! > /var/run/script.pid    # $! = 最后一个后台进程的 PID

# disown - 从当前 shell 的作业列表中移除
java -jar app.jar &
disown %1           # 或 disown -h %1（保留作业表项，忽略 SIGHUP）

# screen / tmux（更推荐用于长时间运行）
screen -S mysession          # 创建命名会话
screen -ls                   # 列出会话
screen -r mysession          # 恢复会话
# Ctrl+a d                   # 分离会话（detach）
```

## 3.5 lsof - 列出打开的文件

lsof（List Open Files）是 Linux 下非常强大的诊断工具，因为"一切皆文件"。

```bash
lsof                      # 列出所有打开的文件（输出很多）
lsof | wc -l              # 统计系统打开的文件总数

# 按进程过滤（-p）
lsof -p 1234              # 查看 PID=1234 的进程打开的所有文件
lsof -p 1234 | grep REG   # 只看普通文件
lsof -p 1234 | grep LISTEN # 查看监听的端口

# 按文件/目录过滤
lsof /var/log/app.log      # 查看哪些进程打开了此文件（删除前检查）
lsof +D /var/www/          # 查看目录下被打开的文件（+D 递归）

# 按网络连接过滤（-i）
lsof -i                    # 所有网络连接
lsof -i :8080              # 查看哪个进程监听 8080 端口
lsof -i :8080-8090         # 端口范围
lsof -i tcp                # 所有 TCP 连接
lsof -i tcp:80             # TCP 80 端口
lsof -i @192.168.1.100     # 连接到特定 IP
lsof -i 4                  # IPv4 连接

# 按用户过滤（-u）
lsof -u alice              # alice 用户打开的文件
lsof -u ^root              # 非 root 用户打开的文件

# 常用场景
# 查找占用端口的进程（比 netstat 更直观）
lsof -i :8080 | grep LISTEN

# 查看 Java 进程打开的文件数（排查"Too many open files"）
lsof -p $(pgrep java) | wc -l

# 查看进程打开了哪些 .jar 文件
lsof -p 1234 | grep '\.jar'

# 文件被删除但磁盘空间未释放（df 和 du 不一致）
lsof | grep deleted         # 找到还在使用的已删除文件
# 解决方式：重启对应进程，或清空文件内容（> /proc/PID/fd/N）

# 输出格式
# COMMAND  PID   USER   FD   TYPE DEVICE SIZE/OFF    NODE NAME
# java    1234   root  cwd    DIR    8,1     4096  123456 /opt/app
# FD 字段含义：
#   cwd = current working directory
#   txt = 程序文本（代码段）
#   mem = 内存映射文件
#   数字 = 文件描述符（0=stdin 1=stdout 2=stderr 后续递增）
#   数字r = 读 数字w = 写 数字u = 读写
```

## 3.6 pstree - 进程树

```bash
pstree              # 显示进程树
pstree -p           # 显示 PID
pstree -u           # 显示用户名
pstree alice        # 显示 alice 的进程树
pstree 1234         # 以 PID=1234 为根的进程树
pstree -A           # ASCII 字符（避免乱码）
pstree -H 1234      # 高亮 PID=1234 及其祖先

# 示例输出（pstree -p）：
# init(1)─┬─sshd(1234)───sshd(5678)───bash(9012)───java(9013)
#         ├─nginx(2000)─┬─nginx(2001)
#         │             └─nginx(2002)
#         └─mysqld(3000)
```

## 3.7 strace - 系统调用追踪

```bash
# 追踪正在运行的进程（-p）
strace -p 1234              # 追踪 PID=1234 的系统调用
strace -p 1234 -e trace=read,write  # 只追踪 read/write 调用

# 追踪新启动的命令
strace ls -l               # 追踪 ls 命令的所有系统调用
strace -o trace.log ls -l  # 输出到文件

# 常用选项
strace -c command          # 统计各系统调用次数和耗时
strace -T command          # 显示每个调用的耗时
strace -e trace=network curl http://example.com  # 只看网络调用
strace -e trace=file ls    # 只看文件相关调用
strace -f java -jar app.jar   # 追踪子进程（-f = follow fork）
strace -tt command         # 显示绝对时间戳（微秒级）
strace -s 256 command      # 增大字符串截断长度（默认32）

# 诊断"卡住"的进程
strace -p 1234             # 看进程在哪个系统调用上阻塞了
```

## 3.8 /proc/[pid] 目录详解

```bash
PID=1234
ls /proc/$PID/

# 主要文件/目录：
# /proc/$PID/status      进程详细状态（VmRSS/VmPeak/Threads等）
# /proc/$PID/cmdline     启动命令（null分隔的参数）
# /proc/$PID/environ     环境变量（null分隔）
# /proc/$PID/maps        内存区域映射（地址/权限/文件）
# /proc/$PID/smaps       详细内存映射（包含每个区域的详细统计）
# /proc/$PID/fd/         打开的文件描述符（符号链接）
# /proc/$PID/fdinfo/     文件描述符详细信息
# /proc/$PID/limits      进程资源限制（ulimit）
# /proc/$PID/stat        进程统计（CPU时间等，top读取此文件）
# /proc/$PID/statm       内存统计（页面数）
# /proc/$PID/wchan       进程等待的内核函数
# /proc/$PID/net/        网络信息
# /proc/$PID/task/       线程目录（每个线程一个子目录）

# 读取示例
cat /proc/$PID/status
# Name:   java
# State:  S (sleeping)
# Pid:    1234
# PPid:   1000
# Threads: 48
# VmPeak:  4096 kB    # 虚拟内存峰值
# VmRSS:   2048 kB    # 当前物理内存使用（常用）
# VmSwap:   256 kB    # 使用的 swap

# 查看进程打开的文件数
ls /proc/$PID/fd | wc -l

# 查看进程命令行（参数含空格时很有用）
cat /proc/$PID/cmdline | tr '\0' '\n'

# 查看环境变量
cat /proc/$PID/environ | tr '\0' '\n' | grep JAVA_HOME

# 查看线程数
cat /proc/$PID/status | grep Threads
```


---

# Part 4: 内存管理

## 4.1 free - 内存使用概览

```bash
free -h        # 人性化显示（Human readable）
free -m        # 以 MB 显示
free -g        # 以 GB 显示
free -s 5      # 每 5 秒刷新一次

# 输出示例：
#               total        used        free      shared  buff/cache   available
# Mem:           7.6G        4.1G        512M        256M        3.0G        3.0G
# Swap:          2.0G          0B        2.0G

# 字段解析：
# total：物理内存总量
# used：已使用（total - free - buffers - cache）
# free：完全未使用的内存
# shared：多进程共享内存（tmpfs等）
# buff/cache：缓冲区（buffers）+ 文件缓存（cache）
# available：可用内存（free + 可被回收的buff/cache）

# available vs free 的区别（高频面试题）：
# free = 完全空闲的内存，数值可能很小
# available = free + 可回收的 buffer/cache，更能反映实际可用量
# 例如：free=512M，available=3.0G，说明有大量缓存可回收
# 判断内存是否不足，应看 available，而不是 free！
```

## 4.2 /proc/meminfo 详解

```bash
cat /proc/meminfo

# 关键字段：
# MemTotal:       8011756 kB    # 物理内存总量
# MemFree:         525060 kB    # 完全未使用内存
# MemAvailable:   3145668 kB    # 可用内存（更准确）
# Buffers:         214748 kB    # 块设备 I/O 缓冲
# Cached:         2636000 kB    # 文件系统缓存（pagecache）
# SwapCached:          0 kB     # 在 swap 中但也在内存中的页面
# Active:         3200000 kB    # 最近使用的内存（不容易被回收）
# Inactive:       1800000 kB    # 较久未使用（可被回收）
# SwapTotal:      2097148 kB    # swap 总大小
# SwapFree:       2097148 kB    # 可用 swap
# Dirty:             1234 kB    # 脏页（已修改但未写回磁盘）
# Writeback:            0 kB    # 正在写回磁盘的页
# AnonPages:      2600000 kB    # 匿名内存（堆/栈，不对应文件）
# Mapped:          420000 kB    # 内存映射文件
# Shmem:           262144 kB    # 共享内存
# Slab:            200000 kB    # 内核 slab 分配器使用的内存
# SReclaimable:   150000 kB     # 可回收的 slab 内存（目录/inode缓存）
# SUnreclaim:      50000 kB     # 不可回收的 slab 内存
# HugePages_Total:      0       # 大页总数
# HugePages_Free:       0       # 空闲大页数
# HugePages_Rsvd:       0       # 已预留大页数
# Hugepagesize:       2048 kB   # 单个大页大小（2MB）

# Java 调优时关注：
# AnonPages 增大 = JVM Heap 使用增多（或内存泄漏）
# SwapFree 减小 = 内存不足，开始使用 swap（Java 性能急剧下降）
# Dirty 长期很大 = IO 瓶颈，数据来不及写盘
```

## 4.3 vmstat - 虚拟内存统计

```bash
vmstat                      # 输出一次
vmstat 1                    # 每秒输出一次
vmstat 1 10                 # 每秒一次，共10次
vmstat -a                   # 显示活跃/非活跃内存
vmstat -s                   # 汇总信息（内存事件统计）
vmstat -d                   # 磁盘统计
vmstat -t                   # 添加时间戳

# 输出解释：
# procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
#  r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
#  1  0      0 512000 214748 2636000   0    0     1     5  100  200  5  1 93  1  0

# procs 进程：
#   r = 运行队列长度（等待 CPU 的进程数，持续 > CPU核心数说明CPU瓶颈）
#   b = 阻塞进程数（不可中断睡眠，通常是IO等待）

# memory 内存：
#   swpd = 已使用 swap
#   free = 空闲内存
#   buff = buffer（块设备缓冲）
#   cache = 文件系统缓存

# swap 换页（重点！）：
#   si = swap in（从 swap 读到内存，单位 KB/s）
#   so = swap out（从内存写到 swap，单位 KB/s）
#   si/so 持续非0 = 内存严重不足，系统在大量使用 swap，Java性能会极差！

# io 磁盘：
#   bi = blocks in（读磁盘，KB/s）
#   bo = blocks out（写磁盘，KB/s）

# system 系统：
#   in = 每秒中断次数（interrupts）
#   cs = 每秒上下文切换次数（context switches）
#       cs 极高可能是锁竞争激烈或线程过多

# cpu（百分比）：
#   us = 用户态，sy = 内核态，id = 空闲，wa = IO等待，st = 虚拟机偷取
```

## 4.4 内存泄漏排查

```bash
# ---- pmap - 查看进程内存映射 ----
pmap -x 1234           # 显示进程内存映射详情（-x 扩展格式）
pmap -x 1234 | tail -1 # 只看最后一行汇总（total kB）

# pmap 输出示例：
# Address           Kbytes     RSS   Dirty Mode  Mapping
# 00007f1234000000  131072   65536   65536 rw---   [ anon ]
#                              ...
# total kB        4194304  524288  131072

# ---- Java 内存排查步骤 ----
# 1. 观察趋势（jstat 监控 GC 后 Old Gen 是否持续增长）
jstat -gcutil 1234 5000     # 每 5 秒输出一次 GC 信息

# 2. 查看堆内存详情
jmap -heap 1234             # 堆内存摘要

# 3. 生成堆 dump 文件
jmap -dump:format=b,file=/tmp/heap.hprof 1234

# 4. 分析堆 dump（使用 MAT / jvisualvm）
# 详见 Part 9 Java 进程排查专题

# ---- 系统级内存泄漏指标 ----
# 以下命令追踪内存缓慢增长
watch -n 5 "ps -p 1234 -o pid,rss,vsz --no-headers"
# 若 RSS 持续增长但 GC 不回收，大概率是堆外内存泄漏（直接内存/JNI）
```

## 4.5 Buffer vs Cache

这是一个高频面试题！

```
Buffer（缓冲区）：
- 用于块设备 I/O 操作的缓冲
- 存储原始磁盘块（raw disk blocks）
- 写操作时，先写到 buffer，再批量写入磁盘
- 通过 sync/fsync 可以强制刷盘
- 读 /dev/sdb 这类块设备时产生的缓存

Cache（文件系统缓存，Page Cache）：
- 文件系统层面的缓存
- 存储最近读写的文件页（page cache）
- 读文件时，OS 会缓存读取过的内容
- 下次读同一文件，直接从 cache 读取（不走磁盘）

两者关系：
- Linux 2.4 后两者基本统一，都使用 page cache
- free 命令的 buff/cache 列是两者之和
- 当内存不足时，OS 可以回收（drop cache）

手动释放缓存（谨慎使用，会影响性能！）：
sync               # 先把脏页写盘
echo 1 > /proc/sys/vm/drop_caches   # 清除 pagecache
echo 2 > /proc/sys/vm/drop_caches   # 清除 dentries 和 inodes
echo 3 > /proc/sys/vm/drop_caches   # 清除 pagecache + dentries + inodes

Java 相关：
- JVM 读写文件（如 mmap）会占用 page cache
- 直接内存（DirectByteBuffer）绕过 JVM 堆，直接分配系统内存
- 堆外内存泄漏时，free -h 的 available 持续减小但 JVM 堆没增长
```

## 4.6 OOM Killer

当系统内存耗尽时，Linux 内核的 **OOM Killer（Out of Memory Killer）** 会自动选择并杀死进程以释放内存。

```bash
# 查看 OOM 日志
dmesg | grep -i "out of memory"
grep -i "out of memory" /var/log/messages   # CentOS
grep -i "killed process" /var/log/kern.log  # 内核日志

# 典型 OOM 日志：
# [12345.678] Out of memory: Kill process 1234 (java) score 800 or sacrifice child
# [12345.679] Killed process 1234 (java) total-vm:4096kB, anon-rss:3072kB

# oom_score - OOM 优先级（0-1000，越高越容易被杀）
cat /proc/1234/oom_score         # 查看 oom 分数
cat /proc/1234/oom_score_adj     # OOM 分数调整值（-1000 到 1000）

# 保护重要进程（不被 OOM Killer 杀死）
echo -1000 > /proc/$(pgrep mysql)/oom_score_adj  # 数据库等关键进程
echo 0 > /proc/1234/oom_score_adj                # 重置为默认

# 调低 Java 进程被 OOM 杀死的优先级
echo -500 > /proc/$(pgrep -f "app.jar")/oom_score_adj

# Java 进程 OOM vs 系统 OOM 区别：
# Java OOM: JVM 堆内存耗尽，抛 java.lang.OutOfMemoryError，JVM 继续运行
# 系统 OOM: 系统物理内存耗尽，内核 OOM Killer 直接杀死进程（JVM 进程消失）
```

## 4.7 大页内存（HugePage）

```bash
# 标准 Linux 页大小为 4KB，HugePage 为 2MB（或 1GB）
# 减少 TLB miss，适合大内存应用（数据库、Java 大堆）

# 查看 HugePage 信息
cat /proc/meminfo | grep -i huge
# HugePages_Total:       0
# HugePages_Free:        0
# HugePages_Rsvd:        0
# Hugepagesize:       2048 kB

# 配置大页数量（需要 root）
echo 512 > /proc/sys/vm/nr_hugepages  # 分配 512 个 2MB 大页 = 1GB
# 永久配置：
echo "vm.nr_hugepages = 512" >> /etc/sysctl.conf

# Java 使用大页（JVM 参数）
java -XX:+UseLargePages -Xmx4g -jar app.jar

# 透明大页（THP - Transparent HugePage）
cat /sys/kernel/mm/transparent_hugepage/enabled
# [always] madvise never
# always = 总是启用（默认），madvise = 应用申请时才用，never = 禁用

# 注意：THP 对 Java 可能有负面影响（GC 时的内存压缩会导致停顿）
# 建议在 Java 生产环境禁用 THP（详见 Part 11）
```

---

# Part 5: CPU 分析

## 5.1 top CPU 指标详解

```
%Cpu(s):  5.2 us,  1.3 sy,  0.0 ni, 92.3 id,  0.8 wa,  0.0 hi,  0.4 si,  0.0 st

us（user）：用户态 CPU 占用
- 应用程序代码执行时间
- Java 程序的业务逻辑、GC 等都算在这里
- 正常业务高峰 us 升高是正常的

sy（system/kernel）：内核态 CPU 占用
- 系统调用、中断处理、内核线程
- sy 很高说明频繁进行系统调用（如大量线程创建/销毁、频繁 IO）
- Java 中：NIO、多线程竞争、网络 IO 会导致 sy 升高

ni（nice）：低优先级进程（nice值>0）的 CPU 占用
- 通常为 0，有 cron 批处理任务时可能升高

id（idle）：CPU 空闲率
- 100% - id = CPU 总使用率
- id 持续接近 0 = CPU 瓶颈

wa（iowait）：等待 IO 完成的时间
- wa 高 = CPU 在等待磁盘/网络 IO
- Java 应用 wa 高：频繁日志写入、数据库慢查询、磁盘慢
- wa 高不等于 CPU 被占用，但会降低吞吐

hi（hardware interrupt）：硬件中断处理时间
- 网卡/磁盘控制器的中断
- hi 很高可能是网络流量过大

si（software interrupt）：软中断处理时间
- 网络协议栈处理（softirq）
- si 很高通常是高并发网络请求

st（steal）：被虚拟机监控器偷取的时间（虚拟机场景）
- 云服务器上 st 高 = 宿主机资源不足，影响虚拟机性能
- st 持续 > 5% 说明云服务器 CPU 资源被大量"偷取"
```

## 5.2 mpstat - 多核 CPU 统计

```bash
# 安装（sysstat 包）
yum install -y sysstat   # CentOS
apt install -y sysstat   # Ubuntu

mpstat                   # 所有 CPU 平均统计
mpstat -P ALL            # 每个 CPU 核心分别显示
mpstat -P ALL 1          # 每秒刷新一次
mpstat -P ALL 1 5        # 每秒刷新，共5次
mpstat -P 0              # 只显示 CPU0

# 输出示例：
# CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
# all    5.20    0.00    1.30    0.80    0.00    0.40    0.00    0.00    0.00   92.30
#   0    8.00    0.00    2.00    1.00    0.00    0.50    0.00    0.00    0.00   88.50
#   1    2.40    0.00    0.60    0.60    0.00    0.30    0.00    0.00    0.00   96.10

# CPU 不均衡问题：
# 如果某个核心 100%，其他核心空闲，说明是单线程问题（如 Java 单线程 CPU 密集）
# 用 top H 查看 Java 线程级别的 CPU 使用
```

## 5.3 pidstat - 进程 CPU/IO 统计

```bash
pidstat                      # 所有进程统计（一次）
pidstat 1                    # 每秒刷新
pidstat 1 5                  # 每秒刷新，共5次
pidstat -p 1234              # 特定进程
pidstat -u -p 1234 1         # CPU 统计（-u = CPU）
pidstat -d -p 1234 1         # IO 统计（-d = disk）
pidstat -r -p 1234 1         # 内存统计（-r = memory）
pidstat -w -p 1234 1         # 上下文切换统计（-w = wait/switch）
pidstat -t -p 1234           # 显示线程级别（-t = threads）

# CPU 统计输出：
# PID    %usr %system  %guest   %wait    %CPU   CPU  Command
# 1234   50.0     2.0     0.0    0.0    52.0     3   java

# IO 统计输出：
# PID   kB_rd/s   kB_wr/s kB_ccwr/s iodelay  Command
# 1234     0.00    1024.0      0.00       0   java

# 上下文切换：
# PID   cswch/s nvcswch/s  Command
# cswch/s = 自愿上下文切换（等待IO/锁等，进程主动放弃CPU）
# nvcswch/s = 非自愿上下文切换（被抢占，时间片用完）
# nvcswch/s 过高说明线程太多或 CPU 过载
```

## 5.4 perf - 性能分析工具

```bash
# 安装
yum install -y perf    # CentOS
apt install -y linux-tools-common linux-tools-$(uname -r)  # Ubuntu

# 基本使用
perf top                       # 实时 CPU 热点函数（类似 top）
perf top -p 1234               # 特定进程的热点函数
perf stat java -jar app.jar    # 统计命令的性能计数器（CPU cycles/instructions等）
perf record -p 1234 sleep 30   # 录制30秒的性能数据
perf report                    # 分析录制的数据（交互式）
perf report --stdio            # 文本格式输出

# 火焰图（Flame Graph）生成
# git clone https://github.com/brendangregg/FlameGraph
perf record -F 99 -p 1234 -g sleep 30   # 以 99Hz 采样，包含调用栈
# perf script | ./FlameGraph/stackcollapse-perf.pl | ./FlameGraph/flamegraph.pl > flame.svg

# Java 火焰图推荐使用 async-profiler（无 perf 依赖）
# ./profiler.sh -d 30 -f /tmp/flame.html <java_pid>
```

## 5.5 CPU 飙高排查完整步骤

**这是面试最高频的问题之一！**

```bash
# ===== 步骤 1：确认 CPU 高的进程 =====
top         # 按 P 排序（CPU降序）
# 记录 CPU 最高的 PID，例如 PID=1234（java 进程）

# ===== 步骤 2：找到消耗 CPU 的线程 =====
# top 按线程显示
top -Hp 1234    # 显示 PID=1234 进程的所有线程

# 或用 ps 列出线程
ps -mp 1234 -o THREAD,tid,time | sort -rn | head -20
# 记录 CPU 最高的线程 TID，例如 TID=5678

# ===== 步骤 3：将线程 TID 转换为十六进制 =====
printf '%x\n' 5678   # 输出：162e（十六进制）

# ===== 步骤 4：用 jstack 获取线程 dump =====
jstack 1234 > /tmp/jstack.txt
# 或
jstack -l 1234 | tee /tmp/jstack.txt    # -l 显示锁信息

# ===== 步骤 5：在 jstack 中搜索该线程 =====
grep -A 20 "nid=0x162e" /tmp/jstack.txt
# nid 是十六进制的线程 ID

# 输出示例：
# "http-nio-8080-exec-1" #48 daemon prio=5 os_prio=0 cpu=12345.67ms elapsed=678.90s
#     tid=0x00007f1234567890 nid=0x162e runnable [0x00007f1234abc000]
#    java.lang.Thread.State: RUNNABLE
#         at com.example.service.OrderService.processOrder(OrderService.java:125)
#         at com.example.controller.OrderController.createOrder(OrderController.java:45)
#         ...
# 这样就定位到了 CPU 热点代码！

# ===== 完整一键脚本 =====
#!/bin/bash
PID=$(jps | grep -i app | awk '{print $1}')
echo "Java PID: $PID"
echo "=== 高CPU线程 ==="
ps -mp $PID -o THREAD,tid,time | sort -rn | head -5
HIGH_TID=$(ps -mp $PID -o THREAD,tid,time | sort -rn | sed -n '2p' | awk '{print $2}')
HEX_TID=$(printf '%x' $HIGH_TID)
echo "High CPU Thread TID: $HIGH_TID (0x$HEX_TID)"
echo "=== Thread Stack ==="
jstack $PID | grep -A 20 "nid=0x$HEX_TID"
```

## 5.6 负载（Load Average）vs CPU 使用率

```bash
# 查看负载
uptime
# 10:30:00 up 5 days,  3:21,  2 users,  load average: 0.52, 0.48, 0.45
#                                                      1分钟  5分钟  15分钟

cat /proc/loadavg
# 0.52 0.48 0.45 1/234 5678
#                ^/^   ^
#                |      最近创建的进程PID
#                运行中进程/总进程数

# Load Average 的含义：
# = 单位时间内，处于"可运行状态"和"不可中断睡眠状态"的平均进程数
# 注意：包含了等待磁盘 IO 的进程（D 状态）！

# Load Average vs CPU 使用率：
# 单核 CPU，Load=1.0 = CPU 100% 利用率（最优）
# 单核 CPU，Load=2.0 = 有进程在等待 CPU（队列长度=1）
# 4核 CPU，Load=4.0 = 每个核心都满负荷（合理）
# 4核 CPU，Load=8.0 = 每个核心平均有1个等待（繁忙）

# 判断 CPU 瓶颈 vs IO 瓶颈：
# CPU 瓶颈：Load 高 + CPU 使用率高（us/sy 高，wa 低）
# IO 瓶颈：Load 高 + CPU 使用率不高（wa 高，us/sy 低）
# Load 很高但 CPU 使用率低：通常是大量 IO 等待

# 获取 CPU 核心数
nproc                         # 可用 CPU 核心数
grep -c processor /proc/cpuinfo
lscpu | grep "^CPU(s):"
```

## 5.7 /proc/cpuinfo

```bash
cat /proc/cpuinfo

# 关键字段：
# processor       : 0          # 逻辑 CPU 编号（0开始）
# physical id     : 0          # 物理 CPU 编号
# core id         : 0          # 核心编号
# siblings        : 4          # 同一物理 CPU 的逻辑 CPU 数（含超线程）
# cpu cores       : 2          # 同一物理 CPU 的物理核心数
# model name      : Intel(R) Core(TM) i7-8750H CPU @ 2.20GHz
# cpu MHz         : 2200.000   # 当前主频
# cache size      : 9216 KB    # L3 缓存
# flags           : fpu vme ... sse4_2 vmx ...  # CPU 特性标志

# 统计信息
nproc                              # 逻辑 CPU 数
grep -c ^processor /proc/cpuinfo  # 逻辑 CPU 数（等同于 nproc）
grep "physical id" /proc/cpuinfo | sort -u | wc -l  # 物理 CPU 数
grep "core id" /proc/cpuinfo | sort -u | wc -l      # 核心数（单个物理 CPU）

# 是否支持虚拟化（vmx = Intel VT-x，svm = AMD-V）
grep -E "vmx|svm" /proc/cpuinfo

# 查看 CPU 主频（当前动态频率）
grep "cpu MHz" /proc/cpuinfo | awk '{print $NF}' | sort -n
```

---

# Part 6: 磁盘与 IO

## 6.1 df - 磁盘使用情况

```bash
df -h                    # 人性化显示（Human readable）
df -hT                   # 显示文件系统类型（-T）
df -i                    # 显示 inode 使用情况（而非磁盘容量）
df -h /var               # 只看 /var 挂载点
df --total               # 最后一行显示总计

# 输出示例：
# Filesystem      Size  Used Avail Use% Mounted on
# /dev/sda1        50G   30G   18G  63% /
# /dev/sdb1       200G  180G    8G  96% /data
# tmpfs           3.9G     0  3.9G   0% /dev/shm

# Use% 接近 100% = 磁盘快满了！

# 查找磁盘使用率 > 80% 的挂载点（监控告警脚本常用）
df -h | awk 'NR>1 && $5+0 > 80 {print "警告: " $6 " 使用率 " $5}'

# inode 耗尽排查（df -i 显示 Use% = 100%，但 df -h 还有空间）
df -i | awk 'NR>1 && $5+0 > 80 {print "警告: " $6 " inode使用率 " $5}'
```

## 6.2 du - 目录磁盘占用

```bash
du -sh /var/log           # 统计目录总大小（s=汇总，h=人性化）
du -sh /var/log/*         # 列出子目录各自大小
du -h --max-depth=1 /var  # 只深入1层
du -h --max-depth=2 /     # 深入2层
du -ah /var/log/ | sort -rh | head -20  # 按大小倒序，前20个文件/目录

# 查找最大的目录/文件（线上磁盘满了必用）
du -sh /* 2>/dev/null | sort -rh | head -10    # 根目录各子目录大小
du -sh /var/* 2>/dev/null | sort -rh | head -10
find / -size +100M -type f 2>/dev/null | xargs ls -lh | sort -k5 -rh

# du vs df 不一致的原因：
# 1. 文件被删除但进程还持有文件描述符（lsof | grep deleted）
# 2. 硬链接（一个文件有多个名字，du 计算多次）
# 3. 稀疏文件（占用空间少于文件大小）
```

## 6.3 fdisk / lsblk / blkid

```bash
# ---- lsblk - 块设备树形展示 ----
lsblk                  # 列出所有块设备
lsblk -f               # 显示文件系统信息
lsblk -d               # 只显示磁盘（不显示分区）
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT,FSTYPE,UUID  # 指定字段

# 示例输出：
# NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
# sda      8:0    0  100G  0 disk
# ├─sda1   8:1    0   50G  0 part /
# └─sda2   8:2    0   50G  0 part /data
# sdb      8:16   0  200G  0 disk

# ---- blkid - 查看块设备 UUID 和文件系统类型 ----
blkid                   # 列出所有块设备的 UUID 和类型
blkid /dev/sda1         # 特定设备
# /dev/sda1: UUID="xxxx" TYPE="ext4" PARTUUID="yyyy"

# ---- fdisk - 分区管理工具 ----
fdisk -l                # 列出所有磁盘和分区信息（只读，推荐）
fdisk -l /dev/sdb       # 特定磁盘信息
fdisk /dev/sdb          # 交互式分区（危险！会修改分区表）
# 交互命令：p=打印, n=新建, d=删除, w=保存, q=退出, m=帮助

# parted（更现代，支持 GPT）
parted -l               # 列出所有磁盘和分区
```

## 6.4 mount / umount

```bash
# 查看当前挂载
mount                              # 所有挂载信息
mount | column -t                  # 格式化输出
mount | grep /dev/sdb              # 过滤特定设备
cat /proc/mounts                   # 更可靠的挂载信息

# 挂载
mount /dev/sdb1 /mnt/data          # 挂载到目录
mount -t ext4 /dev/sdb1 /mnt/data  # 指定文件系统类型
mount -o ro /dev/sdb1 /mnt/        # 只读挂载
mount -o remount,rw /              # 将根目录重新挂载为读写（修复只读问题）
mount -o loop image.iso /mnt/cdrom # 挂载 ISO 文件（loop设备）

# 卸载
umount /mnt/data                   # 卸载
umount -f /mnt/data                # 强制卸载（慎用）
umount -l /mnt/data                # 懒卸载（等待进程完成后卸载）
# 如果提示 "device is busy"，查看哪个进程在使用：
lsof /mnt/data                     # 找出占用进程
fuser -m /mnt/data                 # 同上

# 永久挂载（/etc/fstab）
# UUID=xxxx   /data   ext4   defaults,nofail   0   2
# 格式：设备  挂载点  文件系统  选项  dump  fsck顺序
# nofail：挂载失败不影响系统启动（重要！）

# 验证 fstab 语法
mount -a                           # 挂载 fstab 中所有未挂载的条目
```

## 6.5 iostat - IO 统计

```bash
iostat                   # CPU 和 IO 统计（一次）
iostat 1                 # 每秒刷新
iostat 1 10              # 每秒一次，共10次
iostat -x                # 扩展统计（更详细）
iostat -xz 1             # 扩展统计，不显示空活动的设备
iostat -d /dev/sdb 1     # 只显示特定设备
iostat -m                # 以 MB/s 显示（默认是 KB/s）

# 扩展格式输出（-x）：
# Device  r/s  w/s  rkB/s  wkB/s  rrqm/s  wrqm/s  r_await  w_await  aqu-sz  %util
# sda    10.0 50.0  160.0 4000.0     0.0     5.0     2.50     5.00    0.27   5.00

# 关键指标解读：
# r/s：每秒读请求数
# w/s：每秒写请求数
# rkB/s / wkB/s：每秒读/写字节数（KB）
# r_await / w_await：读/写请求的平均等待时间（ms）
#   < 5ms = 很好（SSD），< 20ms = 正常（HDD），> 50ms = 有问题
# aqu-sz（avgqu-sz）：平均 IO 队列长度
#   > 1 说明 IO 在排队，磁盘压力大
# %util：磁盘繁忙率（类似 CPU 使用率）
#   > 80% = 磁盘繁忙，> 95% = 磁盘饱和，成为瓶颈

# 实际排查：
# 1. %util 高 = 磁盘 IO 成为瓶颈
# 2. r_await / w_await 高 = 磁盘响应慢（可能是 HDD 随机 IO 或故障）
# 3. aqu-sz 大 = 请求积压，磁盘处理不过来
```

## 6.6 iotop - 进程 IO 监控

```bash
# 安装
yum install -y iotop
apt install -y iotop

iotop                    # 实时监控（需要 root 权限）
iotop -o                 # 只显示有 IO 的进程（-o = only）
iotop -a                 # 累计 IO 统计（-a = accumulated）
iotop -b -n 5            # 批处理模式，5次（适合脚本）
iotop -p 1234            # 只监控特定 PID
iotop -u alice           # 只监控特定用户

# 交互快捷键：
# o：切换显示只有 IO 的进程
# a：切换累计/当前模式
# p：切换进程/线程视图
# q：退出
```

## 6.7 inode 耗尽问题排查

```bash
# 问题现象：df -h 显示磁盘有空间，但无法创建文件，报 "No space left on device"

# 步骤1：确认是 inode 耗尽
df -i
# Filesystem      Inodes  IUsed  IFree IUse% Mounted on
# /dev/sda1      6553600 6553600     0  100% /         <-- IUse% 100%!

# 步骤2：找到消耗 inode 最多的目录
find / -xdev -printf '%h\n' 2>/dev/null | sort | uniq -c | sort -rn | head -20
# 解释：-xdev 不跨越挂载点，-printf '%h\n' 打印文件的目录名

# 步骤3：查看目录下文件数量
for dir in /var/spool/postfix/maildrop /tmp /var/log; do
    count=$(find $dir -type f 2>/dev/null | wc -l)
    echo "$count $dir"
done | sort -rn

# 常见原因：
# 1. /var/spool/postfix/ 大量邮件堆积（MTA 故障）
# 2. /tmp 大量小文件（Java 应用写临时文件不清理）
# 3. 应用 session 文件（PHP session）
# 4. 日志文件过多（每次请求一个文件）

# 解决：清理大量小文件
ls /path/to/dir/ | wc -l             # 统计文件数
find /path/to/dir/ -type f -delete   # 删除（注意备份！）
# 或使用 rsync 清空（比 rm * 更快，不会报"参数列表过长"）
mkdir /tmp/empty_dir
rsync -a --delete /tmp/empty_dir/ /path/to/full_dir/
```

## 6.8 磁盘性能测试

```bash
# ---- dd（简单顺序读写测试）----
# 写测试（测试写速度）
dd if=/dev/zero of=/tmp/testfile bs=1G count=1 oflag=direct
# oflag=direct 绕过 page cache，测量真实写速度
# conv=fdatasync 写后调用 fdatasync 确保数据到磁盘

# 读测试（先清除缓存）
echo 3 > /proc/sys/vm/drop_caches    # 清除 page cache
dd if=/tmp/testfile of=/dev/null bs=1G count=1 iflag=direct

# ---- fio（专业 IO 测试工具）----
yum install -y fio

# 顺序读
fio --name=seqread --rw=read --bs=1M --size=1G --runtime=60 \
    --numjobs=1 --iodepth=1 --filename=/tmp/testfile --direct=1

# 随机读（4K IOPS，数据库场景）
fio --name=randread --rw=randread --bs=4k --size=1G --runtime=60 \
    --numjobs=4 --iodepth=32 --filename=/tmp/testfile --direct=1

# 混合读写（7:3）
fio --name=mixed --rw=randrw --rwmixread=70 --bs=4k --size=1G \
    --runtime=60 --numjobs=4 --iodepth=32 --filename=/tmp/testfile --direct=1

# 关键指标：
# IOPS（每秒 IO 操作数）：SSD > 10万，普通 HDD < 200（随机）
# 带宽（MB/s）：SSD > 500MB/s，HDD ~100MB/s（顺序）
# 延迟（latency）：SSD < 1ms，HDD < 10ms（正常）
```


---

# Part 7: 网络管理

## 7.1 ip / ifconfig

```bash
# ---- ip 命令（现代，推荐）----
ip addr                         # 查看所有网卡地址
ip addr show eth0               # 查看特定网卡
ip addr add 192.168.1.100/24 dev eth0  # 临时添加 IP
ip addr del 192.168.1.100/24 dev eth0  # 删除 IP
ip link set eth0 up             # 启用网卡
ip link set eth0 down           # 禁用网卡
ip route show                   # 查看路由表
ip route add default via 192.168.1.1   # 添加默认路由
ip neigh                        # ARP 表

# ---- ifconfig（传统）----
ifconfig                        # 查看所有活跃网卡
ifconfig -a                     # 包含未启用的网卡
ifconfig eth0                   # 查看特定网卡
ifconfig eth0 up/down           # 启用/禁用

# 网卡 RX/TX errors 非0要关注（硬件/驱动问题）
```

## 7.2 ss / netstat

```bash
# ---- ss（Socket Statistics，现代推荐）----
ss -tnlp                        # TCP 监听端口 + 进程（最常用）
ss -anp                         # 所有 socket + 进程
ss -s                           # 统计摘要（各状态连接数）
ss -tn state ESTABLISHED        # 只看 ESTABLISHED 状态
ss -tn state TIME-WAIT          # 只看 TIME_WAIT 状态

# 统计 TIME_WAIT 数量（判断短连接是否过多）
ss -tn state TIME-WAIT | wc -l

# ---- netstat（传统）----
netstat -tnlp                   # TCP 监听端口 + 进程
netstat -anp                    # 所有连接 + 进程
netstat -s                      # 各协议统计摘要

# 统计各 TCP 状态数量
netstat -an | awk '/^tcp/ {count[$6]++} END {for (s in count) print s, count[s]}' | sort

# TCP 状态说明：
# LISTEN       等待连接
# ESTABLISHED  连接已建立（正常状态）
# TIME_WAIT    等待2MSL（短连接频繁时会很多）
# CLOSE_WAIT   等待本地应用关闭（过多说明应用未关闭连接，是代码 bug！）
# SYN_RECV     三次握手第二步（syn flood 攻击时会大量出现）
```

## 7.3 ping / traceroute / mtr

```bash
ping -c 4 8.8.8.8               # 发送4个包
ping -s 1400 8.8.8.8            # 指定包大小（测试 MTU）
traceroute -n 8.8.8.8           # 追踪路由，不解析主机名
mtr --report 8.8.8.8            # 综合工具：生成报告（推荐）
mtr --report --report-cycles 20 8.8.8.8
# Loss% 高的跳数说明该节点或其上游有丢包问题
```

## 7.4 curl - HTTP 请求工具

```bash
curl -v http://example.com       # 详细输出（包含请求/响应头）
curl -o output.html http://example.com  # 保存到文件
curl -I http://example.com       # 只显示响应头（HEAD请求）
curl -L http://example.com       # 跟随重定向
curl -k https://self-signed.example.com  # 忽略 SSL 证书验证

# 带请求头和认证
curl -H "Content-Type: application/json" \
     -H "Authorization: Bearer token123" \
     http://api.example.com

# POST JSON
curl -X POST http://api.example.com/users \
     -H "Content-Type: application/json" \
     -d '{"name":"alice","email":"alice@example.com"}'

# 文件上传
curl -X POST http://example.com/upload -F "file=@/path/to/file.txt"

# 认证
curl -u admin:password http://example.com  # Basic Auth

# 显示响应时间（排查接口耗时）
curl -w "\nTotal: %{time_total}s\nDNS: %{time_namelookup}s\nConnect: %{time_connect}s\n" \
     -s -o /dev/null http://example.com

# 超时
curl --connect-timeout 5 --max-time 30 http://example.com
```

## 7.5 wget

```bash
wget http://example.com/file.zip         # 下载文件
wget -c http://example.com/file.zip      # 断点续传（-c = continue）
wget -q http://example.com              # 静默模式
wget --no-check-certificate https://...  # 忽略 SSL 证书
wget -i urls.txt                        # 从文件读取 URL 列表批量下载
wget --limit-rate=1M http://example.com/file.zip  # 限速（1MB/s）
```

## 7.6 telnet / nc - 端口连通性测试

```bash
telnet 192.168.1.100 8080    # 测试 TCP 端口连通性
nc -zv 192.168.1.100 8080    # TCP 端口测试（推荐，-z = zero IO，-v = verbose）
nc -zv 192.168.1.100 8080-8090  # 端口范围扫描
nc -w 3 192.168.1.100 8080   # 设置超时（3秒）
nc -zvu 192.168.1.100 53     # UDP 端口测试（-u = UDP）

# 检测端口是否开放（脚本中常用）
nc -zv -w 3 192.168.1.100 8080 && echo "Port open" || echo "Port closed"

# 传输文件
nc -l 8888 > received.txt              # 接收端：监听 8888 端口
nc 192.168.1.100 8888 < file.txt      # 发送端：将文件发送到远端
```

## 7.7 tcpdump - 网络抓包分析

```bash
tcpdump -i eth0                  # 监控 eth0 网卡（需要 root）
tcpdump -i eth0 -nn              # 不解析主机名和端口名
tcpdump -i eth0 -c 100           # 只捕获100个包
tcpdump -i eth0 -A               # 以 ASCII 格式打印包内容

# 保存/读取文件
tcpdump -w /tmp/capture.pcap -s 0  # 保存到文件（-s 0 = 完整包）
tcpdump -r /tmp/capture.pcap       # 读取文件分析

# 过滤表达式
tcpdump -i eth0 host 192.168.1.100           # 特定主机
tcpdump -i eth0 tcp port 8080               # TCP 8080
tcpdump -i eth0 "host 192.168.1.100 and port 8080"
tcpdump -i eth0 "port 80 or port 443"
tcpdump -i eth0 "not port 22"               # 排除 SSH

# 实用场景
# 抓取 Java 应用与数据库之间的通信
tcpdump -i lo -nn -s 0 -w /tmp/db.pcap port 3306

# 分析 TCP 四次挥手（TIME_WAIT 问题）
tcpdump -i eth0 -nn 'port 8080 and (tcp[tcpflags] & tcp-fin) != 0'
```

## 7.8 nslookup / dig - DNS 查询

```bash
nslookup google.com                      # 查询域名解析
nslookup google.com 8.8.8.8             # 使用指定 DNS 服务器

dig google.com                           # 查询 A 记录
dig google.com @8.8.8.8                 # 使用指定 DNS 服务器
dig google.com MX                        # 查询 MX 记录
dig +short google.com                    # 只显示 IP
dig +trace google.com                    # 追踪 DNS 解析过程（从根服务器开始）
dig -x 8.8.8.8                          # 反向查询（PTR 记录）
```

## 7.9 iptables / firewalld - 防火墙

```bash
# ---- firewalld（CentOS 7+ 推荐）----
systemctl status firewalld    # 查看状态
firewall-cmd --list-all                  # 查看所有规则
firewall-cmd --list-ports                # 查看开放的端口

# 开放端口
firewall-cmd --permanent --add-port=8080/tcp
firewall-cmd --reload

# 关闭端口
firewall-cmd --permanent --remove-port=8080/tcp
firewall-cmd --reload

# ---- iptables（传统，底层）----
iptables -L -n -v --line-numbers        # 查看所有规则

# 允许入站
iptables -I INPUT -p tcp --dport 8080 -j ACCEPT

# 拒绝特定 IP 的入站
iptables -I INPUT -s 10.0.0.100 -j DROP

# 只允许特定 IP 访问 MySQL
iptables -I INPUT -p tcp --dport 3306 -s 192.168.1.0/24 -j ACCEPT
iptables -A INPUT -p tcp --dport 3306 -j DROP

# 保存规则（重启后生效）
service iptables save                    # CentOS
iptables-save > /etc/iptables/rules.v4  # Ubuntu
```

## 7.10 网络性能测试（iperf3）

```bash
# 安装
yum install -y iperf3

# 服务端
iperf3 -s                   # 启动服务端（监听 5201 端口）

# 客户端
iperf3 -c server_ip         # TCP 带宽测试（默认10秒）
iperf3 -c server_ip -t 60   # 测试 60 秒
iperf3 -c server_ip -P 4    # 4 个并行连接
iperf3 -c server_ip -u -b 1G  # UDP 测试，带宽1G

# 输出示例：
# [ ID] Interval    Transfer     Bitrate
# [  5] 0.00-10.00  11.2 GBytes  9.58 Gbits/sec  sender
```

## 7.11 /proc/net/tcp 状态统计

```bash
# TCP 连接状态码含义：
# 01=ESTABLISHED 02=SYN_SENT 03=SYN_RECV 04=FIN_WAIT1
# 05=FIN_WAIT2  06=TIME_WAIT 07=CLOSE    08=CLOSE_WAIT
# 09=LAST_ACK   0A=LISTEN   0B=CLOSING

# 统计各状态数量
awk 'NR>1 {
    state=$4;
    if (state=="01") s="ESTABLISHED";
    else if (state=="06") s="TIME_WAIT";
    else if (state=="0A") s="LISTEN";
    else if (state=="08") s="CLOSE_WAIT";
    else s=state;
    count[s]++
} END {for (s in count) print count[s], s}' /proc/net/tcp | sort -rn
```


---

# Part 8: 系统日志

## 8.1 /var/log/ 目录

```bash
/var/log/messages      # 系统日志（CentOS/RHEL）
/var/log/syslog        # 系统日志（Ubuntu/Debian）
/var/log/kern.log      # 内核日志（OOM/硬件错误/驱动问题）
/var/log/auth.log      # 认证日志（Ubuntu，SSH登录/sudo/PAM）
/var/log/secure        # 认证日志（CentOS/RHEL）
/var/log/dmesg         # 内核启动信息（硬件检测）
/var/log/cron          # 定时任务日志（CentOS）
/var/log/nginx/        # Nginx 日志（access.log / error.log）
/var/log/mysql/        # MySQL 日志
/var/log/docker/       # Docker 日志

# 查看内核日志
dmesg | grep -i error  # 过滤错误
dmesg | tail -50       # 最近的内核消息
dmesg --level=err,warn # 只显示错误和警告

# 查看登录历史
last                   # 查看登录历史
lastlog                # 查看所有用户最后登录
lastb                  # 查看失败的登录尝试（需要 root）
```

## 8.2 journalctl - systemd 日志

```bash
journalctl -f                   # 实时监控（类似 tail -f）
journalctl -u nginx              # nginx 服务的日志
journalctl -u nginx -f           # 实时监控 nginx 日志
journalctl -u myapp.service -n 200 --no-pager  # 查看服务最近200行日志

# 按时间过滤
journalctl --since "2024-01-01 10:00" --until "2024-01-01 11:00"
journalctl --since "1 hour ago"
journalctl --since yesterday

# 按优先级
journalctl -p err                # 只看错误
journalctl -p warning            # 警告及以上
# 0=emerg, 1=alert, 2=crit, 3=err, 4=warning, 5=notice, 6=info, 7=debug

# 按行数
journalctl -n 50                 # 显示最后50行

# 内核日志
journalctl -k                    # 等同于 dmesg

# 磁盘管理
journalctl --disk-usage          # 查看日志占用空间
journalctl --vacuum-size=500M    # 清理旧日志，保留 500M
journalctl --vacuum-time=30d     # 清理 30 天前的日志

# 查看最近的系统错误
journalctl -p err --since "1 hour ago"
```

## 8.3 logrotate - 日志轮转

```bash
# 配置文件位置
/etc/logrotate.conf            # 主配置文件
/etc/logrotate.d/              # 各应用的配置文件

# 示例配置（/etc/logrotate.d/myapp）：
# /var/log/myapp/*.log {
#     daily                    # 每天轮转
#     rotate 30                # 保留30个旧文件
#     compress                 # 压缩旧文件（gzip）
#     delaycompress            # 延迟压缩（下一次轮转时再压缩）
#     missingok                # 文件不存在不报错
#     notifempty               # 空文件不轮转
#     create 640 root adm      # 新建日志文件的权限/所有者
#     sharedscripts            # 多个文件时只执行一次 postrotate
#     postrotate
#         /usr/bin/kill -USR1 $(cat /var/run/nginx.pid) 2>/dev/null || true
#     endscript
# }

# 手动触发（测试）
logrotate -d /etc/logrotate.d/myapp    # 模拟（不实际执行，-d = debug）
logrotate -f /etc/logrotate.d/myapp    # 强制执行（-f = force）
```

## 8.4 rsyslog - 日志集中收集

```bash
# rsyslog 支持将日志发送到远程服务器
# 配置文件：/etc/rsyslog.conf 和 /etc/rsyslog.d/

# 将日志发送到远程服务器（在 /etc/rsyslog.d/remote.conf 中添加）：
# *.*    @@logserver:514      # TCP 传输（@@ = TCP，@ = UDP）
# *.*    @logserver:514       # UDP 传输

# 接收远程日志（日志服务器配置）
# 在 /etc/rsyslog.conf 中取消注释：
# $ModLoad imtcp
# $InputTCPServerRun 514

systemctl restart rsyslog
```

## 8.5 应用日志最佳实践

```bash
# ---- 日志格式建议（推荐 JSON 格式，便于 ELK 解析）----
# {"timestamp":"2024-01-01T10:00:00.000Z","level":"ERROR","traceId":"xxx",
#  "userId":"123","message":"Payment failed","exception":"..."}

# 使用 MDC（Mapped Diagnostic Context）传递链路追踪信息：
# Java 代码中：
# MDC.put("traceId", UUID.randomUUID().toString());
# MDC.put("userId", currentUser.getId());

# ---- 日志级别建议 ----
# ERROR：影响功能的错误（需要立即处理）
# WARN：潜在问题（需要关注）
# INFO：正常流程关键节点（用户登录/下单/支付等业务事件）
# DEBUG：调试信息（生产环境默认关闭）
# TRACE：非常详细的调试（生产环境不开启）

# ---- 日志切割配置（Logback）----
# <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
#   <fileNamePattern>/var/log/myapp/app.%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
#   <timeBasedFileNamingAndTriggeringPolicy class="...SizeAndTimeBasedFNATP">
#     <maxFileSize>200MB</maxFileSize>
#   </timeBasedFileNamingAndTriggeringPolicy>
#   <maxHistory>30</maxHistory>
#   <totalSizeCap>10GB</totalSizeCap>
# </rollingPolicy>

# ---- 常用日志分析命令 ----
# 查看某时间段的错误
grep "2024-01-01 10:" /var/log/app/app.log | grep "ERROR"

# 统计每分钟的错误数
grep "ERROR" app.log | awk '{print $1, substr($2,1,5)}' | uniq -c | sort -rn | head -20

# 查找耗时超过 1 秒的接口请求
grep "cost=" app.log | awk -F'cost=' '{print $2}' | awk '$1>1000 {print $0}' | sort -rn | head -10
```

## 8.6 ELK 日志中心接入

```bash
# ELK = Elasticsearch + Logstash + Kibana

# ---- Filebeat（轻量日志采集，推荐）----
# 配置 /etc/filebeat/filebeat.yml：
# filebeat.inputs:
# - type: log
#   enabled: true
#   paths:
#     - /var/log/myapp/*.log
#   fields:
#     app: myapp
#     env: production
#   fields_under_root: true
#   json.keys_under_root: true      # JSON 格式日志解析
#   json.overwrite_keys: true
#
# output.elasticsearch:
#   hosts: ["elasticsearch:9200"]
#   index: "myapp-logs-%{+yyyy.MM.dd}"

systemctl start filebeat
systemctl enable filebeat

# ---- 直接用 Java 日志框架对接 ES ----
# 使用 logstash-logback-encoder + appender 直接写 ES/Logstash
# pom.xml 依赖：net.logstash.logback:logstash-logback-encoder:7.3
```


---

# Part 9: 性能分析实战

## 9.1 系统性能分析工具图谱

```
                    性能分析工具层次图
  ┌────────────────────────────────────────────────────────────────┐
  │ 应用层   │ JVM: jstat jmap jstack jcmd Arthas async-profiler    │
  │          │ Java GC: GCEasy MAT VisualVM JMC                     │
  ├──────────┼──────────────────────────────────────────────────── │
  │ 系统调用 │ strace ltrace                                        │
  ├──────────┼──────────────────────────────────────────────────── │
  │ CPU      │ top htop mpstat pidstat perf sar                     │
  ├──────────┼──────────────────────────────────────────────────── │
  │ 内存     │ free vmstat pmap /proc/meminfo valgrind              │
  ├──────────┼──────────────────────────────────────────────────── │
  │ 磁盘 IO  │ iostat iotop blktrace fio dd lsblk                   │
  ├──────────┼──────────────────────────────────────────────────── │
  │ 网络     │ ss netstat tcpdump ping mtr iperf3 sar               │
  ├──────────┼──────────────────────────────────────────────────── │
  │ 内核     │ /proc /sys dmesg sysctl perf-events bpftrace         │
  └────────────────────────────────────────────────────────────────┘
```

## 9.2 CPU 性能问题排查流程

```bash
# 1. 确认全局 CPU 状况
top           # 看 us/sy/wa/si 哪个高
mpstat -P ALL 1  # 查看各核心负载是否均衡

# 2. 找出高 CPU 进程
ps aux --sort=-%cpu | head -10

# 3. 定位具体情况：
# 情况A：us 高（用户态）
#   - 找 CPU 最高的进程（ps aux --sort=-%cpu）
#   - 若是 Java：用 top -Hp <pid> 找高 CPU 线程 → jstack 分析
#   - 可能原因：死循环、GC 频繁、正则回溯、JSON 序列化等

# 情况B：sy 高（内核态）
#   - 检查上下文切换：vmstat 1 | awk '{print $12}' 或 pidstat -w 1
#   - 检查系统调用：strace -c -p <pid>
#   - 可能原因：过多线程切换、频繁 fork/exec、大量 IO 系统调用

# 情况C：wa 高（IO等待）
#   - 检查磁盘：iostat -x 1（看 %util / await）
#   - 找 IO 高的进程：iotop -o
#   - 可能原因：磁盘慢、日志大量写入、内存不足导致频繁 swap

# 情况D：si 高（软中断）
#   - 通常是网络包处理压力大
#   - 查看网卡统计：sar -n DEV 1 或 ifconfig / ip -s link
#   - 考虑开启网卡多队列
```

## 9.3 内存性能问题排查流程

```bash
# 1. 确认内存使用情况
free -h                  # 看 available 是否快耗尽
cat /proc/meminfo        # 看 MemAvailable / SwapFree
vmstat 1                 # 看 si/so（swap 使用量，非0说明内存紧张）

# 2. 找出内存消耗大的进程
ps aux --sort=-%mem | head -10

# 3. 分析 Java 进程内存
jstat -gcutil <pid> 2000  # 每2秒查看 GC 状态（O区是否持续增长）
jmap -heap <pid>          # 堆内存摘要
jmap -histo <pid> | head -30  # 对象统计（找最多的类）

# 4. 生成 heap dump 分析
jmap -dump:format=b,file=/tmp/heap.hprof <pid>
# 用 MAT 打开分析 Dominator Tree / Leak Suspects

# 5. 堆外内存泄漏排查
# 现象：Java 进程 RSS 远大于 -Xmx 值
pmap -x <pid> | tail -1  # 看 total kB
# 可能原因：Metaspace 泄漏/DirectBuffer/JNI/Netty 等

# 6. OOM 后自动 dump（提前配置）
java -XX:+HeapDumpOnOutOfMemoryError \
     -XX:HeapDumpPath=/var/log/heap.hprof \
     -jar app.jar
```

## 9.4 IO 性能问题排查流程

```bash
# 1. 确认 IO 瓶颈存在
iostat -xz 1                 # 看 %util / await（重点！）
# %util > 80% 或 await > 20ms，说明磁盘有压力

# 2. 找出 IO 高的进程
iotop -o                     # 实时 IO 高进程
pidstat -d 1                 # 进程 IO 统计

# 3. 定位具体文件
lsof -p <pid> | grep -E 'REG|CHR'  # 进程打开的文件
strace -p <pid> -e trace=read,write,open  # 追踪 IO 系统调用

# 4. 常见原因与解决：
# 原因1：Java 日志频繁同步写入（每条日志 fsync）
#   解决：异步日志（AsyncAppender），减少 IO 等待
# 原因2：数据库慢查询导致大量磁盘读取
#   解决：优化 SQL，加索引，增大 buffer_pool
# 原因3：GC 频繁导致大量 JVM 写操作
#   解决：调整堆大小，减少 GC 频率
```

## 9.5 网络性能问题排查流程

```bash
# 1. 基本连通性
ping -c 10 target_ip             # 丢包率 / 延迟
mtr --report target_ip           # 逐跳检测

# 2. 连接状态分析
ss -s                             # 连接统计摘要
ss -tn state TIME-WAIT | wc -l   # TIME_WAIT 数量
ss -tn state CLOSE-WAIT | wc -l  # CLOSE_WAIT 数量
# TIME_WAIT 多：短连接频繁，考虑 tcp_tw_reuse / keepalive
# CLOSE_WAIT 多：应用未及时关闭连接（代码 bug！）

# 3. 网络带宽测试
iperf3 -c server_ip

# 4. 抓包分析
tcpdump -i eth0 -w /tmp/cap.pcap port 8080 -c 1000

# 5. 常见网络问题：
# 连接超时：检查防火墙（firewall-cmd / iptables -L）
# 连接数打满：ulimit -n（增大文件描述符限制）
# 高延迟：检查 TCP Nagle 算法（Java NIO 可能需要 TCP_NODELAY）
```

## 9.6 Java 进程排查专题

### jps - 查看 Java 进程

```bash
jps                    # 列出所有 Java 进程（PID + 主类名）
jps -l                 # 显示完整类名/jar路径
jps -v                 # 显示 JVM 参数
jps -m                 # 显示传给 main 方法的参数
```

### jstat - GC 统计

```bash
# gcutil：GC 百分比统计（最常用）
jstat -gcutil 1234               # 输出一次
jstat -gcutil 1234 1000          # 每 1 秒输出
jstat -gcutil 1234 1000 10       # 每 1 秒输出，共10次

# 输出示例（gcutil）：
#   S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT
#  0.00  99.00  50.00  80.00  95.00  90.00    500    2.000    10    5.000    7.000

# 字段含义：
# S0/S1：Survivor 0/1 空间使用率（%）
# E：Eden 空间使用率（%）
# O：Old Gen 使用率（%）  -- 持续增长且 FGC 不回收 = 内存泄漏！
# M：Metaspace 使用率（%）
# YGC：Young GC 次数
# YGCT：Young GC 累计时间（秒）
# FGC：Full GC 次数
# FGCT：Full GC 累计时间（秒）
# GCT：GC 总时间（秒）

jstat -gc 1234 1000      # 详细 GC 统计（显示各内存区域 C=容量 U=已用，单位KB）
jstat -gcnew 1234        # 年轻代统计
jstat -gcold 1234        # 老年代统计
```

### jmap - 堆内存分析

```bash
# 查看堆摘要（会触发 STW，生产慎用）
jmap -heap 1234

# 对象统计（按类名统计实例数和内存）
jmap -histo 1234 | head -30
jmap -histo:live 1234 | head -30   # live：先触发一次 Full GC 再统计

# 生成 heap dump
jmap -dump:format=b,file=/tmp/heap.hprof 1234
jmap -dump:live,format=b,file=/tmp/heap.hprof 1234  # 只 dump 存活对象

# 自动生成 dump（OOM 时）
java -XX:+HeapDumpOnOutOfMemoryError \
     -XX:HeapDumpPath=/var/log/java/ \
     -jar app.jar
```

### jstack - 线程 dump

```bash
jstack 1234                  # 输出线程 dump（每个线程的调用栈）
jstack -l 1234               # 包含锁信息（lock owners）
jstack -F 1234               # 强制 dump（进程无响应时）

# 线程状态说明：
# RUNNABLE：正在运行或等待 CPU 调度
# BLOCKED：等待获取 monitor 锁（等待 synchronized 锁）
# WAITING：无限等待（Object.wait() / Thread.join() / LockSupport.park()）
# TIMED_WAITING：有时限等待（Thread.sleep() / Object.wait(timeout)）

# 检测死锁
jstack 1234 | grep -A 5 "deadlock"
# jstack 会在末尾自动显示死锁信息：
# Found one Java-level deadlock:
# "Thread-1": waiting to lock ... which is held by "Thread-0"
# "Thread-0": waiting to lock ... which is held by "Thread-1"

# 统计各状态线程数
jstack 1234 | grep "java.lang.Thread.State" | sort | uniq -c
```

### jcmd - 综合诊断工具

```bash
jcmd                          # 列出所有 Java 进程
jcmd 1234 help                # 查看可用命令

jcmd 1234 Thread.print > /tmp/threads.txt  # 线程 dump
jcmd 1234 GC.heap_dump /tmp/heap.hprof     # heap dump
jcmd 1234 GC.run                           # 触发 GC
jcmd 1234 VM.flags                         # JVM 参数
jcmd 1234 VM.system_properties             # 系统属性
jcmd 1234 VM.info                          # JVM 详细信息
```

### Arthas - Java 线上诊断工具

Arthas 是阿里巴巴开源的 Java 诊断工具，无需重启应用即可进行诊断。

```bash
# 安装
curl -L https://arthas.aliyun.com/arthas-boot.jar -o arthas-boot.jar
java -jar arthas-boot.jar     # 启动后选择要诊断的 Java 进程

# ---- 常用命令 ----

# dashboard：系统实时数据面板（类似 top）
dashboard

# thread：线程信息
thread                      # 所有线程列表
thread -b                   # 找出阻塞其他线程的线程
thread --state BLOCKED      # 只显示 BLOCKED 状态线程
thread -n 5                 # 最忙的5个线程

# jad：反编译
jad com.example.OrderService         # 反编译类
jad com.example.OrderService processOrder  # 只反编译一个方法

# watch：观察方法调用（入参/返回值/异常）
watch com.example.OrderService processOrder '{params,returnObj,throwExp}' -x 2
# -x：展开深度  条件过滤：
watch com.example.OrderService processOrder '{params,returnObj}' 'params[0].orderId==100'

# trace：方法调用链路追踪（找耗时方法）
trace com.example.OrderService processOrder
trace com.example.OrderService processOrder '#cost>100'  # 只显示耗时>100ms的

# monitor：方法执行监控（成功/失败次数、平均耗时）
monitor com.example.OrderService processOrder -c 5  # 每5秒统计一次

# stack：查看方法被哪些调用路径调用
stack com.example.OrderService processOrder

# tt（time tunnel）：记录方法调用记录，可以重放
tt -t com.example.OrderService processOrder  # 开始记录
tt -l                                         # 列出记录
tt -i 1000 -p                                 # 重放第1000次调用

# ognl：执行 OGNL 表达式（动态修改变量/调用方法）
ognl "@java.lang.System@currentTimeMillis()"
ognl "@com.example.config.Config@instance.getConfig()"
# 动态修改开关
ognl "@com.example.config.FeatureFlag@DEBUG_MODE = true"

# sc/sm：搜索类/搜索方法
sc com.example.*              # 搜索类
sc -d com.example.OrderService  # 类详情（包含类加载器等）
sm -d com.example.OrderService processOrder  # 方法详情

# profiler：采样火焰图（基于 async-profiler）
profiler start               # 开始采样
profiler stop --file /tmp/flame.html  # 停止并生成火焰图

# 退出 Arthas（不影响应用）
quit         # 退出 Arthas，不影响目标 JVM
stop         # 完全停止 Arthas
```

### 线上 CPU 100% 排查完整步骤

```bash
#!/bin/bash
# 步骤 1：确认 CPU 高的进程
top         # 按 P 排序（CPU降序），记录 Java 进程 PID=1234

# 步骤 2：找到消耗 CPU 的线程
top -Hp 1234    # 显示进程的所有线程，记录高CPU线程 TID=5678
# 或
ps -mp 1234 -o THREAD,tid,time | sort -rn | head -10

# 步骤 3：将线程 TID 转换为十六进制
printf '%x\n' 5678   # 输出：162e

# 步骤 4：用 jstack 获取线程 dump
jstack 1234 > /tmp/jstack.txt

# 步骤 5：在 jstack 中搜索该线程
grep -A 30 "nid=0x162e" /tmp/jstack.txt
# nid 是十六进制的线程 ID，找到后就能看到对应的调用栈！

# 完整一键脚本：
PID=$(pgrep -f app.jar)
echo "Java PID: $PID"
HIGH_TID=$(ps -mp $PID -o THREAD,tid,time | sort -rn | sed -n '2p' | awk '{print $2}')
HEX_TID=$(printf '%x' $HIGH_TID)
echo "High CPU Thread TID: $HIGH_TID (0x$HEX_TID)"
jstack $PID | grep -A 30 "nid=0x$HEX_TID"
```

### 线上 OOM 排查完整步骤

```bash
# 步骤1：确认 OOM 类型
# Java 堆 OOM：java.lang.OutOfMemoryError: Java heap space
# Metaspace OOM：java.lang.OutOfMemoryError: Metaspace
# 系统 OOM（被 OOM Killer 杀死）：dmesg | grep "killed process"
grep -i "OutOfMemoryError" /var/log/app/app.log | tail -20
dmesg | grep -i "killed process" | tail -10

# 步骤2：获取 heap dump
PID=$(pgrep -f "app.jar")
jmap -dump:format=b,file=/tmp/heap_$(date +%Y%m%d_%H%M%S).hprof $PID

# 步骤3：快速查看占用内存最多的类
jmap -histo $PID | head -25

# 步骤4：用 MAT 分析 heap dump
# 1. scp 将 hprof 文件下载到本地
# 2. Eclipse MAT 打开（File -> Open Heap Dump）
# 3. 查看 Dominator Tree（占用内存最多的对象树）
# 4. 查看 Leak Suspects Report（自动分析泄漏嫌疑）

# 常见 OOM 原因：
# 1. 大量缓存未设置上限（Map/List 持续增长）
# 2. 数据库查询未分页，全量加载到内存
# 3. 线程 ThreadLocal 未清理（在线程池中 ThreadLocal 不会自动清理）
# 4. 静态集合类持有大量对象
# 5. 内部类持有外部类引用（导致外部类无法被 GC）
```

### GC 日志分析

```bash
# JDK 9+ GC 日志参数
java -Xlog:gc*:file=/var/log/gc.log:time,tags,uptime:filecount=5,filesize=20m \
     -jar app.jar

# JDK 8 GC 日志参数
java -Xloggc:/var/log/gc.log \
     -XX:+PrintGCDetails \
     -XX:+PrintGCDateStamps \
     -XX:+UseGCLogFileRotation \
     -XX:NumberOfGCLogFiles=5 \
     -XX:GCLogFileSize=20m \
     -jar app.jar

# GC 日志示例（G1GC）：
# 2024-01-01T10:00:00.000+0800 GC(100) Pause Young (Normal) (G1 Evacuation Pause)
# 2024-01-01T10:00:00.000+0800 GC(100) Pause Young (Normal) 1024M->512M(2048M) 20.123ms
# 指标：GC前后堆使用量变化 和 GC 停顿时间

# GC 分析工具
# 1. GCEasy（在线）：https://gceasy.io 上传日志文件
# 2. GCViewer（本地）：java -jar gcviewer.jar gc.log
# 3. JVM 自带 JMC（Java Mission Control）：实时监控+飞行记录
```


---

# Part 10: Shell 脚本

## 10.1 Bash 基础语法

```bash
#!/bin/bash
# 变量定义（等号两边不能有空格！）
NAME="hello"
AGE=25
readonly PI=3.14159      # 只读变量

# 变量使用
echo $NAME
echo ${NAME}             # 推荐加花括号（避免歧义）
echo "Hello, ${NAME}!"

# 命令替换
CURRENT_DATE=$(date +%Y-%m-%d)
FILES_COUNT=$(ls -l | wc -l)

# 算术运算
SUM=$((1 + 2))           # $(( )) 表达式（推荐）
let SUM=1+2              # let 命令

# 字符串操作
STR="Hello World"
echo ${#STR}             # 字符串长度：11
echo ${STR:0:5}          # 截取：Hello（从第0位取5个字符）
echo ${STR:6}            # 截取：World（从第6位到末尾）
echo ${STR/World/Shell}  # 替换：Hello Shell（替换第一个）
echo ${STR//l/L}         # 全局替换：HeLLo WorLd

# 引号区别
echo "$NAME"             # 双引号：变量被展开
echo '$NAME'             # 单引号：原样输出，不展开变量

# 特殊变量
echo $0                  # 脚本名
echo $1 $2 $3            # 第1/2/3个参数
echo $#                  # 参数个数
echo $@                  # 所有参数（每个参数分别引用）
echo $*                  # 所有参数（作为一个整体）
echo $?                  # 上一个命令的退出状态（0=成功，非0=失败）
echo $$                  # 当前进程 PID
echo $!                  # 最后一个后台进程 PID
```

## 10.2 条件判断

```bash
# ---- if/elif/else ----
if [ condition ]; then
    # ...
elif [ condition2 ]; then
    # ...
else
    # ...
fi

# 数字比较
[ $a -eq $b ]   # equal（等于）
[ $a -ne $b ]   # not equal（不等于）
[ $a -lt $b ]   # less than（小于）
[ $a -le $b ]   # less or equal（小于等于）
[ $a -gt $b ]   # greater than（大于）
[ $a -ge $b ]   # greater or equal（大于等于）

# 字符串比较
[ "$a" = "$b" ]   # 字符串相等（注意加引号！）
[ "$a" != "$b" ]  # 字符串不相等
[ -z "$a" ]       # 字符串为空（zero length）
[ -n "$a" ]       # 字符串非空（non-zero）
[[ "$a" =~ regex ]]  # 正则匹配（[[ ]] 支持，[ ] 不支持）

# 文件测试
[ -f file ]    # 是普通文件
[ -d dir ]     # 是目录
[ -e path ]    # 路径存在
[ -r file ]    # 可读
[ -w file ]    # 可写
[ -x file ]    # 可执行
[ -s file ]    # 文件非空（大小>0）
[ -L file ]    # 是软链接

# 逻辑运算
[ cond1 ] && [ cond2 ]    # AND
[ cond1 ] || [ cond2 ]    # OR
! [ cond ]                 # NOT
[[ cond1 && cond2 ]]      # AND（双括号内可用 && ||）

# 实例
if [ -f "/etc/nginx/nginx.conf" ]; then
    echo "Nginx config exists"
fi

PORT=8080
if [ $PORT -ge 1024 ]; then
    echo "Non-privileged port"
fi
```

## 10.3 循环

```bash
# ---- for 循环 ----
for item in a b c d e; do
    echo "$item"
done

for i in {1..10}; do
    echo "Number: $i"
done

# C 风格 for
for ((i=0; i<10; i++)); do
    echo "$i"
done

# 遍历文件
for file in /var/log/*.log; do
    echo "Processing $file"
done

# ---- while 循环 ----
count=0
while [ $count -lt 5 ]; do
    echo "count: $count"
    ((count++))
done

# 读取文件每一行
while IFS= read -r line; do
    echo "$line"
done < /etc/passwd

# ---- break / continue ----
for i in {1..10}; do
    if [ $i -eq 5 ]; then
        break       # 跳出循环
    fi
    if [ $((i % 2)) -eq 0 ]; then
        continue    # 跳过本次循环
    fi
    echo "$i"
done
```

## 10.4 函数

```bash
# 定义函数
function greet() {
    local name=$1      # local 定义局部变量
    echo "Hello, ${name}!"
}

# 调用
greet "Alice"

# 返回值（通过 echo + 命令替换）
get_java_version() {
    java -version 2>&1 | awk -F'"' '/version/ {print $2}'
}
VERSION=$(get_java_version)
echo "Java version: $VERSION"

# 函数参数
function backup() {
    local src=$1
    local dest=$2
    local timestamp=$(date +%Y%m%d_%H%M%S)
    tar -czvf "${dest}/backup_${timestamp}.tar.gz" "$src"
    echo "Backup completed"
}
backup /var/www/html /backup
```

## 10.5 数组

```bash
# ---- 普通数组 ----
arr=(one two three four)
echo ${arr[0]}           # 访问第0个元素
echo ${arr[@]}           # 所有元素
echo ${#arr[@]}          # 元素个数
echo ${!arr[@]}          # 所有下标

# 遍历
for item in "${arr[@]}"; do
    echo "$item"
done

# ---- 关联数组（哈希表，Bash 4+）----
declare -A map
map["key1"]="value1"
map["key2"]="value2"
echo ${map["key1"]}      # 访问
for key in "${!map[@]}"; do
    echo "$key = ${map[$key]}"
done
```

## 10.6 错误处理

```bash
#!/bin/bash
set -euo pipefail
# set -e：任何命令返回非零值时立即退出脚本
# set -u：使用未定义的变量时报错
# set -o pipefail：管道中任何命令失败则整个管道失败

# trap：捕获信号和错误
trap 'echo "Error on line $LINENO"; exit 1' ERR
trap 'cleanup' EXIT  # 脚本退出时执行清理

cleanup() {
    echo "Cleaning up..."
    rm -f /tmp/temp_file.$$
}

# 错误处理函数
die() {
    echo "Fatal: $1" >&2
    exit ${2:-1}
}

[ -f /etc/required_file ] || die "Required file not found" 2
```

## 10.7 常用脚本模板

### 日志清理脚本

```bash
#!/bin/bash
# clean_logs.sh - 清理超过指定天数的日志文件

set -euo pipefail
LOG_DIR="${1:-/var/log/app}"
DAYS="${2:-30}"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"
}

[ -d "$LOG_DIR" ] || { log "Directory $LOG_DIR not found"; exit 1; }

log "Starting log cleanup: dir=$LOG_DIR, days=$DAYS"
COUNT=$(find "$LOG_DIR" -name "*.log*" -mtime +$DAYS -type f | wc -l)
log "Found $COUNT files older than $DAYS days"

find "$LOG_DIR" -name "*.log*" -mtime +$DAYS -type f -exec rm -f {} \;
log "Cleanup completed"

# 检查磁盘空间
FREE=$(df -BG "$LOG_DIR" | awk 'NR==2 {print $4}' | tr -d G)
log "Available disk space: ${FREE}GB"
```

### Java 服务健康检查+自动重启脚本

```bash
#!/bin/bash
# health_check.sh - Java 服务健康检查和自动重启

APP_NAME="myapp"
JAR_PATH="/opt/myapp/myapp.jar"
LOG_PATH="/var/log/myapp/app.log"
PID_FILE="/var/run/myapp.pid"
HEALTH_URL="http://localhost:8080/actuator/health"
MAX_RESTART=3
RESTART_INTERVAL=30

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] [$APP_NAME] $*" | tee -a /var/log/myapp/monitor.log
}

start_app() {
    log "Starting application..."
    nohup java -Xms512m -Xmx2g \
               -XX:+HeapDumpOnOutOfMemoryError \
               -XX:HeapDumpPath=/var/log/myapp/ \
               -jar "$JAR_PATH" \
               > "$LOG_PATH" 2>&1 &
    echo $! > "$PID_FILE"
    log "Application started with PID: $(cat $PID_FILE)"
}

is_running() {
    if [ -f "$PID_FILE" ]; then
        local pid=$(cat "$PID_FILE")
        kill -0 "$pid" 2>/dev/null && return 0
    fi
    return 1
}

health_check() {
    local http_code=$(curl -s -o /dev/null -w "%{http_code}" \
                     --connect-timeout 5 --max-time 10 \
                     "$HEALTH_URL")
    [ "$http_code" = "200" ] && return 0 || return 1
}

restart_count=0
while true; do
    if ! is_running; then
        log "Process not running, starting..."
        start_app
        sleep 20
    elif ! health_check; then
        log "Health check failed!"
        ((restart_count++))
        if [ $restart_count -le $MAX_RESTART ]; then
            log "Restarting (attempt $restart_count/$MAX_RESTART)..."
            pid=$(cat "$PID_FILE")
            kill -15 "$pid" 2>/dev/null || true
            sleep 5
            kill -9 "$pid" 2>/dev/null || true
            sleep 2
            start_app
            sleep 20
        else
            log "ERROR: Max restart attempts exceeded!"
            exit 1
        fi
    else
        restart_count=0
        log "Health check OK"
    fi
    sleep $RESTART_INTERVAL
done
```

### 数据库备份脚本

```bash
#!/bin/bash
# db_backup.sh - MySQL 数据库备份

set -euo pipefail
DB_HOST="${DB_HOST:-localhost}"
DB_PORT="${DB_PORT:-3306}"
DB_USER="${DB_USER:-backup}"
DB_PASS="${DB_PASS}"
DB_NAME="${DB_NAME:-mydb}"
BACKUP_DIR="/backup/mysql"
KEEP_DAYS=7

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }

TODAY=$(date +%Y%m%d)
BACKUP_FILE="${BACKUP_DIR}/${DB_NAME}_${TODAY}.sql.gz"
mkdir -p "$BACKUP_DIR"

log "Starting backup: $DB_NAME -> $BACKUP_FILE"
mysqldump \
    --host="$DB_HOST" \
    --port="$DB_PORT" \
    --user="$DB_USER" \
    --password="$DB_PASS" \
    --single-transaction \
    --routines \
    --triggers \
    "$DB_NAME" | gzip > "$BACKUP_FILE"

if [ -s "$BACKUP_FILE" ]; then
    SIZE=$(du -sh "$BACKUP_FILE" | cut -f1)
    log "Backup successful: $BACKUP_FILE ($SIZE)"
else
    log "ERROR: Backup file is empty!"; exit 1
fi

# 清理旧备份
find "$BACKUP_DIR" -name "${DB_NAME}_*.sql.gz" -mtime +$KEEP_DAYS -delete
log "Backup completed"
```


---

# Part 11: 系统调优

## 11.1 内核参数调优（/etc/sysctl.conf）

```bash
# 查看当前参数
sysctl -a                        # 所有参数
sysctl net.ipv4.tcp_max_syn_backlog  # 单个参数

# 临时修改（重启后失效）
sysctl -w net.ipv4.tcp_max_syn_backlog=65535

# 永久修改：编辑 /etc/sysctl.conf，然后执行
sysctl -p   # 应用配置（立即生效）
```

### 网络参数优化

```bash
# ---- /etc/sysctl.conf 网络优化配置 ----

# 增大连接队列
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 65535

# TCP keepalive（减少死连接占用）
net.ipv4.tcp_keepalive_time = 600   # 连接空闲多久后发 keepalive（默认7200秒）
net.ipv4.tcp_keepalive_intvl = 30   # keepalive 探测间隔
net.ipv4.tcp_keepalive_probes = 3   # keepalive 探测失败次数才断开

# TIME_WAIT 优化
net.ipv4.tcp_tw_reuse = 1           # 允许重用 TIME_WAIT 的 socket
net.ipv4.tcp_fin_timeout = 30       # FIN_WAIT2 等待时间（默认60秒）
net.ipv4.tcp_max_tw_buckets = 10000 # 最大 TIME_WAIT socket 数

# 增大端口范围
net.ipv4.ip_local_port_range = 10000 65535

# 接收发送缓冲区优化
net.core.rmem_max = 134217728        # 最大接收缓冲区（128MB）
net.core.wmem_max = 134217728        # 最大发送缓冲区（128MB）
net.ipv4.tcp_rmem = 4096 65536 134217728
net.ipv4.tcp_wmem = 4096 65536 134217728

# 网络包队列
net.core.netdev_max_backlog = 65535

# 应用 sysctl 配置
sysctl -p
```

### 文件描述符限制

```bash
# 查看当前限制
ulimit -n                        # 当前 shell 的文件描述符限制

# 永久修改所有用户（编辑 /etc/security/limits.conf）：
# * soft nofile 65535
# * hard nofile 65535
# root soft nofile 65535
# root hard nofile 65535

# 服务级别限制（systemd service 文件中）
# [Service]
# LimitNOFILE=65535

# 系统全局文件描述符限制
echo "fs.file-max = 2097152" >> /etc/sysctl.conf
sysctl -p
```

### 虚拟内存参数

```bash
# vm.swappiness：swapping 倾向（0-100）
# 0 = 几乎不用 swap，服务器推荐设为1
vm.swappiness = 1

# vm.overcommit_memory：内存分配策略
# 0 = 启发式（默认），1 = 始终允许超分配（Redis 推荐）
vm.overcommit_memory = 1

# vm.dirty_ratio：脏页比例（超此比例阻塞写操作）
vm.dirty_ratio = 10
vm.dirty_background_ratio = 5

echo "vm.swappiness = 1" >> /etc/sysctl.conf
echo "vm.overcommit_memory = 1" >> /etc/sysctl.conf
sysctl -p
```

## 11.2 JVM 参数与 Linux 内存关系

```bash
# JVM 内存结构（Java 8+）：
# 1. Java Heap（-Xms / -Xmx）：老年代 + 年轻代
# 2. Metaspace（-XX:MaxMetaspaceSize）：类元数据（不在堆上）
# 3. Thread Stack（-Xss）：每个线程的栈（默认 512KB~1MB）
# 4. Direct Memory（-XX:MaxDirectMemorySize）：NIO 直接内存
# 5. Code Cache（-XX:ReservedCodeCacheSize）：JIT 编译代码
# 进程实际 RSS = Heap + Metaspace + Thread*Xss + DirectMemory + CodeCache + ...
# 因此 RSS 远大于 -Xmx 是正常的！

# 常用生产 JVM 参数（以 4G 内存机器为例）
java \
  -Xms2g -Xmx2g \
  -XX:MetaspaceSize=256m \
  -XX:MaxMetaspaceSize=512m \
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=200 \
  -XX:G1HeapRegionSize=8m \
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/var/log/java/ \
  -Xlog:gc*:file=/var/log/java/gc.log:time:filecount=5,filesize=20m \
  -XX:+DisableExplicitGC \
  -jar app.jar
```

## 11.3 透明大页（THP）对 Java 的影响

```bash
# THP 对 Java 的负面影响：
# GC 进行内存整理时，Linux 会拆分大页，产生额外延迟
# 导致 GC 停顿时间增大，特别是 Full GC 时

# 查看 THP 状态
cat /sys/kernel/mm/transparent_hugepage/enabled

# 禁用 THP（临时）
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag

# 永久禁用（创建 systemd 服务）
cat > /etc/systemd/system/disable-thp.service << 'EOF'
[Unit]
Description=Disable Transparent Huge Pages (THP)
After=sysinit.target local-fs.target

[Service]
Type=oneshot
ExecStart=/bin/sh -c 'echo never > /sys/kernel/mm/transparent_hugepage/enabled'
ExecStart=/bin/sh -c 'echo never > /sys/kernel/mm/transparent_hugepage/defrag'

[Install]
WantedBy=multi-user.target
EOF
systemctl enable disable-thp
```

## 11.4 NUMA 架构对 Java 的影响

```bash
# NUMA（Non-Uniform Memory Access）：多 CPU socket 架构
# 每个 CPU socket 有自己的本地内存，访问远端内存更慢

numactl --hardware         # NUMA 节点信息
numastat                   # NUMA 统计

# Java 启用 NUMA 支持
java -XX:+UseNUMA -jar app.jar

# 绑定 NUMA 节点运行 Java（避免跨节点访问）
numactl --cpunodebind=0 --membind=0 java -jar app.jar

# taskset - CPU 核心绑定
taskset -c 0,1,2,3 java -jar app.jar  # 绑定到 CPU 0-3
```

---

# Part 12: 容器化运维（Docker）

## 12.1 Docker 常用命令速查

```bash
# ---- 镜像管理 ----
docker images                    # 列出本地镜像
docker pull nginx:latest         # 拉取镜像
docker rmi nginx:latest          # 删除镜像
docker image prune -a            # 清理所有未被容器使用的镜像
docker build -t myapp:v1.0 .     # 构建镜像

# ---- 容器管理 ----
docker ps                        # 运行中的容器
docker ps -a                     # 所有容器（包含停止的）
docker run -d \
    --name myapp \
    -p 8080:8080 \
    -e JAVA_OPTS="-Xmx2g" \
    -v /host/path:/container/path \
    --memory="2g" \
    --cpus="2" \
    --restart=always \
    myimage:latest

docker start/stop/restart myapp
docker rm -f myapp               # 强制删除
docker container prune           # 清理所有停止的容器

# ---- 容器日志 ----
docker logs -f myapp                    # 实时监控日志
docker logs --tail 100 myapp            # 最后100行
docker logs --since "1 hour ago" myapp  # 最近1小时

# ---- 进入容器调试 ----
docker exec -it myapp bash          # 进入容器（bash）
docker exec -it myapp sh            # 进入容器（sh，如 Alpine）
docker exec myapp ps aux            # 在容器内执行命令

# 查看容器资源使用
docker stats                  # 实时统计所有容器
docker stats --no-stream      # 输出一次

# ---- 容器网络 ----
docker network ls                    # 列出网络
docker network create mynet          # 创建自定义网络
docker inspect myapp | grep IPAddress  # 获取容器 IP
```

## 12.2 Dockerfile 最佳实践

```dockerfile
# Java 应用 Dockerfile 最佳实践

# 1. 使用精简的基础镜像
FROM eclipse-temurin:17-jre-alpine

# 2. 设置工作目录
WORKDIR /app

# 3. 创建非 root 用户（安全）
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# 4. 分层优化（将不常变的层放在前面）
COPY --chown=appuser:appgroup target/dependency/ ./dependency/

# 5. 复制应用
COPY --chown=appuser:appgroup target/app.jar ./

# 6. 切换到非 root 用户
USER appuser

# 7. 声明端口
EXPOSE 8080

# 8. 健康检查
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
    CMD curl -f http://localhost:8080/actuator/health || exit 1

# 9. 启动命令（使用 exec 形式）
ENTRYPOINT ["java", \
            "-XX:+UseContainerSupport", \
            "-XX:MaxRAMPercentage=75.0", \
            "-XX:+HeapDumpOnOutOfMemoryError", \
            "-XX:HeapDumpPath=/tmp/heap.hprof", \
            "-jar", "app.jar"]

# 关键：使用 -XX:MaxRAMPercentage 而非固定 -Xmx，更适合容器环境
```

## 12.3 Docker Compose 操作

```bash
docker-compose up -d             # 后台启动所有服务
docker-compose down              # 停止并删除容器
docker-compose down -v           # 同时删除数据卷
docker-compose stop/start/restart
docker-compose logs -f myservice # 实时监控特定服务日志
docker-compose exec myservice bash  # 进入服务容器
docker-compose ps                # 查看服务状态
```

## 12.4 容器网络排查

```bash
# 查看容器 IP
docker inspect myapp | grep IPAddress

# 容器间网络测试
docker exec myapp ping mydb

# 查看容器内的网络连接
docker exec myapp ss -tnlp

# 查看容器内的进程
docker exec myapp ps aux
docker top myapp

# 在容器内抓包
docker exec myapp tcpdump -i eth0 port 3306 -nn

# 排查启动失败
docker logs myapp 2>&1 | tail -50
```

---

# Part 13: 定时任务与服务管理

## 13.1 crontab - 定时任务

```bash
# crontab 语法
# 分 时 日 月 周 命令
# ┌───── 分钟（0-59）
# │ ┌───── 小时（0-23）
# │ │ ┌───── 日（1-31）
# │ │ │ ┌───── 月（1-12）
# │ │ │ │ ┌───── 星期（0-7，0和7都是周日）
# │ │ │ │ │
# * * * * * command

# 常见示例
0 * * * * command                  # 每小时的第0分
*/5 * * * * command                # 每5分钟
0 2 * * * /backup/db_backup.sh    # 每天凌晨2点
0 2 * * 0 /backup/weekly.sh       # 每周日凌晨2点
0 2 1 * * /backup/monthly.sh      # 每月1日凌晨2点
0 9-18 * * 1-5 command            # 工作日9-18点，每小时

# 管理 crontab
crontab -l                        # 列出当前用户的 crontab
crontab -e                        # 编辑 crontab（推荐）
crontab -r                        # 删除（谨慎！）

# 最佳实践：使用绝对路径，重定向输出到日志
MAILTO=""
0 2 * * * /opt/scripts/backup.sh >> /var/log/backup.log 2>&1

# 查看 crontab 执行日志
grep CRON /var/log/cron            # CentOS
journalctl -u cron -f              # systemd 日志
```

## 13.2 systemd 服务管理

```bash
systemctl start nginx                # 启动
systemctl stop nginx                 # 停止
systemctl restart nginx              # 重启
systemctl reload nginx               # 重载配置（不中断服务）
systemctl status nginx               # 查看状态
systemctl enable nginx               # 开机自启
systemctl disable nginx              # 禁止开机自启

# 列出服务
systemctl list-units --type=service --state=running  # 运行中的

# 日志
journalctl -u nginx -f               # 实时监控服务日志

# 重载配置（新增/修改 service 文件后必须执行！）
systemctl daemon-reload
```

## 13.3 编写 Java 服务的 systemd 单元文件

```bash
cat > /etc/systemd/system/myapp.service << 'EOF'
[Unit]
Description=My Java Application
After=network.target mysql.service

[Service]
Type=simple
User=appuser
Group=appuser
WorkingDirectory=/opt/myapp
Environment="JAVA_HOME=/usr/lib/jvm/java-17"
Environment="APP_ENV=production"
ExecStart=/usr/lib/jvm/java-17/bin/java \
    -Xms1g -Xmx2g \
    -XX:+UseG1GC \
    -XX:+HeapDumpOnOutOfMemoryError \
    -XX:HeapDumpPath=/var/log/myapp/ \
    -Xlog:gc*:file=/var/log/myapp/gc.log:time:filecount=5,filesize=20m \
    -Dspring.profiles.active=prod \
    -jar /opt/myapp/myapp.jar
ExecStop=/bin/kill -15 $MAINPID
KillSignal=SIGTERM
TimeoutStopSec=30
Restart=on-failure
RestartSec=5
StartLimitIntervalSec=60
StartLimitBurst=3
LimitNOFILE=65535
StandardOutput=journal
StandardError=journal
SyslogIdentifier=myapp

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable myapp
systemctl start myapp
journalctl -u myapp -f    # 查看启动日志
```

## 13.4 supervisor - 进程守护

```bash
# 安装
pip install supervisor
yum install -y supervisor    # CentOS EPEL

# 配置示例（/etc/supervisor/conf.d/myapp.conf）
# [program:myapp]
# command=/usr/bin/java -jar /opt/myapp/myapp.jar
# directory=/opt/myapp
# user=appuser
# autostart=true
# autorestart=true
# redirect_stderr=true
# stdout_logfile=/var/log/supervisor/myapp.log
# stdout_logfile_maxbytes=50MB
# stdout_logfile_backups=10
# environment=JAVA_HOME="/usr/lib/jvm/java-17"

# 管理命令
supervisorctl status                 # 查看所有程序状态
supervisorctl start myapp            # 启动
supervisorctl stop myapp             # 停止
supervisorctl restart myapp          # 重启
supervisorctl reread                 # 重读配置文件
supervisorctl reload                 # 重载
```


---

# 第十四章：安全加固

## 14.1 SSH 安全配置

SSH 是 Linux 服务器最常用的远程登录方式，也是攻击者最常尝试的入口。

### 14.1.1 SSH 配置文件

```bash
# 主配置文件
/etc/ssh/sshd_config      # 服务端配置
/etc/ssh/ssh_config       # 客户端配置
~/.ssh/config             # 用户级客户端配置
~/.ssh/authorized_keys    # 已授权的公钥列表
~/.ssh/known_hosts        # 已知主机指纹

# 修改配置后重启服务
systemctl restart sshd
```

### 14.1.2 禁用 root 直接登录

```bash
# 编辑 sshd_config
vim /etc/ssh/sshd_config

# 找到并修改以下行
PermitRootLogin no          # 禁止 root 登录（推荐）
# PermitRootLogin prohibit-password  # 仅允许 root 密钥登录（折中）

# 重启 SSH
systemctl restart sshd

# 验证配置
sshd -T | grep permitrootlogin
```

**注意**：禁用 root 前，确保已创建有 sudo 权限的普通用户！

```bash
# 创建运维用户
useradd -m -s /bin/bash devops
passwd devops

# 添加到 sudoers
usermod -aG wheel devops        # CentOS/RHEL
usermod -aG sudo devops         # Ubuntu/Debian

# 验证 sudo 权限
su - devops
sudo whoami    # 应输出 root
```

### 14.1.3 修改默认 SSH 端口

```bash
vim /etc/ssh/sshd_config

# 修改端口（选择 1024-65535 之间的端口）
Port 22022

# 如果开启了 SELinux，需要允许新端口
semanage port -a -t ssh_port_t -p tcp 22022

# 如果使用 firewalld，需要开放新端口
firewall-cmd --permanent --add-port=22022/tcp
firewall-cmd --reload
# 同时关闭旧端口
firewall-cmd --permanent --remove-service=ssh
firewall-cmd --reload

# 重启 SSH
systemctl restart sshd

# 客户端连接时指定端口
ssh -p 22022 user@server
```

### 14.1.4 配置密钥登录（禁用密码登录）

```bash
# ===== 客户端操作 =====
# 生成 SSH 密钥对（推荐 ed25519 算法）
ssh-keygen -t ed25519 -C "your_email@example.com"
# 或 RSA 4096
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"

# 密钥默认保存位置
~/.ssh/id_ed25519        # 私钥（绝对不能泄露！）
~/.ssh/id_ed25519.pub    # 公钥

# 将公钥复制到服务器
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@server
# 或手动追加
cat ~/.ssh/id_ed25519.pub | ssh user@server "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"

# ===== 服务端操作 =====
# 确保 .ssh 目录和文件权限正确
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys

# 修改 sshd_config 禁用密码登录
vim /etc/ssh/sshd_config
PubkeyAuthentication yes          # 启用公钥认证
AuthorizedKeysFile .ssh/authorized_keys
PasswordAuthentication no         # 禁用密码登录（配置好密钥后再改！）
ChallengeResponseAuthentication no

systemctl restart sshd
```

### 14.1.5 其他 SSH 安全加固

```bash
vim /etc/ssh/sshd_config

# 限制登录用户/组
AllowUsers devops admin          # 只允许这些用户登录
AllowGroups sshusers             # 只允许这些组的用户登录
DenyUsers testuser               # 拒绝这些用户登录

# 连接超时设置
ClientAliveInterval 300          # 300秒无操作发送心跳
ClientAliveCountMax 3            # 最多3次无响应后断开
LoginGraceTime 60                # 登录认证超时时间（秒）

# 禁用 X11 转发（如不需要）
X11Forwarding no

# 禁用空密码登录
PermitEmptyPasswords no

# 最大认证尝试次数
MaxAuthTries 3

# 限制并发未认证连接
MaxStartups 10:30:60

# 协议版本（只使用 SSH2）
# Protocol 2  # 新版本默认已是2，可不写

# 完整安全配置示例
cat > /etc/ssh/sshd_config << 'EOF'
Port 22022
ListenAddress 0.0.0.0
Protocol 2

HostKey /etc/ssh/ssh_host_ed25519_key
HostKey /etc/ssh/ssh_host_rsa_key

SyslogFacility AUTH
LogLevel INFO

LoginGraceTime 60
PermitRootLogin no
StrictModes yes
MaxAuthTries 3
MaxSessions 10

PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys

PasswordAuthentication no
PermitEmptyPasswords no
ChallengeResponseAuthentication no

X11Forwarding no
PrintMotd no
ClientAliveInterval 300
ClientAliveCountMax 3

AllowUsers devops admin
EOF

systemctl restart sshd
```

### 14.1.6 SSH 登录失败防护（Fail2Ban）

```bash
# 安装 fail2ban
yum install -y fail2ban          # CentOS
apt-get install -y fail2ban      # Ubuntu

# 创建本地配置文件（不要直接修改 .conf 文件）
cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local

vim /etc/fail2ban/jail.local

[DEFAULT]
bantime  = 3600       # 封禁时长（秒）
findtime = 600        # 检测窗口（秒）
maxretry = 5          # 最大失败次数

[sshd]
enabled  = true
port     = 22022      # 你修改后的 SSH 端口
logpath  = %(sshd_log)s
backend  = %(sshd_backend)s

# 启动 fail2ban
systemctl enable fail2ban
systemctl start fail2ban

# 查看被封禁的 IP
fail2ban-client status sshd

# 解封某个 IP
fail2ban-client set sshd unbanip 192.168.1.100
```

---

## 14.2 用户权限管理

### 14.2.1 用户和组管理

```bash
# 查看用户信息
id username                      # 查看用户 UID/GID/组
cat /etc/passwd                  # 用户数据库
cat /etc/shadow                  # 密码哈希（需 root）
cat /etc/group                   # 组数据库

# 用户管理
useradd -m -s /bin/bash -G wheel username   # 创建用户
usermod -aG docker username                 # 添加到附加组
userdel -r username                         # 删除用户（-r 删除家目录）
passwd username                             # 设置密码
passwd -l username                          # 锁定用户
passwd -u username                          # 解锁用户
chage -l username                           # 查看密码有效期
chage -M 90 username                        # 设置密码90天过期

# 组管理
groupadd groupname
groupdel groupname
gpasswd -a username groupname               # 添加用户到组
gpasswd -d username groupname               # 从组删除用户
groups username                             # 查看用户所在组
```

### 14.2.2 sudo 权限配置

```bash
# 安全编辑 sudoers（使用 visudo，有语法检查）
visudo

# 常用配置示例

# 给用户完全 sudo 权限
devops  ALL=(ALL:ALL) ALL

# 给用户无密码 sudo（生产环境谨慎使用）
devops  ALL=(ALL) NOPASSWD: ALL

# 只允许特定命令
deploy  ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart myapp, /usr/bin/systemctl status myapp

# 给组配置 sudo（%表示组）
%wheel  ALL=(ALL:ALL) ALL
%sudo   ALL=(ALL:ALL) ALL

# 禁止某用户使用 sudo
testuser  ALL=(ALL) !ALL

# 记录所有 sudo 命令日志
Defaults logfile="/var/log/sudo.log"
Defaults log_input, log_output

# 查看 sudo 日志
cat /var/log/sudo.log
journalctl | grep sudo
```

### 14.2.3 文件权限审计

```bash
# 查找 SUID 文件（可能被提权利用）
find / -perm -4000 -type f 2>/dev/null
find / -perm -u=s -type f 2>/dev/null

# 查找 SGID 文件
find / -perm -2000 -type f 2>/dev/null

# 查找全局可写文件（危险！）
find / -perm -0002 -type f 2>/dev/null | grep -v /proc

# 查找无主文件（孤儿文件，可能是残留）
find / -nouser -o -nogroup 2>/dev/null

# 查找最近修改的文件（排查入侵）
find /etc /bin /sbin /usr/bin -mtime -1 2>/dev/null   # 最近1天
find /var/www -newer /tmp/baseline -type f             # 比基准文件新

# 文件完整性检查（AIDE）
yum install -y aide
aide --init                    # 初始化数据库
mv /var/lib/aide/aide.db.new.gz /var/lib/aide/aide.db.gz
aide --check                   # 检查文件完整性
```

### 14.2.4 PAM 认证配置

```bash
# PAM 配置目录
ls /etc/pam.d/

# 限制密码复杂度（/etc/pam.d/system-auth 或 /etc/pam.d/common-password）
# 安装 pam_pwquality
yum install -y libpwquality

# /etc/security/pwquality.conf
minlen = 12                # 最小长度
minclass = 3               # 最少字符类型（大/小写/数字/特殊字符）
maxrepeat = 3              # 最大重复字符
dcredit = -1               # 至少1个数字
ucredit = -1               # 至少1个大写字母
lcredit = -1               # 至少1个小写字母
ocredit = -1               # 至少1个特殊字符

# 限制登录失败次数（pam_faillock）
# /etc/pam.d/system-auth 开头添加：
auth required pam_faillock.so preauth silent audit deny=5 unlock_time=1800
auth [default=die] pam_faillock.so authfail audit deny=5 unlock_time=1800

# 查看锁定状态
faillock --user username
# 解锁
faillock --user username --reset
```

---

## 14.3 防火墙配置

### 14.3.1 firewalld（CentOS 7+）

```bash
# 基本操作
systemctl start firewalld
systemctl enable firewalld
systemctl status firewalld

# 查看当前规则
firewall-cmd --list-all
firewall-cmd --list-all-zones

# 区域（Zone）概念
# public（默认）：公网，只接受指定服务
# trusted：完全信任，接受所有连接
# internal：内网
# dmz：隔离区

# 开放端口/服务
firewall-cmd --permanent --add-port=8080/tcp
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https

# 关闭端口/服务
firewall-cmd --permanent --remove-port=8080/tcp
firewall-cmd --permanent --remove-service=http

# 允许特定 IP 访问特定端口
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.1.0/24" port port="3306" protocol="tcp" accept'

# 封禁某个 IP
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="1.2.3.4" reject'

# 端口转发
firewall-cmd --permanent --add-forward-port=port=80:proto=tcp:toport=8080

# 重载配置
firewall-cmd --reload
```

### 14.3.2 iptables

```bash
# 查看规则
iptables -L -n -v
iptables -L -n -v --line-numbers

# 基本规则
# 允许已建立的连接
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# 允许本地回环
iptables -A INPUT -i lo -j ACCEPT

# 允许特定端口
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# 允许特定 IP
iptables -A INPUT -s 192.168.1.0/24 -j ACCEPT

# 拒绝其他所有入站
iptables -A INPUT -j DROP

# 删除规则
iptables -D INPUT 3             # 按行号删除
iptables -D INPUT -p tcp --dport 8080 -j ACCEPT  # 按规则删除

# 保存规则（CentOS）
service iptables save
# 或
iptables-save > /etc/iptables/rules.v4

# 查看 NAT 规则
iptables -t nat -L -n -v
```

---

## 14.4 端口扫描与漏洞排查

### 14.4.1 nmap 使用

```bash
# 安装
yum install -y nmap

# 基本扫描
nmap 192.168.1.1                      # 扫描单个主机
nmap 192.168.1.0/24                   # 扫描整个子网
nmap 192.168.1.1-50                   # 扫描 IP 范围

# 端口扫描类型
nmap -sS 192.168.1.1                  # SYN 扫描（半开放扫描，默认，需root）
nmap -sT 192.168.1.1                  # TCP 全连接扫描（不需root）
nmap -sU 192.168.1.1                  # UDP 扫描
nmap -sV 192.168.1.1                  # 服务版本探测

# 指定端口
nmap -p 22,80,443,8080 192.168.1.1
nmap -p 1-1000 192.168.1.1
nmap -p- 192.168.1.1                  # 扫描所有65535个端口

# OS 探测
nmap -O 192.168.1.1

# 综合扫描（常用）
nmap -sV -sC -O -p- 192.168.1.1

# 扫描结果输出到文件
nmap -oN output.txt 192.168.1.1       # 普通格式
nmap -oX output.xml 192.168.1.1       # XML 格式

# 快速扫描（只扫常用100个端口）
nmap -F 192.168.1.1

# 扫描本机开放端口
nmap localhost
ss -tlnp                              # 也可以用 ss
netstat -tlnp
```

### 14.4.2 常见安全漏洞排查

```bash
# ===== 检查异常进程 =====
ps aux
# 重点关注：
# - CPU/内存占用异常高的进程
# - 进程名含有乱码或看起来可疑的
# - /tmp 目录下运行的程序（高度可疑！）

ps aux | awk '{print $11}' | grep /tmp
ps aux | grep -v "^root" | awk '{print $1}' | sort | uniq  # 非root用户运行的进程

# ===== 检查异常网络连接 =====
ss -tnp
netstat -tnp
# 重点关注：
# - 未知的监听端口
# - 连接到可疑外部 IP 的连接
# - 大量 TIME_WAIT 或 ESTABLISHED 状态

# 查看连接到外部的进程
ss -tnp | grep ESTABLISHED

# ===== 检查异常定时任务 =====
crontab -l                   # 当前用户
crontab -l -u root           # root 用户
cat /etc/crontab
ls /etc/cron.d/
ls /etc/cron.daily/
ls /etc/cron.hourly/

# ===== 检查异常启动项 =====
systemctl list-unit-files --type=service | grep enabled
ls /etc/init.d/
ls /etc/rc.d/rc3.d/

# ===== 检查最近登录记录 =====
last                          # 登录历史
lastb                         # 登录失败历史
lastlog                       # 所有用户最后登录时间
who                           # 当前登录用户
w                             # 当前登录用户及操作

# ===== 检查异常文件 =====
# 查找最近24小时内修改的文件
find /etc /bin /sbin /usr/bin /tmp -mtime -1 2>/dev/null

# 查找 webshell（PHP）
find /var/www -name "*.php" | xargs grep -l "eval\|base64_decode\|system\|exec\|passthru" 2>/dev/null

# ===== 检查 /etc/passwd 和 /etc/shadow =====
# 查找 UID 为 0 的用户（除了 root 都是可疑的！）
awk -F: '($3 == 0) {print}' /etc/passwd

# 查找没有密码的用户
awk -F: '($2 == "") {print}' /etc/shadow

# ===== 检查 .bash_history =====
cat /root/.bash_history
cat /home/*/.bash_history
# 查找可疑命令
grep -i "wget\|curl\|chmod 777\|base64\|nc -e\|/bin/bash" /root/.bash_history

# ===== 检查 /tmp 目录 =====
ls -la /tmp/
# /tmp 下不应该有可执行文件！
find /tmp -type f -perm /0111

# ===== 系统完整性检查 =====
rpm -Va 2>/dev/null | grep "^..5"   # 检查 RPM 包文件是否被篡改（CentOS）
dpkg --verify 2>/dev/null            # Ubuntu
```

---

## 14.5 系统审计（auditd）

```bash
# 安装并启动
yum install -y audit
systemctl enable auditd
systemctl start auditd

# 配置审计规则 /etc/audit/rules.d/audit.rules

# 监控敏感文件访问
-w /etc/passwd -p wa -k passwd_changes
-w /etc/shadow -p wa -k shadow_changes
-w /etc/sudoers -p wa -k sudoers_changes
-w /etc/ssh/sshd_config -p wa -k sshd_config

# 监控特权命令执行
-a always,exit -F path=/usr/bin/sudo -F perm=x -k sudo_exec
-a always,exit -F path=/bin/su -F perm=x -k su_exec

# 监控网络配置变更
-w /etc/sysconfig/network -p wa -k network_changes
-a always,exit -F arch=b64 -S sethostname -S setdomainname -k system-locale

# 重载规则
augenrules --load

# 查询审计日志
ausearch -k passwd_changes          # 按 key 查询
ausearch -ua 1000                   # 按用户 ID 查询
ausearch -ts today                  # 今天的记录
ausearch -ts recent                 # 最近10分钟

# 生成审计报告
aureport --summary                  # 总体摘要
aureport --login                    # 登录报告
aureport --auth                     # 认证报告
aureport --failed                   # 失败事件报告
```

---

## 14.6 SELinux 基础

```bash
# 查看 SELinux 状态
getenforce                    # Enforcing / Permissive / Disabled
sestatus                      # 详细状态

# 临时修改模式（重启后失效）
setenforce 0                  # 切换到 Permissive（日志但不阻止）
setenforce 1                  # 切换到 Enforcing

# 永久修改 /etc/selinux/config
SELINUX=enforcing             # 强制模式（推荐）
SELINUX=permissive            # 宽容模式（排查问题时用）
SELINUX=disabled              # 禁用（不推荐，需要重启）

# 查看 SELinux 拒绝日志
ausearch -m avc -ts today
cat /var/log/audit/audit.log | grep AVC

# 常用 SELinux 命令
ls -Z /var/www/html/           # 查看文件安全上下文
ps -eZ | grep httpd            # 查看进程安全上下文
chcon -t httpd_sys_content_t /myfile  # 临时修改上下文
restorecon -Rv /var/www/html/  # 恢复默认上下文

# 管理布尔值
getsebool -a | grep httpd      # 查看 httpd 相关布尔值
setsebool -P httpd_can_network_connect on  # 允许 httpd 网络连接
```

---

---

# 第十五章：常见面试题 FAQ

> 本章收录 Linux 运维高频面试题，涵盖进程、内存、网络、文件系统、性能优化等核心方向，每题均附详细答案与实战延伸。

---

## FAQ-01：Linux 中进程和线程的区别？

**答：**

| 维度 | 进程 | 线程 |
|------|------|------|
| 资源 | 独立的地址空间、文件描述符、信号处理 | 共享所在进程的资源 |
| 创建开销 | 大（fork 需要复制页表等） | 小（pthread_create） |
| 通信 | IPC（管道/共享内存/消息队列/Socket） | 共享内存，直接读写变量 |
| 隔离性 | 强（一个进程崩溃不影响其他） | 弱（一个线程崩溃可能导致整个进程崩溃） |
| 调度单位 | 是 | 是（Linux 中线程是轻量级进程 LWP） |

**Linux 实现**：Linux 没有专门的"线程"概念，线程是通过 `clone()` 系统调用实现的轻量级进程（LWP），与进程的区别只是共享了哪些资源（通过 `CLONE_VM`、`CLONE_FS`、`CLONE_FILES` 等标志控制）。

**查看线程**：
```bash
ps -eLf | grep java       # 查看 Java 进程的所有线程
top -H -p <pid>           # 查看进程的线程视图
cat /proc/<pid>/status | grep Threads  # 线程数
```

**延伸**：Java 中每个线程对应一个 OS 线程（1:1 模型），所以线程数不能太多，建议控制在几百到几千以内。

---

## FAQ-02：fork() 和 exec() 的区别？写时复制（COW）是什么？

**答：**

- **fork()**：创建当前进程的副本（子进程），子进程继承父进程的代码、数据、文件描述符等，返回值：父进程得到子进程 PID，子进程得到 0。
- **exec()**：在当前进程中加载并执行新程序，替换当前进程的代码段/数据段，但保留 PID 和文件描述符。
- **vfork()**：不复制父进程地址空间，子进程直接在父进程地址空间运行，父进程阻塞直到子进程 exec 或 exit。

**写时复制（Copy-On-Write, COW）**：
fork() 时并不立即复制父进程的内存页，而是让父子进程共享同一物理内存，并将这些页标记为只读。当任意一方尝试写入时，才触发缺页中断，内核此时才真正复制该内存页给写入方。

**好处**：极大提高了 fork 的性能，尤其是 fork 后立即 exec 的场景（如 shell 执行命令）几乎不需要复制内存。

**与 Java 的关系**：Redis 做 RDB 持久化时使用 fork+COW，父进程继续服务，子进程负责写磁盘。如果写操作频繁，COW 复制的页很多，会导致内存使用量翻倍。

---

## FAQ-03：僵尸进程是什么？如何产生和处理？

**答：**

**僵尸进程（Zombie）**：子进程已经终止，但父进程还没有调用 `wait()`/`waitpid()` 读取其退出状态，此时子进程处于 Z（zombie）状态。僵尸进程已释放了大部分资源，但仍占用进程表项（PID）。

**产生原因**：父进程没有正确处理子进程退出（忘记调用 wait，或父进程 bug）。

**危害**：如果僵尸进程过多，会耗尽 PID（Linux 默认最大 PID 为 32768），导致无法创建新进程。

**查看僵尸进程**：
```bash
ps aux | grep Z
ps aux | awk '{if ($8=="Z") print}'
top  # 查看 zombie 数量（第一行的 zombie 字段）
```

**处理方法**：
```bash
# 方法1：杀死父进程（让 init/systemd 接管并回收）
kill -9 <父进程PID>

# 方法2：向父进程发送 SIGCHLD 信号（提示父进程回收）
kill -SIGCHLD <父进程PID>

# 方法3：如果父进程是自己写的程序，修复代码，正确调用 wait()
```

---

## FAQ-04：Linux 内存管理——Buffer 和 Cache 的区别？

**答：**

```
free -h 输出中：
               total   used   free   shared  buff/cache   available
Mem:           15Gi    8Gi    1Gi    512Mi      6Gi         6Gi
```

- **Buffer（缓冲区）**：用于缓冲**写操作**，主要是块设备（磁盘）的写缓冲。数据先写入 buffer，由内核异步刷写到磁盘（write-back）。
- **Cache（缓存）**：用于缓存**读操作**，主要是文件系统的页缓存（Page Cache）。读取文件时，内容会被缓存在 Cache 中，下次读取直接从内存返回。

**现代 Linux**：Buffer 和 Cache 在实现上高度融合（都基于 Page Cache），free 命令将二者合并显示为 `buff/cache`。

**关键点**：
- buff/cache 是可以被回收的内存，当应用程序需要内存时，内核会自动回收 Cache
- `available` 字段 = free + 可回收的 buff/cache，代表实际可用内存，比 free 更有参考价值
- 不要因为 free 很少就认为内存不够，要看 available

**手动清理缓存**（生产环境慎用！）：
```bash
echo 1 > /proc/sys/vm/drop_caches   # 清除 PageCache
echo 2 > /proc/sys/vm/drop_caches   # 清除 dentries 和 inodes
echo 3 > /proc/sys/vm/drop_caches   # 清除 PageCache + dentries + inodes
sync                                  # 先同步，防止数据丢失
```

---

## FAQ-05：什么是 OOM Killer？如何防止 Java 应用被 OOM Kill？

**答：**

**OOM Killer**：当系统内存严重不足时，Linux 内核的 Out-of-Memory Killer 会选择一个进程将其强制终止，以释放内存，保证系统继续运行。

**选择算法**：每个进程有一个 `oom_score`（0-1000），分值越高越容易被杀死。评分考虑：进程内存占用、运行时间、是否有特权等。

```bash
# 查看进程的 OOM 分数
cat /proc/<pid>/oom_score

# 调整 OOM 权重（-1000 到 1000，越小越不容易被杀）
echo -500 > /proc/<pid>/oom_score_adj
# 设为 -1000 表示完全不被 OOM Kill（谨慎！可能导致系统崩溃）
echo -1000 > /proc/<pid>/oom_score_adj

# 查看 OOM Kill 记录
dmesg | grep -i "oom\|killed process"
journalctl -k | grep -i oom
grep -i "out of memory" /var/log/messages
```

**防止 Java 应用被 OOM Kill 的方法**：

1. **合理设置 JVM 堆内存**，不要设置过大：`-Xmx` 不超过物理内存的 70%
2. **降低 OOM 分数**：`echo -300 > /proc/$(pgrep java)/oom_score_adj`
3. **使用 cgroup 限制内存**：容器环境下设置合理的 memory limit
4. **监控内存使用**：提前发现内存增长趋势
5. **分析 GC 日志**：排查内存泄漏

---

## FAQ-06：系统 load average 是什么？高 load 一定代表 CPU 繁忙吗？

**答：**

**Load Average（系统负载均值）**：过去 1/5/15 分钟内，处于**可运行状态**（Running/Runnable）和**不可中断睡眠状态**（Uninterruptible Sleep，D 状态）的进程数量的平均值。

```bash
uptime
# 输出：load average: 1.23, 0.98, 0.87
#                      1min  5min  15min
```

**判断标准**：
- Load 的参考值是 **CPU 核心数**
- Load = CPU 核心数：CPU 刚好满负载
- Load < CPU 核心数：系统不繁忙
- Load > CPU 核心数：存在排队等待，系统过载

```bash
# 查看 CPU 核心数
nproc
grep -c processor /proc/cpuinfo
```

**高 load 不一定 CPU 繁忙！**

- 如果大量进程处于 **D 状态**（等待 IO），load 会很高，但 CPU 使用率（us + sy）可能很低
- `top` 中 `wa`（IO wait）很高时，说明是 IO 瓶颈，不是 CPU 瓶颈

**排查步骤**：
```bash
top                          # 查看 CPU us/sy/wa 和进程状态
ps aux | awk '{print $8}' | sort | uniq -c  # 统计进程状态
# 如果 D 状态进程多 → 磁盘/网络 IO 问题
# 如果 R 状态进程多 → CPU 计算瓶颈
iostat -x 1                  # 查看磁盘 IO 情况
```

---

## FAQ-07：TCP 三次握手和四次挥手的 Linux 实践

**答：**

**三次握手**：
1. 客户端发 SYN → 服务端（客户端进入 SYN_SENT）
2. 服务端发 SYN+ACK → 客户端（服务端进入 SYN_RCVD）
3. 客户端发 ACK → 服务端（双方进入 ESTABLISHED）

**四次挥手**：
1. 主动方发 FIN（进入 FIN_WAIT_1）
2. 被动方回 ACK（主动方进入 FIN_WAIT_2，被动方进入 CLOSE_WAIT）
3. 被动方发 FIN（被动方进入 LAST_ACK）
4. 主动方回 ACK（主动方进入 TIME_WAIT，等待 2MSL 后关闭）

**Linux 调优参数**：
```bash
# 查看各种 TCP 状态的连接数
ss -s
netstat -an | awk '{print $6}' | sort | uniq -c | sort -rn

# SYN Flood 防护
sysctl -w net.ipv4.tcp_syncookies=1          # 开启 SYN Cookie
sysctl -w net.ipv4.tcp_max_syn_backlog=8192  # SYN 队列大小

# TIME_WAIT 优化（大量短连接场景）
sysctl -w net.ipv4.tcp_tw_reuse=1            # 允许复用 TIME_WAIT socket
sysctl -w net.ipv4.tcp_fin_timeout=30        # 缩短 FIN_WAIT_2 超时时间

# CLOSE_WAIT 过多：通常是应用 bug，没有正确关闭连接
# 排查：lsof -p <pid> | grep CLOSE_WAIT
```

---

## FAQ-08：文件系统 inode 是什么？inode 满了怎么办？

**答：**

**inode（索引节点）**：存储文件元数据（权限、属主、时间戳、数据块指针等）的数据结构，每个文件/目录对应一个 inode，通过 inode number 唯一标识。inode 不存储文件名（文件名存储在目录项 dentry 中）。

**inode 包含的信息**：
- 文件类型、权限、UID/GID
- 文件大小、时间戳（atime/mtime/ctime）
- 硬链接计数
- 数据块地址列表
- **不包含**：文件名、文件内容

```bash
# 查看 inode 使用情况
df -i
df -ih

# 查看文件的 inode 号
ls -i filename
stat filename

# inode 满了的症状
# 即使磁盘空间充足，也无法创建新文件，报错：No space left on device

# 排查 inode 耗尽的目录（大量小文件）
for i in /proc /sys /run /dev /var /tmp /usr /home; do
    echo "$i: $(find $i -maxdepth 1 -mindepth 1 | wc -l)"
done
find / -xdev -maxdepth 5 -type d | xargs -I{} sh -c 'echo "$(find {} -maxdepth 1 | wc -l) {}"' | sort -rn | head -20

# 常见原因：Java 应用产生大量小日志文件、邮件队列、session 文件未清理
```

---

## FAQ-09：软链接和硬链接的区别？

**答：**

| 对比项 | 硬链接（Hard Link） | 软链接（Symbolic Link） |
|--------|--------------------|-----------------------|
| inode | 与原文件相同 inode | 新的 inode |
| 存储内容 | 直接指向数据块 | 存储目标文件路径字符串 |
| 跨文件系统 | 不支持 | 支持 |
| 对目录 | 不允许（防止循环） | 允许 |
| 原文件删除 | 数据仍可访问（inode 引用计数>0） | 链接失效（悬空链接） |
| 文件大小 | 与原文件相同 | 路径字符串的长度 |

```bash
# 创建硬链接
ln original.txt hardlink.txt
ls -li original.txt hardlink.txt  # 相同 inode 号，链接数=2

# 创建软链接
ln -s /path/to/original.txt symlink.txt
ls -la symlink.txt  # 显示 -> /path/to/original.txt

# 常用场景
ln -s /usr/local/jdk-17 /usr/local/java   # 软链接方便版本切换
ln -s /data/logs/app.log /var/log/app.log  # 软链接统一日志路径
```

---

## FAQ-10：如何排查服务器 CPU 使用率飙到 100%？

**答：**

这是最高频的运维面试题之一，需要给出完整的排查步骤。

**完整排查流程**：

```bash
# Step 1：确认 CPU 使用情况
top
# 观察：us（用户态）高还是 sy（内核态）高？wa（IO wait）高？

# Step 2：找到高 CPU 进程
top                           # 按 P 键排序
ps aux --sort=-%cpu | head -10

# 假设是 Java 进程，PID 为 12345

# Step 3：找到高 CPU 线程（Java 特有步骤）
# 方法A：top 命令
top -H -p 12345               # 查看线程级 CPU，记录高CPU线程的PID（十进制）
# 假设线程 PID 为 12390

# 方法B：ps 命令
ps -mp 12345 -o THREAD,tid,time | sort -rn | head -20

# Step 4：将线程 PID 转换为十六进制
printf '%x\n' 12390           # 输出：3066

# Step 5：获取 Java 线程栈
jstack 12345 | grep -A 30 "nid=0x3066"
# 或者导出完整 jstack
jstack 12345 > /tmp/jstack.log
grep -A 30 "nid=0x3066" /tmp/jstack.log

# Step 6：分析线程栈
# 场景1：死循环
# 栈中重复出现相同的代码行，是死循环

# 场景2：GC 线程占用高 CPU
# 线程名包含 "GC task thread" 或 "Concurrent Mark-Sweep GC Thread"
# → 说明频繁 GC，检查堆内存
jstat -gcutil 12345 1000      # 查看 GC 情况

# 场景3：使用 Arthas 在线诊断（推荐）
java -jar arthas-boot.jar 12345
thread -n 3                   # 查看最忙的3个线程及其栈
```

**常见原因及解决方案**：

| 原因 | 现象 | 解决 |
|------|------|------|
| 死循环 | 单线程 CPU 100% | 修复代码逻辑 |
| 频繁 Full GC | GC 线程 CPU 高，内存 GC 占比 90%+ | 增加堆内存，排查内存泄漏 |
| 正则表达式回溯 | 单线程 CPU 高，处理某类请求超慢 | 优化正则表达式 |
| 加密解密运算 | CPU sys 高 | 使用硬件加速或优化算法 |

---

## FAQ-11：如何排查 Java 服务出现 OutOfMemoryError（OOM）？

**答：**

**Step 1：配置 JVM OOM 时自动 dump**
```bash
# JVM 启动参数添加
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/tmp/heapdump.hprof
```

**Step 2：OOM 发生时排查**
```bash
# 查看 GC 状态
jstat -gcutil <pid> 1000
# 如果 Old 区接近 100% 且每次 Full GC 后回收很少，说明内存泄漏

# 查看堆内存占用（按对象数量排序前20）
jmap -histo <pid> | head -20
jmap -histo:live <pid> | head -20   # 触发一次 Full GC 后统计活对象

# 导出 heap dump（会导致服务短暂停止，生产环境慎用！）
jmap -dump:format=b,file=/tmp/heapdump.hprof <pid>
```

**Step 3：分析 heap dump**
```bash
# 使用 Eclipse MAT（Memory Analyzer Tool）分析
# 1. 打开 heapdump.hprof
# 2. 查看 Leak Suspects Report（泄漏嫌疑报告）
# 3. 查看 Dominator Tree（对象占用最多内存的对象树）
# 4. 找到占用最多内存的对象类型，结合代码排查

# 使用 JDK 自带工具 jhat（已废弃，了解即可）
jhat -port 7000 heapdump.hprof
# 浏览器访问 http://localhost:7000

# 使用 Arthas 在线分析
heapdump /tmp/heapdump.hprof
```

**常见 OOM 类型**：

| OOM 类型 | 原因 |
|----------|------|
| `Java heap space` | 堆内存不足，可能内存泄漏或堆设置过小 |
| `GC overhead limit exceeded` | GC 时间超过 98% 但回收不到 2% 内存 |
| `Metaspace` | 类加载过多，动态代理/反射生成大量类 |
| `Direct buffer memory` | NIO 直接内存耗尽（Netty 常见） |
| `unable to create new native thread` | 线程数超限（`ulimit -u` 或内存不足） |

---

## FAQ-12：什么是零拷贝（Zero-Copy）？在 Java 中如何使用？

**答：**

**传统文件发送流程（4次拷贝，4次上下文切换）**：
```
磁盘 → 内核 Page Cache（DMA 拷贝）
内核 Page Cache → 用户空间 buffer（CPU 拷贝）
用户空间 buffer → Socket buffer（CPU 拷贝）
Socket buffer → 网卡（DMA 拷贝）
```

**sendfile 零拷贝（2次拷贝，2次上下文切换）**：
```
磁盘 → 内核 Page Cache（DMA 拷贝）
内核 Page Cache → 网卡（DMA 拷贝，通过 file descriptor 传递，不经过用户空间）
```

**mmap 内存映射（3次拷贝）**：
```
磁盘 → 内核 Page Cache（DMA 拷贝）
内核 Page Cache 映射到用户空间（共享，不拷贝）
内核 Page Cache → Socket buffer → 网卡（DMA 拷贝）
```

**Java 中的应用**：
```java
// Java NIO FileChannel.transferTo() → 底层使用 sendfile
FileChannel sourceChannel = new FileInputStream(file).getChannel();
SocketChannel destChannel = socketChannel;
sourceChannel.transferTo(0, sourceChannel.size(), destChannel);

// Java NIO MappedByteBuffer → 底层使用 mmap
FileChannel channel = new RandomAccessFile(file, "rw").getChannel();
MappedByteBuffer buffer = channel.map(FileChannel.MapMode.READ_WRITE, 0, channel.size());
```

**使用场景**：Kafka、Netty、Nginx 等高性能中间件都大量使用零拷贝技术。

---

## FAQ-13：如何查看和分析系统调用？strace 怎么用？

**答：**

**strace** 是 Linux 下用于跟踪进程系统调用和信号的工具，是排查程序阻塞、文件访问、网络问题的利器。

```bash
# 跟踪命令的系统调用
strace ls -la

# 跟踪正在运行的进程
strace -p <pid>

# 跟踪进程及其所有子进程（-f）
strace -f -p <pid>

# 只显示特定系统调用（-e）
strace -e trace=open,read,write ls
strace -e trace=network curl http://example.com    # 只看网络相关调用
strace -e trace=file ls                             # 只看文件相关调用

# 统计系统调用次数和时间（-c）
strace -c -p <pid>
# 输出每种系统调用的次数、时间占比、错误次数

# 输出到文件（-o）
strace -o /tmp/strace.log -p <pid>

# 跟踪线程（-f 跟踪 clone 创建的线程）
strace -f -p <pid>

# 实战：排查 Java 进程为什么一直阻塞
strace -f -p <javapid> 2>&1 | grep -E "futex|epoll|poll|select"
# futex 是 Java synchronized/wait 的底层实现
# epoll 是 NIO 的底层实现

# 查看进程打开了哪些文件
strace -e trace=openat -p <pid> 2>&1 | grep -v ENOENT
# 或用 lsof
lsof -p <pid>
```

---

## FAQ-14：Linux 中的信号（Signal）有哪些？SIGKILL 和 SIGTERM 的区别？

**答：**

**常用信号列表**：

```bash
kill -l   # 列出所有信号

# 常用信号
信号编号  信号名     含义
1        SIGHUP    挂起，终端断开。守护进程重新加载配置
2        SIGINT    键盘中断（Ctrl+C）
3        SIGQUIT   键盘退出（Ctrl+\），产生 core dump
9        SIGKILL   强制杀死，不可忽略/捕获/阻塞
15       SIGTERM   终止请求，可捕获，默认信号（kill 不带参数）
18       SIGCONT   继续运行（fg 命令发送此信号）
19       SIGSTOP   暂停进程，不可忽略（Ctrl+Z 发送 SIGTSTP）
```

**SIGKILL vs SIGTERM**：

| 对比 | SIGKILL (9) | SIGTERM (15) |
|------|-------------|--------------|
| 可捕获/忽略 | 不可以（内核直接处理） | 可以 |
| 进程响应 | 立即被杀死 | 进程可以优雅关闭（清理资源、关闭连接等） |
| 使用场景 | 进程无响应时最后手段 | 正常关闭进程的首选 |

**最佳实践**：
```bash
# 正确的停止进程流程
kill <pid>                  # 先发 SIGTERM，等待进程优雅退出
sleep 10
kill -0 <pid> && kill -9 <pid>  # 10秒后仍未退出，再发 SIGKILL

# Java 应用监听 SIGTERM 实现优雅关闭
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    // 关闭连接池、等待处理中的请求完成等
}));

# systemd 停止服务也是先发 SIGTERM，超时后发 SIGKILL
# TimeoutStopSec=30 配置超时时间
```

---

## FAQ-15：磁盘 IO 性能排查——如何找到 IO 密集的进程和文件？

**答：**

```bash
# Step 1：宏观查看 IO 情况
iostat -x 1
# 重点关注：
# %util：磁盘繁忙程度（接近100%说明磁盘已成瓶颈）
# await：IO 请求平均等待时间（毫秒）
# r/s, w/s：每秒读写请求数
# rkB/s, wkB/s：每秒读写 KB 数
# svctm：服务时间（已废弃，参考意义不大）

# Step 2：找到 IO 密集的进程
iotop -o              # 只显示有 IO 的进程（-o 选项）
iotop -o -b -n 3      # 批量模式，适合脚本

# Step 3：找到 IO 密集的文件（lsof）
# 查看正在操作某个磁盘的进程
lsof /dev/sda

# 找到某进程打开的所有文件
lsof -p <pid>

# 找到大文件写入
lsof -p <pid> | awk '{print $7, $9}' | sort -rn | head

# Step 4：使用 strace 跟踪 IO 系统调用
strace -e trace=read,write,pread64,pwrite64 -p <pid>

# Step 5：使用 perf 进行深入分析
perf record -e block:block_rq_issue -ag sleep 10
perf report

# Step 6：检查 dirty page 刷写
cat /proc/meminfo | grep -i dirty
# 如果 Dirty 很大，说明有大量待写入数据

# 常见解决方案
# - 大量顺序写：考虑异步写入、批量合并写入
# - 大量随机读：考虑使用 SSD、增加读缓存
# - 日志写入过多：限制日志级别、使用异步日志（如 Log4j2 的 AsyncAppender）
# - 数据库 IO 高：检查慢 SQL，增加索引，使用读写分离
```

---

## FAQ-16：如何实现 Linux 下 Java 服务的平滑重启（不丢请求）？

**答：**

**原理**：平滑重启需要在新实例完全启动并接受请求后，再停止旧实例，确保不中断服务。

**方案一：nginx upstream + 脚本**
```bash
#!/bin/bash
# 平滑重启 Java 应用

APP_NAME="myapp"
JAR_PATH="/app/myapp.jar"
PORT=8080
HEALTH_URL="http://localhost:${PORT}/actuator/health"

# Step 1：启动新实例（使用不同端口或进程）
NEW_PID=$(nohup java -jar $JAR_PATH &; echo $!)

# Step 2：等待新实例健康检查通过
for i in $(seq 1 60); do
    STATUS=$(curl -s -o /dev/null -w "%{http_code}" $HEALTH_URL)
    if [ "$STATUS" = "200" ]; then
        echo "新实例启动成功"
        break
    fi
    sleep 2
done

# Step 3：通知 nginx 切流量到新实例（或通过注册中心自动切换）
# nginx -s reload

# Step 4：优雅停止旧实例
OLD_PID=$(cat /var/run/${APP_NAME}.pid)
kill -15 $OLD_PID

# Step 5：等待旧实例退出
for i in $(seq 1 30); do
    kill -0 $OLD_PID 2>/dev/null || break
    sleep 1
done

# Step 6：确认旧实例已退出
kill -0 $OLD_PID 2>/dev/null && kill -9 $OLD_PID

echo "平滑重启完成"
```

**方案二：使用 Kubernetes Rolling Update（容器化场景）**
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1            # 最多多启动1个新 Pod
    maxUnavailable: 0      # 不允许有不可用 Pod（确保零停机）
```

**方案三：Spring Boot Graceful Shutdown（Spring Boot 2.3+）**
```yaml
# application.yml
server:
  shutdown: graceful          # 开启优雅关闭

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s   # 等待进行中请求最长30秒
```

---

## FAQ-17：top 命令输出中各指标的含义？

**答：**

```
top - 14:23:45 up 30 days, 3:12,  2 users,  load average: 1.20, 0.85, 0.70
Tasks: 215 total,   1 running, 214 sleeping,   0 stopped,   0 zombie
%Cpu(s):  8.3 us,  2.1 sy,  0.0 ni, 88.5 id,  0.8 wa,  0.0 hi,  0.3 si,  0.0 st
MiB Mem :  15870.1 total,   1024.5 free,   8192.3 used,   6653.3 buff/cache
MiB Swap:   2048.0 total,   1500.0 free,    548.0 used.   7200.0 avail Mem
```

**第一行（系统概要）**：
- `up 30 days`：系统运行时间
- `2 users`：登录用户数
- `load average: 1.20, 0.85, 0.70`：1/5/15分钟平均负载

**Tasks 行**：
- `running`：正在运行的进程数（R 状态）
- `sleeping`：睡眠中的进程（S 状态）
- `stopped`：已停止（T 状态）
- `zombie`：僵尸进程（Z 状态）

**%Cpu(s) 行（最重要！）**：

| 字段 | 含义 | 异常值说明 |
|------|------|-----------|
| `us` | 用户空间 CPU 使用率 | 高→应用 CPU 密集，排查具体进程 |
| `sy` | 内核空间 CPU 使用率 | 高→系统调用频繁，可能是 IO 或网络 |
| `ni` | nice 值调整进程的 CPU | 通常为0 |
| `id` | CPU 空闲率 | 低→CPU 繁忙 |
| `wa` | IO 等待 | 高→磁盘 IO 瓶颈 |
| `hi` | 硬件中断 | 高→网络包量很大 |
| `si` | 软件中断 | 高→网络包处理繁忙 |
| `st` | 虚拟化 steal time | 高→宿主机抢占 vCPU（云主机常见） |

**进程列表重要列**：
- `PR`：内核调度优先级（越小优先级越高）
- `NI`：nice 值（-20 到 19，影响 PR）
- `VIRT`：进程虚拟地址空间大小（包含未实际使用的内存）
- `RES`：常驻内存（实际占用物理内存）
- `SHR`：共享内存（与其他进程共享的内存）
- `S`：进程状态（R/S/D/Z/T）
- `%CPU`：CPU 使用率
- `%MEM`：物理内存使用率

**top 常用交互命令**：
```
P     按 CPU 使用率排序
M     按内存使用率排序
T     按 CPU 时间排序
k     kill 进程
r     修改进程 nice 值
H     显示线程（进入线程视图）
1     显示每个 CPU 核心的使用情况
q     退出
```

---

## 附录：Linux 运维速查手册

### 常用命令速查表

```bash
# ===== 系统信息 =====
uname -a                    # 内核版本
cat /etc/os-release         # 发行版信息
uptime                      # 运行时间和负载
hostname -I                 # 本机 IP
date                        # 当前时间
timedatectl                 # 时间和时区
locale                      # 语言设置

# ===== 用户管理 =====
whoami                      # 当前用户
id                          # 用户 UID/GID/组
who                         # 已登录用户
w                           # 登录用户及操作
last                        # 登录历史
useradd/usermod/userdel     # 用户管理
passwd                      # 修改密码

# ===== 进程管理 =====
ps aux                      # 所有进程
ps -ef --forest             # 进程树
top / htop                  # 动态进程监控
kill -15 <pid>              # 优雅停止
kill -9 <pid>               # 强制停止
nohup cmd &                 # 后台运行
jobs / fg / bg              # 作业控制

# ===== 文件操作 =====
ls -la                      # 列出文件
find / -name "*.log"        # 查找文件
grep -r "error" /var/log    # 递归搜索
tail -f /var/log/app.log    # 实时查看日志
wc -l file                  # 统计行数
du -sh /path                # 目录大小
df -h                       # 磁盘使用

# ===== 网络 =====
ip addr / ifconfig          # 网卡信息
ss -tlnp / netstat -tlnp    # 监听端口
ping / traceroute           # 连通性测试
curl -v http://url          # HTTP 请求
tcpdump -i eth0             # 抓包
nmap -sV host               # 端口扫描

# ===== 性能分析 =====
vmstat 1                    # 系统综合指标
iostat -x 1                 # IO 统计
iotop                       # IO 进程监控
free -h                     # 内存使用
sar -u 1 10                 # CPU 历史

# ===== 日志查看 =====
journalctl -f               # 实时系统日志
journalctl -u nginx         # 服务日志
dmesg | tail                # 内核日志
tail -f /var/log/messages   # 系统消息
```

### Java 诊断命令速查

```bash
# ===== JVM 监控 =====
jps -l                      # Java 进程列表
jinfo -flags <pid>          # JVM 参数
jstat -gcutil <pid> 1000    # GC 统计（每1秒）
jstat -gc <pid>             # GC 详细统计

# ===== 线程分析 =====
jstack <pid>                # 线程栈
jstack -l <pid>             # 含锁信息
top -H -p <pid>             # 线程 CPU

# ===== 内存分析 =====
jmap -heap <pid>            # 堆信息
jmap -histo:live <pid>      # 对象直方图
jmap -dump:format=b,file=/tmp/dump.hprof <pid>

# ===== Arthas 常用命令 =====
# 启动：java -jar arthas-boot.jar
dashboard         # 系统面板
thread -n 5       # 最忙的5个线程
thread -b         # 找死锁
trace MyClass method  # 方法调用链路追踪
watch MyClass method  # 监听方法入参/返回值
jad MyClass       # 反编译
ognl '@System@getProperty("java.version")'  # 执行表达式
heapdump /tmp/dump.hprof  # dump 堆
```

---

*文档结束*

*版本：v1.0 | 最后更新：2026年 | 适用范围：Linux 运维、Java 后端开发*
