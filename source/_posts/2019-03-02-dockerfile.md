---
layout:     post
title:      "Dockerfile 语法梳理"
subtitle:   "Dockerfile 语法梳理"
date:       2019-03-02 13:30:00
author:     "Psl"
catalog:    true
tags:
  - Docker
  - Dockerfile
---

## Dockerfile语法梳理

### FROM

from 后面接base image

eg:
```dockerfile
FROM scratch
FROM centos
FROM ubuntu
```
> 尽量使用官方的image 作为base image

### LABEL

```dockerfile
LABEL maintainer="username@gmail.com" 
LABEL version="1.0" 
LABEL description="this is description" 
```

>MetaData 不可少

### RUN
执行命令并创建新的IMAGE LAYER

为了美观，复杂的RUN用反斜杠换行，避免无用分层，合并多条命令成一行。

#### 最佳实践
```dockerfile
RUN yum update && yum install -y vim \
python-dev
```

```dockerfile
RUN apt-get update && apt-get install -y perl \
pwgen --no-install-recommends && rm -rf \
/var/lib/apt/lists/*     #清理cache
```

```dockerfile
RUN /bin/bash -c 'source $HOME/.bashrc;echo #HOME'
```

### WORKDIR

```dockerfile
WORKDIR test       # 如果没有会自动创建test文件夹
WORKDIR demo
RUN pwd            # 打印/test/demo
```

> 尽量使用WORKDIR,不要使用RUN cd,尽量使用绝对路径

### ADD and COPY

```dockerfile
ADD hello /
ADD test.tar.gz / # 添加到根目录并解压缩
WORKDIR /root
ADD hello test /    # /root/test/hello
WORKDIR /root
COPY hello test /
```

> 大部分情况COPY优于，ADD有额外的解压功能，添加远程文件或目录用curl或wget

### ENV

```dockerfile
ENV MYSQL_VERSION 5.6 #常量
RUN apt-get install -y mysql-server= "${MYSQL_VERSION}" \
&& rm -rf /var/lib/apt/lists/*
```

### CMD & ENTRYPOINT

CMD:设置容器启动后默认执行的命令和参数
如果docker run指定了其它的命令，则忽略CMD命令
定义多个CMD,只有最后一个会执行

```bash
docker run <image>
docker run -it <image> /bin/bash    //此命令会忽略CMD中的命令
```


ENTRYPOINT:设置容器启动时运行的命令
让容器已应用程序或者服务的方式执行
不会被忽略，一定会执行


#### SHELL & EXEC

SHELL:

```dockerfile
RUN apt-get install -y vim
CMD echo "hello docker"
ENTRYPOINT echo "hello docker"
```

EXEC:

```dockerfile
RUN ["apt-get", "install", "y", "vim"]
CMD ["/bin/echo", "hello docker"]
ENTRYPOINT ["/bin/echo", "hello docker"]
```

EXEC方式需要指明运行环境，eg:

```dockerfile
FROM centos
ENV name word
ENTRYPOINT ["/bin/bash", "c", "echo hello $name"]
```

更多详见[扩展阅读](https://docs.docker.com/engine/reference/builder/)


## Dockerfile实战

```bash
mkdir flask-hello-word
cd flask-hello-word
vim app.py
```

app.py内容

```python
from flask import Flask
app = Flask(__name__)
@app.route('/')
def hello():
    return "hello docker"
if __name__ == '__main__':
    app.run(host="0.0.0.0", port=5000)
```
编写Dockerfile

```dockerfile
FROM python:2.7
LABEL maintainer="peng.shiliang<1390509500@qq.com>"
RUN pip install flask
COPY app.py /app/       # app后面必须接/，否则会当作文件
WORKDIR /app
EXPOSE 5000             # 端口映射,保证远程能够访问
CMD ["python", "app.py"]
```

执行
```
docker build -t pengshiliang/flask-hello-word .
docker push pengshiliang/flask-hello-word:latest
```

运行flask-hello-word
```bash
docker run -d --name=demo pengshiliang/flask-hello-word     //--name 便于docker container 操作
docker exec -it demo ip a       //查看docker容器ip
curl <demo ip>      //输出hello docker
```















