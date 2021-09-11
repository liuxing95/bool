---
title: docker指令
date: 2020-01-14 14:04:51
toc: true
categories: 工具集
tags:
  docker
---
# Docker命令
Docker 提供了丰富的命令用于创建、管理、销毁、查询 Docker 镜像和容器
<!--more-->

## 一 容器生命周期管理

### run
docker run 命令用于创建一个新的容器并运行一个命令
```docker
docker run [OPTIONS] IMAGE [COMMAND] [ARG...]
```
|  OPTIONS   | 	说明  |
|  ----  | ----  |
| -d  | 后台运行容器，并返回容器ID |
| -i  | 以交互模式运行容器，通常与 -t 同时使用 |
| -t  | 为容器重新分配一个伪输入终端，通常与 -i 同时使用  |
| -m  | 设置容器使用内存最大值  |

### start
docker start 命令用于启动一个已经被停止的容器
``` docker
docker start [OPTIONS] CONTAINER [CONTAINER...]
```

### stop
docker stop 命令用于停止一个容器，如果容器已经停止，则什么都不做
``` docker
docker stop [OPTIONS] CONTAINER [CONTAINER...]
```

### restart
docker restart 命令用于重新启动一个已经在运行的容器
``` docker
docker restart [OPTIONS] CONTAINER [CONTAINER...]
```

### kill
docker kill 命令用于杀掉一个运行中的容器
``` docker
docker kill [OPTIONS] CONTAINER [CONTAINER...]
```

|  OPTIONS   | 	说明  |
|  ----  | ----  |
| -s  | 向容器发送一个信号 |

### rm
docker rm 命令用于删除一个或多个容器
``` docker
docker rm [OPTIONS] CONTAINER [CONTAINER...]
```

|  OPTIONS   | 	说明  |
|  ----  | ----  |
| -f | 通过SIGKILL信号强制删除一个运行中的容器 |
| -l | 移除容器间的网络连接，而非容器本身 |
| -v | 删除与容器关联的卷 |

### pause
docker pause 用于暂停容器中所有的进程
``` docker
docker pause [OPTIONS] CONTAINER [CONTAINER...]
```

### unpause
docker unpause 命令用于恢复容器中所有的进程
``` docker
docker unpause [OPTIONS] CONTAINER [CONTAINER...]
```


### create
docker create 创建一个新的容器但不启动它
``` docker
docker create [OPTIONS] CONTAINER [CONTAINER...]
```

### exec
docker exec 命令在运行的容器中执行命令
``` docker
docker exec [OPTIONS] CONTAINER COMMAND [ARG...]
```

|  OPTIONS   | 	说明  |
|  ----  | ----  |
| -d | 守护进程模式，在后台运行 |
| -i | 即使没有附加也保持 STDIN 打开 |
| -t | 分配一个伪终端 |

## 二 容器操作
### ps
docker ps 命令用于列出容器
``` docker
docker ps [OPTIONS]
```
默认情况下只会列出正在运行的容器

|  OPTIONS   | 	说明  |
|  ----  | ----  |
| -a | 显示所有色容器，包括未运行的 |
| -f | 根据条件过滤显示的内容 |
| --format | 指定返回值的模板文件 |
| -l | 显示最近创建的容器 |
| -n | 列出最近创建的n个容器 |
| -q | 静默模式，只显示容器编号 |

### inspect
docker inspect 命令用于获取容器/镜像的元数据
``` docker
docker inspect [OPTIONS] NAME|ID [NAME|ID...]
```
|  OPTIONS   | 	说明  |
|  ----  | ----  |
| -f | 指定返回值的模板文件 |
| -s | 显示总的文件大小 |
| --type | 为指定类型返回 JSON |

### top
docker top 查看容器中运行的进程信息，支持ps命令参数
``` docker
docker top [OPTIONS] CONTAINER [ps OPTIONS]
```

因为容器运行时不一定有 <font color=#c7254e size=3>bin/bash</font>终端来交互执行<font color=#c7254e size=3>top</font>，而且容器还不一定有 <font color=#c7254e size=3>top</font>命令  
这种情况下可以使用 <font color=#c7254e size=3>docker top</font>来实现查看 <font color=#c7254e size=3>container</font> 中正在运行的进程

### events
docker events 从服务器获取实时事件
``` docker
docker events [OPTIONS]
```
|  OPTIONS   | 	说明  |
|  ----  | ----  |
| -f | 根据条件过滤事件 |
| --since | 从指定的时间戳后显示所有事件 |
| --until | 流水时间显示到指定的时间为止 |
如果指定的时间是到秒级，需要将时间转成时间戳  
如果时间为日期的话，可以直接使用，如 --since="2016-07-01"


### logs
docker logs 命令用于获取容器的日志
``` docker
docker events [OPTIONS]
```

|  OPTIONS   | 	说明  |
|  ----  | ----  |
| -f | 跟踪日志输出 |
| --since | 显示总的文件大小 |
| -t | 显示时间戳 |
| --tail | 仅列出最新 N 条容器日志 |

### export
docker export 命令将文件系统作为一个tar归档文件导出到 STDOUT
``` docker
docker export [OPTIONS] CONTAINER
```
|  OPTIONS   | 	说明  |
|  ----  | ----  |
| -o | 将输入内容写到文件 |


### port
docker port 列出指定的容器的端口映射或者查找 PRIVATE_PORT NAT 到公开的端口

```
docker port [OPTIONS] CONTAINER [PRIVATE_PORT[/PROTO]]
```

## 容器rootfs命令

### commit
docker commit 命令从容器创建一个新的镜像
```
docker commit [OPTIONS] CONTAINER [REPOSITORY[:TAG]]
```
|  OPTIONS   | 	说明  |
|  ----  | ----  |
| -a | 提交的镜像作者 |
| -c | 使用Dockerfile 指令创建镜像 |
| -m | 提交时的说明文字 |
| -p | 在 commit 时，将容器暂停 |


### cp
docker cp 命令用于容器与主机之间的数据拷贝
```
docker cp [OPTIONS] CONTAINER:SRC_PATH DEST_PATH|-
docker cp [OPTIONS] SRC_PATH|- CONTAINER:DEST_PATH
```
