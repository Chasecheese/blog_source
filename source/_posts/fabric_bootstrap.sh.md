---
title: Fabric 搭建环境 bootstrap.sh做了哪些工作？
date: 2020-05-01 12:04:21
tags: Hyperledger
categories: Hyperledger
---

# bootstrap.sh安装Hyperledger Fabric-samples
在运行**bootstrap.sh**脚本安装HyperLedger Fabric-samples 和相关环境时，由于网络问题，进行了很多的手动下载和安装，所以，这次来分析一下这个脚本有哪些功能，做了哪几件事情。

<!-- more -->


# 主要安装部分：
*除了一些版本tag的检查外，这个脚本主要运行了这三大部分功能：除了一些版本tag的检查外，这个脚本主要运行了这三大部分功能：*

## 第一部分，下载fabric-samples的仓库：
这一步没有出现什么问题，很快就下载好了，这时，bootstrap.sh所在的目录下会出现一个fabric-samples文件夹
```bash
cloneSamplesRepo() {
    # clone (if needed) hyperledger/fabric-samples and checkout corresponding
    # version to the binaries and docker images to be downloaded
    if [ -d first-network ]; then
        # if we are in the fabric-samples repo, checkout corresponding version
        echo "===> Checking out v${VERSION} of hyperledger/fabric-samples"
        git checkout v${VERSION}
    elif [ -d fabric-samples ]; then
        # if fabric-samples repo already cloned and in current directory,
        # cd fabric-samples and checkout corresponding version
        echo "===> Checking out v${VERSION} of hyperledger/fabric-samples"
        cd fabric-samples && git checkout v${VERSION}
    else
        echo "===> Cloning hyperledger/fabric-samples repo and checkout v${VERSION}"
        git clone -b master https://github.com/hyperledger/fabric-samples.git && cd fabric-samples && git checkout v${VERSION}
    fi
}
```
## 第二部分，下载并安装最新版本的fabric和fabric-ca的二进制文件:

此处，两个二进制的文件，下载非常缓慢，用户选择可以手动下载，自行解压fabric和fabric-ca的包到fabric-simples的文件夹下，解压时两个包里相同的目录直接合并即可。最终会有两个目录`fabric-samples/bin`和`fabric-samples/config`

这样就可以跳过这一步命令的执行，具体的跳过方法可以看本文最后的说明。
```bash
# This will download the .tar.gz
download() {
    local BINARY_FILE=$1
    local URL=$2
    echo "===> Downloading: " "${URL}"
    curl -L --retry 5 --retry-delay 3 "${URL}" | tar xz || rc=$?
    if [ -n "$rc" ]; then
        echo "==> There was an error downloading the binary file."
        return 22
    else
        echo "==> Done."
    fi
}
```
`pullBinaries()`的前半部分是下载解压fabric，后半部分是下载解压fabric-ca
```bash

pullBinaries() {
    echo "===> Downloading version ${FABRIC_TAG} platform specific fabric binaries"
    download "${BINARY_FILE}" "https://github.com/hyperledger/fabric/releases/download/v${VERSION}/${BINARY_FILE}"
    if [ $? -eq 22 ]; then
        echo
        echo "------> ${FABRIC_TAG} platform specific fabric binary is not available to download <----"
        echo
        exit
    fi

    echo "===> Downloading version ${CA_TAG} platform specific fabric-ca-client binary"
    download "${CA_BINARY_FILE}" "https://github.com/hyperledger/fabric-ca/releases/download/v${CA_VERSION}/${CA_BINARY_FILE}"
    if [ $? -eq 22 ]; then
        echo
        echo "------> ${CA_TAG} fabric-ca-client binary is not available to download  (Available from 1.1.0-rc1) <----"
        echo
        exit
    fi
}
```
## 第三部分：安装Fabric相关的Docker镜像：
这一步的速度取决于Docker的下载速度，如果过非常慢，可以考虑替换其它的Docker源，改善会比较明显。
```bash
dockerPull() {
    #three_digit_image_tag is passed in, e.g. "1.4.6"
    three_digit_image_tag=$1
    shift
    #two_digit_image_tag is derived, e.g. "1.4", especially useful as a local tag for two digit references to most recent baseos, ccenv, javaenv, nodeenv patch releases
    two_digit_image_tag=$(echo $three_digit_image_tag | cut -d'.' -f1,2)
    while [[ $# -gt 0 ]]
    do
        image_name="$1"
        echo "====> hyperledger/fabric-$image_name:$three_digit_image_tag"
        docker pull "hyperledger/fabric-$image_name:$three_digit_image_tag"
        docker tag "hyperledger/fabric-$image_name:$three_digit_image_tag" "hyperledger/fabric-$image_name"
        docker tag "hyperledger/fabric-$image_name:$three_digit_image_tag" "hyperledger/fabric-$image_name:$two_digit_image_tag"
        shift
    done
}

pullDockerImages() {
    command -v docker >& /dev/null
    NODOCKER=$?
    if [ "${NODOCKER}" == 0 ]; then
        FABRIC_IMAGES=(peer orderer ccenv tools)
        case "$VERSION" in
        1.*)
            FABRIC_IMAGES+=(javaenv)
            shift
            ;;
        2.*)
            FABRIC_IMAGES+=(nodeenv baseos javaenv)
            shift
            ;;
        esac
        echo "FABRIC_IMAGES:" "${FABRIC_IMAGES[@]}"
        echo "===> Pulling fabric Images"
        dockerPull "${FABRIC_TAG}" "${FABRIC_IMAGES[@]}"
        echo "===> Pulling fabric ca Image"
        CA_IMAGE=(ca)
        dockerPull "${CA_TAG}" "${CA_IMAGE[@]}"
        echo "===> Pulling thirdparty docker images"
        THIRDPARTY_IMAGES=(zookeeper kafka couchdb)
        dockerPull "${THIRDPARTY_TAG}" "${THIRDPARTY_IMAGES[@]}"
        echo
        echo "===> List out hyperledger docker images"
        docker images | grep hyperledger
    else
        echo "========================================================="
        echo "Docker not installed, bypassing download of Fabric images"
        echo "========================================================="
    fi
}
```

------------
# 说明：跳过某些安装部分的方法：
**最开始，我进行了最为迷惑行为——注释/删除代码：**
*通过注释下面的相应代码，可以实现部分安装：*
```bash
if [ "$SAMPLES" == "true" ]; then
    echo
    echo "Clone hyperledger/fabric-samples repo"
    echo
    cloneSamplesRepo
fi
if [ "$BINARIES" == "true" ]; then
    echo
    echo "Pull Hyperledger Fabric binaries"
    echo
   pullBinaries
fi
if [ "$DOCKER" == "true" ]; then
    echo
    echo "Pull Hyperledger Fabric docker images"
    echo
    pullDockerImages
fi
```
**但是！不推荐这样做！因为这样比较蠢！**

**后来我醒悟了！！！可以先运行`./bootstrap.sh -h` 查看提示信息：**
```bash
printHelp() {
    echo "Usage: bootstrap.sh [version [ca_version [thirdparty_version]]] [options]"
    echo
    echo "options:"
    echo "-h : this help"
    echo "-d : bypass docker image download"
    echo "-s : bypass fabric-samples repo clone"
    echo "-b : bypass download of platform-specific binaries"
    echo
    echo "e.g. bootstrap.sh 2.1.0 1.4.6 0.4.18 -s"
    echo "would download docker images and binaries for Fabric v2.1.0 and Fabric CA v1.4.6"
}
```
可以看到，我们可以通过后缀，跳过上面三个部分任意部分的安装，最开始看的不仔细，我是通过注释相应流程代码实现的跳过安装,实在是有些蠢 :D