## 训练任务镜像规范

自定义的训练任务镜像需要满足以下条件

1. 设置镜像环境编码

centos基础环境用指令`ENV LANG en_US.UTF-8`设置；

ubuntu基础环境用指令`ENV LANG C.UTF-8`设置；

2. 安装s3cmd/zip命令

centos基础环境使用指令`RUN yum install -y s3cmd zip`安装

ubuntu基础环境使用指令`RUN apt-get install -y s3cmd zip`安装

训练任务自定义镜像的Dockerfile示例：

```dockerfile
FROM ubuntu:14.04
ENV LANG C.UTF-8
ENV TZ=Asia/Shanghai
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone
RUN apt-get update && apt-get install -y \
   build-essential \
   git \
   graphviz \
   liblapack-dev \libopenblas-dev \
   libopencv-dev \
   python-dev \
   python-numpy \
   python-pip \
   python-setuptools \
   wget \
   libcurl4-openssl-dev \
   libssl-dev && \
   rm -rf /var/lib/apt/lists/*

#git clone --recursive https://github.com/apache/incubator-mxnet mxnet --branch v0.11.0
ADD mxnet /root/mxnet

COPY ./s3_filesys.cc /root/mxnet/dmlc-core/src/io

RUN cd /root/mxnet && \
     make -j $(nproc) USE_CUDA=0 USE_CUDNN=0 USE_BLAS=openblas USE_LAPACK=1 USE_S3=1 USE_DIST_KVSTORE=1 USE_OPENCV=0

# Install Python package

RUN cd /root/mxnet/python && python setup.py install
# install S3 tool and uncompression tool(tar should've been installed by default)
RUN pip install s3cmd
RUN apt-get update && apt-get install -y --no-install-recommends unzip zip && rm -rf /var/lib/apt/lists/*
# To avoid issue with lib1394
RUN ln /dev/null /dev/raw1394

WORKDIR /root/mxnetl
```

## 交互式建模任务镜像规范

1. 如果要使用交互式建模任务的在线jupyter功能，对应的交互式建模任务镜像中需要安装jupyter，由于本系统对jupyter的展示页面做了定制，所以不建议用户自己打包含jupyter环境的交互式建模任务镜像，可以基于系统提供的交互式建模任务基础镜像打新镜像，交互式建模任务基础镜像是基于ubuntu16.04版本，内置了s3cmd，zip，s3fs，jupyter内核[Python3\Perl\R]以及常见python库，支持中文环境。[点击下载交互式建模任务基础镜像](http://10.24.127.4:9060/deeplearn-server/api/v1/download/image?name=expert_ubuntu)。

2. 如果不需要使用交互式建模任务的在线jupyter功能，因为启动交互式建模任务时需要挂载ceph目录，所以对应的交互式建模任务镜像中要支持bash指令，还需要安装s3fs。

首先要生成run.sh脚本文件，本地创建run.sh文件，在run.sh文件中输入以下内容：

```bash
#!/bin/bash
echo "***Start to create s3 conf***"
data=$JOB_PREFIX
temp=${data:5}
bucket=${temp%%/*}
mountdir=/${temp#*/}touch /jupyter/.passwd-s3fs
echo "$ACCESS_KEY:SECRET_KEY" > /jupyter/.passwd-s3fs
chmod 600 /jupyter/.passwd-s3fs
echo "***Start to mount S3***"
rm -rf /workdir/*
mkdir -p /workdir
s3fs $bucket:$mountdir /workdir -o passwd_file=/jupyter/.passwd-s3fs -o url=http://$S3_ENDPOINT -o use_path_request_style -o uid=0,umask=227,gid=0 -o use_cache=/tmp/cache
if [ $? -ne 0 ];then echo "S3 mount failed"; exit 1; fi
```

使用指令`COPY ./run.sh /run.sh`在镜像环境中设置s3相关脚本文件；

其次使用指令`COPY chmod u+x /run.sh`设置脚本可执行；

最后安装s3fs，

centos基础环境，可以用指令`RUN yum install -y s3fs`；

ubuntu基础环境，可以用指令`RUN apt-get install -y s3fs`。

镜像的启动指令需要先执行run.sh脚本。

不包含jupyter的交互式建模任务自定义镜像的Dockerfile示例：

```dockerfile
FROM ubuntu:14.04
RUN  apt-get update && apt-get install -y s3fs
COPY ./run.sh /run.sh
RUN chmod u+x /run.sh
CMD /run.sh && bash
```
