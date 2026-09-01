# Linux

操作系统是硬件和程序之间的中间层：管内存、进程/线程、设备、网络，再给你一套接口去用它们。

Linux 这一组笔记按「你在干什么」拆开，不要从这一份从头看到尾。

| 文件 | 干什么时翻 |
|---|---|
| [linux.md](linux.md)（本页） | 机器怎么起来：内核、GRUB、终端环境 |
| [linux-file.md](linux-file.md) | 一切皆文件：权限、目录命令、文件 IO、描述符 |
| [linux-process.md](linux-process.md) | 进程怎么生怎么死，进程之间怎么说话 |
| [linux-thread.md](linux-thread.md) | 进程内部的并发：锁、条件变量、线程池 |

和 ROS 的关系：ROS 节点是进程，Topic/Service/Action 是进程间通信的一种约定。底层管道、共享内存、套接字，在进程篇。

---

## 1. 开机这条链

```
电源
  → BIOS / UEFI          找到磁盘上的启动器
  → GRUB                 第一个真正的程序：列出能启动的内核
  → 内核 vmlinuz + initrd 接管硬件，挂上根文件系统
  → systemd 等           拉起用户态服务（sshd、你的桌面、ROS 守护进程…）
  → 你的 shell           读 ~/.bashrc，这才是你敲命令的地方
```

GRUB 是菜单，不是内核。它读 `/boot/grub/grub.cfg`，把你选中的 `vmlinuz-xxx` 和 `initrd.img-xxx` 交给 CPU，然后自己下场。

---

## 2. 内核：翻译官 + 调度员

内核跑在硬件上，应用程序不能直接碰磁盘和网卡，只能通过系统调用（`open` / `read` / `fork`…）请内核代劳。

核心职责就四块：进程调度、内存隔离、设备驱动、系统调用。

发行版会同时留几个内核，新的用来修漏洞，旧的当启动失败时的退路，文件在 `/boot`。

```bash
hostnamectl                          # 主机名、系统、当前内核
uname -r                             # 正在跑的内核版本
dpkg --get-selections | grep linux-image
ls /boot/vmlinuz*
```

换默认启动项：改 `/etc/default/grub`，然后必须：

```bash
sudo update-grub
```

`update-grub` 会重新扫 `/boot`，生成新菜单。只 `rm` 了内核文件却不更新 GRUB，开机菜单里会留下点了就失败的条目。

删旧内核用包管理器，不要手删 `/boot` 里的文件：

```bash
sudo apt remove linux-image-6.17.0-23-generic
sudo update-grub
```

---

## 3. 终端环境：`~/.bashrc`

每个**新开的交互式 bash** 都会跑一遍家目录下的 `~/.bashrc`。改的是「这个终端的记事本」（环境变量、别名），不是改系统本身。改完要：

```bash
source ~/.bashrc
```

或新开一个终端。

常见会写进去的东西：

```bash
export EDITOR=vim
export PATH="$HOME/bin:$PATH"

alias ll='ls -alF'
alias ..='cd ..'

# ROS 2：免得每次手动 source（版本和路径换成你的）
source /opt/ros/jazzy/setup.bash
source ~/ros2_ws/install/setup.bash
export ROS_WS=~/ros2_ws
alias cdros='cd $ROS_WS'
alias rosbuild='colcon build --symlink-install'
```

`source` 的本质：把脚本里的命令当成你在当前终端亲手敲的。所以 `PATH`、`AMENT_PREFIX_PATH` 的修改只活在这个终端里。这和 ROS 工作空间那一套是同一件事。
