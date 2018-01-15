---
title: "一个简易的项目构建与部署脚本"
date: 2018-01-03T17:32:52+08:00
tags: [ "Java", "部署", "脚本" ]
categories: [ "研发" ]
---

在创始型的一个物流互联网平台团队里，有伙伴反复抱怨为了适用不同的环境构建、部署工程要多长多长的时间，而且一步小心就忘了啥，劳神费时！交接情况后，发现基本如此，并且一些当前高大上的一些做法也无法开展。

思前思后，觉得搞个脚本能解决些体力的事情。于是硬着头皮一遍凭着很久前对 shell 的一些记忆，一边百度，写了2个脚本。

<!--more-->

## 工程介绍

工程部署在阿里云上，采用了不少的阿里云服务，是个比较典型前后端完全分离的小型分布式平台：

 1. dubbo集群实现业务逻辑服务；
 2. SpringMVC集群实现Restful服务；
 3. nginx发布静态网站，负载均衡和流控等。


## 构建脚本

{{< highlight Bash "linenos=table" >}}
#!/bin/sh
    
ARTIFACT_LIST_SERVICE=(tms ams oms ams-web oms-web)
ARTIFACT_LIST_H5=(manager)
ARTIFACT_LIST_ALL=(${ARTIFACT_LIST_SERVICE[*]} ${ARTIFACT_LIST_H5[*]})
ARTIFACTS=${ARTIFACT_LIST_ALL[*]}
HAS_SERVICE=FALSE
HAS_H5=FALSE
PROFILE=
    
VERSION=
BUILD_NUMBER=
TAG=

ARGS=`getopt -o a:p:v:h --long artifact:,profile:,version:,help'build.sh' -- "$@"`
if [ $? != 0 ]; then
    echo -e "参数\033[49;31;4m错误\033[0m，程序已经终止。"
    exit 1
fi
eval set -- "${ARGS}"

while true
do
    case "$1" in
        -a|--artifact)
            UPPER_ARTIFACTS=$(echo $2 | tr '[a-z]' '[A-Z]')
            if [[ "$UPPER_ARTIFACTS" = "ALL" ]]; then
                ARTIFACTS=(${ARTIFACT_LIST_ALL[*]})
            elif [[ "$UPPER_ARTIFACTS" = "SERVICE" ]]; then
                ARTIFACTS=(${ARTIFACT_LIST_SERVICE[*]})
            elif [[ "$UPPER_ARTIFACTS" = "H5" ]]; then
                ARTIFACTS=(${ARTIFACT_LIST_H5[*]})
            else
                ARTIFACTS=($2)
            fi
            shift 2;
            ;;
        -p|--profile)
            PROFILE=$(echo $2 | tr '[a-z]' '[A-Z]')
            shift 2;
            ;;
        -v|--version)
            VERSION="$2"
            shift 2;
            ;;
        --help)
            echo "只支持以下参数："
            echo "-a | --artifact :  构建名称(${ARTIFACT_LIST_ALL[或构建类别(ALL, SERVICE, H5)"
            echo "-p | --profile :  构建名称的环境(test 或 prod)"
            echo "-v | --version : 构建产物的版本号，如果不指定则忽略。"
            echo "-h |--help : 显示帮助信息"
            exit 1
            ;;
        --)
            shift
            break
            ;;
        *)
            echo "Internal error!"
            exit 1
            ;;
    esac
done

if [[ $PROFILE = "PROD" || $PROFILE = "ALITEST" ]]; then
    echo -e "\033[33m构建${PROFILE}环境\033[0m"
 elif [[ -z "$PROFILE" ]]; then
    echo -e "\033[33m必须指定构建产物的部署环境\033[0m" 
else
    echo -e "\033[31;1m不能识别(支持)的构建环境：${PROFILE}\033[0m"
    exit 1
fi

for artifact in ${ARTIFACTS[*]}
do
    for service in ${ARTIFACT_LIST_SERVICE[*]}
    do
        if [[ "$artifact" = "$service" ]]; then
            HAS_SERVICE=TRUE
            break
        fi
    done
    for h5 in ${ARTIFACT_LIST_H5[*]}         
    do    
        if [[ "$artifact" = "$h5" ]]; then    
            HAS_H5=TRUE                    
            break    
        fi          
    done
done

BUILD_PATH="/opt/build"
BUILD_NUMBER_FILE="${BUILD_PATH}/build_number"
if [[ ! -f "$BUILD_NUMBER_FILE" ]]; then
    echo 0 > "$BUILD_NUMBER_FILE"
fi
BUILD_NUMBER=`cat $BUILD_NUMBER_FILE`

if [[ -z "$BUILD_NUMBER" ]]; then
    echo"\033[49;31;1m请在构建信息文件${BUILD_NUMBER_FILE}中指定初始构建非负整数)\033[0m"
    exit 1
elif [[ "$PROFILE" != "PROD"  ]]; then 发布的版本基于测试版本，不升级构建版本号
    BUILD_NUMBER=$[BUILD_NUMBER+1]
fi
echo $BUILD_NUMBER > "$BUILD_NUMBER_FILE"

if [[ ! -z "$VERSION" ]]; then
    TAG="${VERSION}-${BUILD_NUMBER}"
else
    TAG="$BUILD_NUMBER"
fi

# 创建 BUILD_NUMBER 产出物的跟目录 
BUILD_REPO="${BUILD_PATH}/$(echo $PROFILE | tr '[A-Z]a-z]')_${BUILD_NUMBER}"
mkdir "$BUILD_REPO" > /dev/null

# $1=svnSource, $2=tag, $3=logFile, ret=tagUrl
SVN_TAG_URL=
function create_svn_tag
{
    CREATE_TIME=`date "+%Y-%m-%d %H:%M:%S"`
    SVN_TAG_URL="${1}Tags/${TAG}"
    svn copy ${1} ${SVN_TAG_URL} -m "构建服务器创建于 ${CREATE_TIME}"
    echo -n "$CREATE_TIME" >> "${3}"
    echo  "   创建源代码标签：${2}" >> "${3}"
    echo  "   创建源代码标签：${2}" 
}

SVN_REPO_BASE="https://116.228.168.118:700/svn/intelligent-platform/nhxtx"
SOURCE_BASE_DIR="/opt/svn/nhxtx/${TAG}"
if [[ ! -d $SOURCE_BASE_DIR ]]; then
    if [[ "$PROFILE" = "PROD" ]]; then
        echo -e "\033[49;31;1m版本${TAG}尚未进行过测试，确定直接发环境吗？\033[0m"
        # 这里添加交互功能
        exit 0
    else
        mkdir $SOURCE_BASE_DIR || exit 1
        create_svn_tag $SVN_REPO_BASE $TAG ${BUILD_REPO}/info
        svn co $SVN_TAG_URL $SOURCE_BASE_DIR
        echo  "   更新代码到目录：${SOURCE_BASE_DIR}" >> ${BUILD_REinfo  
    fi
else
    cd $SOURCE_BASE_DIR
    svn update
fi

if [[ $HAS_SERVICE = "TRUE" ]]; then
    echo -e '\033[32m =======================================\033[0m'
    echo -e '\033[32m *         编译【Java】工程...         *\033[0m'
    echo -e '\033[32m =======================================\033[0m'
    cd $SOURCE_BASE_DIR
    mvn clean package -Pprod -Dmaven.test.skip=true -U || exit -1
    
    # 拷贝Java二进制代码到 BUILD_NUMBER 目录
    echo -e "\033[33m拷贝Java工程编译产物到目录${JAVA_BUILD_REPO}\033[0m"
    for artifact in ams ams-web oms oms-web tms
    do
    echo copying "/opt/svn/nhxtx/${artifatarget/${artifact}-1.0.0-SNAPSHOT.jar"
        mv ${SOURCE_BASE_DIR}/${artifatarget/${artifact}-1.0.0-SNAPSHOT."${BUILD_REPO}/${artifact}.jar"
    done 
    
    echo  -n `date "+%Y-%m-%d %H:%M:%S"` >> "${BUILD_REPO}/info"
    echo  "   构建[服务]成功" >> "${BUILD_REPO}/info"
fi

if [[ "$HAS_H5" = "TRUE" ]]; then
    echo -e '\033[32m =====================================\033[0m'
    echo -e '\033[32m *        编译 【OMS】 静态工程...   *\033[0m'
    echo -e '\033[32m =====================================\033[0m' 
    sleep 1

    cd ${SOURCE_BASE_DIR}/omsH5
    npm install || exit -1
    npm run build-$(echo ${PROFILE} | tr '[A-Z]' '[a-z]') || exit -1

    # 删除main文件
 
    # 拷贝H5静态代码到 BUILD_NUMBER 目录
    echo -e "\033[32m拷贝 OMS H5 工程编译结果到目录${JAVA_BUILD_REPO}\0m"
    mv ${SOURCE_BASE_DIR}/omsH5/dist "${BUILD_REPO}/"
    mv "${BUILD_REPO}/dist" "${BUILD_REPO}/manager"
    echo -n `date "+%Y-%m-%d %H:%M:%S  "` >> "${BUILD_REPO}/info"
    echo "   构建[H5]成功" >> "${BUILD_REPO}/info"
fi
{{< /highlight >}}


## 部署脚本

{{< highlight bash >}}
#!/bin/bash
    
PROFILE=prod
SERVER_SERVICE=(10.30.*.* 10.30.*.*)
SERVER_WEB=(10.31.*.* 10.31.*.*)
SERVER_H5=(10.31.*.* 10.31.*.*)

ARTIFACT_SERVICE=(tms ams oms)
ARTIFACT_WEB=(ams-web oms-web)
ARTIFACT_JAVA=(${ARTIFACT_SERVICE[*]} ${ARTIFACT_WEB[*]})
ARTIFACT_H5=(manager)
ARTIFACT_ALL=(${ARTIFACT_JAVA[*]} ${ARTIFACT_H5[*]})
HOST_DIR="/opt/product"

function usage
{
    echo ""
    echo " -h | --host:         部署的目标主机"
    echo " -d | --dir:          部署的目标目录。默认为：${HOST_DIR}"
    echo " -s | --service-name: 部署的的服务名称，可用的服${ARTIFACT_ALL[@]}。默认为全部服务"
    echo " -v | --version:      部署的服务的版本。默认为最新的生产构建版本"
    echo " -p | --profile:      部署的环境。"
    echo " --init:              部署后初始化目标机器，将部署的物设置为自动重启的系统服务"
    echo " --start:             部署后启动(重启)服务"
    echo ""
}

ARGS=`getopt -o h::d::s::v:p: --lhost::,dir::,service-name::,version:,init,start,profile -n 'deploy.sh'"$@"`
if [ $? != 0 ]; then
    usage && exit 1
fi
eval set -- "${ARGS}"

ARTIFACT_PATH="/opt/build"
ARTIFACT_VERSION=

SPEC_HOST=
ARTIFACTS=(${ARTIFACT_ALL[*]})

SPEC_INIT=
SPEC_START=
while true
do
    case "$1" in
        -h|--host) 
            if [[ ! -z "$2" ]]; then
                SPEC_HOST=($2)
            fi
            shift 2;
            ;;
        -d|--dir)
            if [[ ! -z "$2" ]]; then
                HOST_DIR=$2
            fi
            shift 2
            ;;
        -s|--service-name)
            if [[ ! -z "$2" ]]; then
                ARTIFACTS=($2)
            fi
            shift 2
            ;;
        -v|--version)
            if [[ ! -z "$2" ]]; then
                ARTIFACT_VERSION=$2
            fi
            shift 2
            ;;
        -p|--profile)
            if [[ ! -z "$2" ]]; then
                PROFILE=$2
            fi
            shift 2
            ;;
        --init)
            SPEC_INIT="true"
            shift
            ;;
        --start)
            SPEC_START="true"
            shift
            ;;
        --)
            shift
            break
            ;;
        *)
            usage && exit 1
            ;;
    esac
done

if [[ -z "$ARTIFACT_VERSION" ]]; then
   if [[ -f "${ARTIFACT_PATH}/build_number" ]]; then
       ARTIFACT_VERSION=`cat ${ARTIFACT_PATH}/build_number`
   else
       echo -e "\033[32;49;1m 构建文件[${ARTIFACT_PATH}/build_num丢失，请修复或通过-v 或 --version 指定部署的版本号" 
       exit 1
   fi
fi

# 1=service, 2=host, 3=host-dir
function transter()
{
    ssh "$2" "[[ -d $3 ]] || mkdir $3"

    DST_REPO="${2}:${3}"
    SRC_REPO="${ARTIFACT_PATH}/${PROFILE}_${ARTIFACT_VERSION}"

    SRC_ARTIFACT=
    SCP_FLAG=""
    if [[ -f "${SRC_REPO}/${1}.jar" ]]; then
        SRC_ARTIFACT="${SRC_REPO}/${1}.jar"
    elif [[ -d "${SRC_REPO}/${1}" ]]; then
        SRC_ARTIFACT="${SRC_REPO}/${1}"
        SCP_FLAG="-r "
    else
        echo -e "\033[32;49;1m 资源[${SRC_REPO}/${1}]不存在\033[39;49;0m"
        exit 1
    fi    

    echo "copying ${SRC_ARTIFACT} to ${DST_REPO}/" 
    scp ${SCP_FLAG}${SRC_ARTIFACT} ${DST_REPO}/ > /dev/null
    if [ $? == 0 ]; then
        if [[ -z "$SCP_FLAG" ]]; then
            ssh "$HOST" "chmod u+x ${3}/${1}.jar"
        fi              
        echo -n `date "+%Y-%m-%d %H:%M:%S"` >> "${SRC_REPO}/info"
        echo "   同步[SRC_ARTIFACT]到${DST_REPO}/" >> "${SRC_REPO}/info"

        echo "${ARTIFACT_VERSION}" > "TMP_${ARTIFACT_VERSION}"
        scp "TMP_${ARTIFACT_VERSION}" "${DST_REPO}/version" > /dev/null
        rm -f "TMP_${ARTIFACT_VERSION}"
    else
        echo -e "\033[31;1m 拷贝资源[SRC_ARTIFACT]失败\033[0m"
        exit 1
    fi
    return 0
}

function init()
{
    if [[ "${ARTIFACT_JAVA[*]}" =~ "$3" ]]; then
        ssh $1 "ln -sif ${2}/${3}.jar /etc/init.d/${3}"
        TMP_CONF_FILE="TMP_CONF_${ARTIFACT_VERSION}"
        echo "JAVA_OPTS=\"-Djava.security.egd=file:/dev/./urandom -ser-Xms1024m -Xmx2048m\"" > "$TMP_CONF_FILE"
        echo "LOG_FOLDER=\"/var/log/paltformname/${ARTIFACT_VERSION}\"""$TMP_CONF_FILE"
        echo "RUN_ARGS=\"--spring.profiles.active=${PROFI--application.build-number=${ARTIFACT_VERSION}\"""$TMP_CONF_FILE"
        echo -n "# 自动配置于" >> "$TMP_CONF_FILE"
        echo `date "+%Y-%m-%d %H:%M:%S"` >> "$TMP_CONF_FILE"
        
        CONF_DIR="/etc/hd"
        ssh "$1" "[[ -d ${CONF_DIR} ]] || mkdir ${CONF_DIR}"
        scp "$TMP_CONF_FILE" "$1:${CONF_DIR}/${3}.conf" > /dev/null
        
        rm -rf $TMP_CONF_FILE
        ssh $1 "chkconfig --add ${3} && chkconfig ${3} on"
        
        if [[ $? == 0 ]]; then
            echo -e "\033[32m 初始化主机[${1}]服务[${3}]成功 \033[0m" 
        else
            echo -e "\033[31m 初始化主机[${1}]服务[${3}]失败 \033[0m"
        fi
        return 0
    else
       echo "${3}不是JAVA构件，不能进行服务化初始化"
       return 1
    fi
}

for ARTIFACT in ${ARTIFACTS[*]}
do
    if [[ ! "${ARTIFACT_ALL[*]}" =~ "${ARTIFACT}" ]]; then
        echo -e "\033[31;1m 不支持服务[${ARTIFACT}]的部署\033[0m"
        continue
    fi
  
    ARTIFACT_DST_HOST=
    if [[ ! -z "$SPEC_HOST" ]]; then
        ARTIFACT_DST_HOST=(${SPEC_HOST[*]})
    elif [[ "${ARTIFACT_SERVICE[*]}" =~ "$ARTIFACT" ]]; then
        sleep 2     #  让服务有足够的时间启动
        ARTIFACT_DST_HOST=${SERVER_SERVICE[*]}
    elif [[ "${ARTIFACT_WEB[*]}" =~ "$ARTIFACT" ]]; then
        ARTIFACT_DST_HOST=${SERVER_WEB[*]}
    elif [[ "${ARTIFACT_H5[*]}" =~ "$ARTIFACT" ]]; then
        ARTIFACT_DST_HOST=${SERVER_H5[*]}
    else
        echo -e "\033[31;1m 脚本出错了，请报告运维人员修复\033[0m"
        continue
    fi
    
    echo -e "\033[32m \n正在同步服务[${ARTIFACT}]...... \033[0m"
    for HOST in ${ARTIFACT_DST_HOST[*]}
    do
        transter $ARTIFACT $HOST $HOST_DIR
        if [[ $? == 0 ]]; then
            if [[ ! -z "$SPEC_INIT" ]]; then
                init $HOST $HOST_DIR $ARTIFACT && [[ ! -z "$SPEC_START"&& ssh $HOST "service ${ARTIFACT} start"
            elif [[ "${ARTIFACT_JAVA[*]}" =~ "$ARTIFACT" ]]; then
               CONF_FILE="/etc/hd/${ARTIFACT}.conf"
               ssh $HOST "[[ ! -f $CONF_FILE ]]"
               if [[ $? == 0 ]]; then
                   echo "主机[${HOST}]尚未配置服务[${ARTIFACT}]，建deploy.sh -h${HOST} -d${HOST_DIR} --i-s${ARTIFACT}进行初始化"
               else
                   ssh $HOST "sed -i 's/\(--application.build-number=\)igit:]]*/\1${ARTIFACT_VERSION}/g' $CONF_FILE"
                   ssh $HOST "sed -i 's/^LOG_FOLDER=.*/LOG_FOLDER=\"\/vlog\/paltformname\/${ARTIFACT_VERSION}\"/g' $CONF_FILE"
                   ssh $HOST "[[ -d /var/log/paltformname ]] || mkdir /log/paltformname"
                   ssh $HOST "[[ -d /var/paltformname/${ARTIFACT_VERSION} ]] || mkdir /var/paltformname/${ARTIFACT_VERSION}"
                   [[ ! -z "$SPEC_START" ]] && ssh $HOST "serv${ARTIFACT} restart"
               fi
            fi
        fi
    done
done
{{< /highlight >}}