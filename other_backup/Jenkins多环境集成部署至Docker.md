## 一、放于开发侧方式：

#### 1.在根目录分别放上不同环境的配置文件，后跟不同配置名：

```
application-local.yml
application-hwcloud.yml
application-dev.yml
application-test.yml
application-stg.yml
application-prod.yml
```

application-local.yml示例：

```yml
spring:
  application:
    name: inmessage
  profiles:
      active: @profileActive@ # dev, test, prod, 这个参数从下述的maven配置文件的properties元素里配置，然后就去读取响应的文件，例如application-dev.yml
```

application-dev.yml示例：

```yml
server:
  port: 30012

# 公共属性配置
wind:
  dataMachineId: 12 # 必填。id生成器数据机器标识配置，范围1-1023，需要唯一
  # apiVersion:
  aspect:
    apilog: true # 接口日志切面开关(默认false)
  datasources:
    inmessage:
      url: jdbc:mysql://xxx.xxx.xx.xx:3306/szcdm_inmessage?useUnicode=true&characterEncoding=utf8&serverTimezone=Asia/Shanghai
      username: xxx
      password: xxxx
      mapper-interface-location: com.xxx.xx.service.biz.inmessage.mapper
      transaction-base-packages: com.xxx.xx.service.biz.inmessage.service
      mapper-xml-location: classpath*:com/xxx/xx/service/biz/inmessage/mybatis/mapper/*Mapper.xml
```

#### 2.Maven配置如下：

```xml
    <profiles>
        <profile>
            <id>local</id>
            <properties><profileActive>local</profileActive></properties>
            <activation><activeByDefault>true</activeByDefault></activation>
        </profile>
        <profile>
            <id>hwcloud</id>
            <properties><profileActive>hwcloud</profileActive></properties>
        </profile>
        <profile>
            <id>dev</id>
            <properties><profileActive>dev</profileActive></properties>
        </profile>
        <profile>
            <id>test</id>
            <properties><profileActive>test</profileActive></properties>
        </profile>
        <profile>
            <id>stg</id>
            <properties><profileActive>stg</profileActive></properties>
        </profile>
        <profile>
            <id>prod</id>
            <properties><profileActive>prod</profileActive></properties>
        </profile>
    </profiles>

    <build>
        <resources>
            <resource>
                <directory>src/main/resources</directory>
                <filtering>true</filtering>
                <excludes>
                    <exclude>application-local.yml</exclude>
                    <exclude>application-hwcloud.yml</exclude>
                    <exclude>application-dev.yml</exclude>
                    <exclude>application-test.yml</exclude>
                    <exclude>application-stg.yml</exclude>
                    <exclude>application-prod.yml</exclude>
                </excludes>
            </resource>
            <resource>
                <directory>src/main/resources</directory>
                <filtering>true</filtering>
                <includes>
                    <include>application.yml</include>
                    <include>application-${profileActive}.yml</include>
                    <include>*/**.*</include>
                </includes>
            </resource>
        </resources>

        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>                    <mainClass>com.szatc.cdm.service.biz.flightcollab.FlightcollabApplication</mainClass>
                </configuration>
            </plugin>
        </plugins>
    </build>
```

#### 3.打包时加上profile命令即可

```shell
clean install deploy -e -X -U -P test
```

#### 4.打完包后打镜像

```shell
# 构建并推送eureka镜像
echo '================上传jar包成功==============='

echo '================构建镜像开始================'

# -------------------------------------------------------------------------------------
# -> jenkins项目名称
JENKINS_PROJECT_NAME='cdm-test-image-all'

# -> jar包项目名称
JAR_PROJECT_NAME='wind-cloud-support-service-eureka'
# -> jar包版本名称
JAR_PROJECT_VERSION='1.0.0-SNAPSHOT'

# -> POM.xml文件路径
POM_PATH='/wind-cloud-atm/wind-cloud-support-service/wind-cloud-support-service-eureka'

# -------------------------------------------------------------------------------------
# -> profile
PROFILE='test'

# -------------------------------------------------------------------------------------
# -> 镜像脚本名称
DOCKER_FILE_NAME='dockerfile-'${PROFILE}

# -> 镜像名称
IMAGE_NAME=${PROFILE}'-'${JAR_PROJECT_NAME}
# -> 镜像tag
IMAGE_TAG=${JAR_PROJECT_VERSION} #`date +%Y%m%d-%H%M%S`按时间打tag

#Nexus仓库镜像地址
NEXUS_IP='192.168.53.62:8082'
NEXUS_USER_NAME='jenkins'
NEXUS_PASSWORD='cdm12345'

# -------------------------------------------------------------------------------------
# jenkins项目工作目录
WORK_PATH='/var/jenkins_home/workspace/'${JENKINS_PROJECT_NAME}${POM_PATH}

# jar包名称
JAR_NAME=${JAR_PROJECT_NAME}'-'${JAR_PROJECT_VERSION}'.jar'

# -------------------------------------------------------------------------------------
# Build image.
cd ${WORK_PATH} 

rm -rf ./docker
mkdir docker
cp ./${DOCKER_FILE_NAME} ./docker/Dockerfile
cp './target/'${JAR_NAME} ./docker/app.jar
cd ./docker

docker build -t ${NEXUS_IP}'/'${IMAGE_NAME}':'${IMAGE_TAG} . #&>/dev/null

echo '================推送镜像开始================'
docker login --username=${NEXUS_USER_NAME} --password=${NEXUS_PASSWORD} ${NEXUS_IP}
docker push ${NEXUS_IP}'/'${IMAGE_NAME}':'${IMAGE_TAG}
# 推送完成后删除本地镜像
docker rmi ${NEXUS_IP}'/'${IMAGE_NAME}':'${IMAGE_TAG}
```

打镜像脚本：

```shell
# 基础镜像使用java
FROM java:8
# 其效果是在主机 /var/lib/docker 目录下创建了一个临时文件，并链接到容器的/tmp
VOLUME /tmp /app
# 将jar包添加到容器中并更名为app.jar
ADD app.jar app.jar
# EXPOSE 30000
ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","-Xms512M","-Xmx800M","/app/app.jar"]
```

#### 5.打完镜像后发布

```shell
# -> profile
PROFILE='test'

# -> jar包项目名称
JAR_PROJECT_NAME='wind-cloud-support-service-eureka'
# -> jar包/镜像版本名称
JAR_PROJECT_VERSION='1.0.0-SNAPSHOT'

# -> 容器名称
CONTAINER_NAME=${PROFILE}'-eureka-1'

# -> 宿主机映射IP
HOST_PORT='30000'
# -> 容器映射IP
DOCKER_PORT='30000'

# -------------------------------------------------------------------
#Nexus仓库镜像地址
NEXUS_IP='192.168.53.62:8082'
NEXUS_USER_NAME='jenkins'
NEXUS_PASSWORD='cdm12345'

echo '================处理'${CONTAINER_NAME}'================'
# -------------------------------------------------------------------
# -> 镜像名称
IMAGE_NAME=${PROFILE}'-'${JAR_PROJECT_NAME}
# -> 镜像tag
IMAGE_TAG=${JAR_PROJECT_VERSION} #`date +%Y%m%d-%H%M%S`按时间打tag

echo '================删除容器及镜像================'
# Stop container, and delete the container.
CONTAINER_ID=`docker ps | grep ${CONTAINER_NAME} | awk '{print $1}'`
if [ -n "$CONTAINER_ID" ]; then
    docker stop $CONTAINER_ID
    docker rm $CONTAINER_ID
else #如果容器启动时失败了，就需要docker ps -a才能找到那个容器
    CONTAINER_ID=`docker ps -a | grep ${CONTAINER_NAME} | awk '{print $1}'`
    if [ -n "$CONTAINER_ID" ]; then  # 如果是第一次在这台机器上拉取运行容器，那么docker ps -a也是找不到这个容器的
        docker rm $CONTAINER_ID
    fi
fi

# Delete image early version.
IMAGE_ID=`docker images | grep ${IMAGE_NAME} | awk '{print $3}'`
if [ -n "${IMAGE_ID}" ];then
    docker rmi ${IMAGE_ID}
fi

# Delete unused volume 
docker volume rm $(docker volume ls -qf dangling=true)

echo '================获取新镜像及容器================'
docker login --username=${NEXUS_USER_NAME} --password=${NEXUS_PASSWORD} ${NEXUS_IP}
docker pull ${NEXUS_IP}'/'${IMAGE_NAME}':'${IMAGE_TAG}

docker run -itd --name ${CONTAINER_NAME} -p ${HOST_PORT}:${DOCKER_PORT} --network host ${NEXUS_IP}'/'${IMAGE_NAME}':'${IMAGE_TAG}
```

## 二、放于Jenkins侧并使用动态变量方式

#### 1.application就一个配置文件，使用动态变量

```yml
spring:
  application:
    name: systemmana # 唯一，必填

server:
  port: ${cdm_app_port:30013}  # 唯一，必填

wind: # 公共属性配置
  # apiVersion:
  dataMachineId: 13 # id生成器数据机器标识配置，范围1-1023，需要唯一
  aspect:
    apilog: true # 接口日志切面开关(默认false)
  datasources:
    flightcollab:
      url: ${cdm_app_db_url:jdbc:mysql://192.168.53.60:3306/szcdm_syst?useUnicode=true&characterEncoding=utf8&serverTimezone=Asia/Shanghai}
      username: ${cdm_app_db_username:root}
      password: ${cdm_app_db_password:123456}
      mapper-interface-location: com.szatc.cdm.service.basic.systemmana.mapper
      transaction-base-packages: com.szatc.cdm.service.basic.systemmana.service
      mapper-xml-location: classpath*:com/szatc/cdm/service/basic/systemmana/mybatis/mapper/*Mapper.xml
```

#### 2.maven的XML文件，无需特别配置

```xml
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <mainClass>com.szatc.cdm.service.biz.flightcollab.FlightcollabApplication</mainClass>
                </configuration>
            </plugin>
        </plugins>
    </build>

    <!-- jar包引用地址 -->
    <repositories>
        <!-- 私服jar包地址 -->
        <repository>
            <id>nexus-maven-central</id>
            <url>http://192.168.53.62:8081/repository/nexus-maven-central/</url>
            <releases><enabled>true</enabled></releases>
            <snapshots><enabled>false</enabled></snapshots>
        </repository>

        <repository>
            <id>nexus-maven-releases</id>
            <name>nexus-maven-releases</name>
            <url>http://192.168.53.62:8081/repository/maven-releases/</url>
            <releases><enabled>true</enabled></releases>
            <snapshots><enabled>false</enabled></snapshots>
        </repository>

        <repository>
            <id>nexus-maven-snapshots</id>
            <name>nexus-maven-snapshots</name>
            <url>http://192.168.53.62:8081/repository/maven-snapshots/</url>
            <releases><enabled>false</enabled></releases>
            <snapshots><enabled>true</enabled></snapshots>
        </repository>

    </repositories>

    <pluginRepositories>
        <pluginRepository>
            <id>nexus-maven-central</id>
            <url>http://192.168.53.62:8081/repository/nexus-maven-central/</url>
            <releases><enabled>true</enabled></releases>
            <snapshots><enabled>true</enabled></snapshots>
        </pluginRepository>
    </pluginRepositories>

    <!-- jar包发布地址 -->
    <distributionManagement>
        <repository>
            <id>nexus-maven-releases</id>
            <name>nexus-maven-releases</name>
            <url>http://192.168.53.62:8081/repository/maven-releases/</url>
        </repository>

        <snapshotRepository>
            <id>nexus-maven-snapshots</id>
            <name>nexus-maven-snapshots</name>
            <url>http://192.168.53.62:8081/repository/maven-snapshots/</url>
        </snapshotRepository>
    </distributionManagement>
```

#### 3.jenkins打包指令

jenkins配置参数化构建。打镜像时直接指定动态变量的值。这些值也可以在k8s里用key-value覆盖。

脚本如下：

```shell
# --------------------上传jar包成功，开始构建镜像--------------------------------------
# -------------------------------------------------------------------------------------
# ================进入工作路径================
cd ${WORKSPACE}/szatc-cdm/${cdmservice}

# -> 镜像标签(DEV 固定是latest)
IMAGE_TAG=${cdmtag}
#IMAGE_TAG='latest'
# -> 镜像名称
IMAGE_NAME=${cdmservice}
# -> 仓库配置
NEXUS_IP='192.168.53.62:8082'
NEXUS_USER_NAME='jenkins'
NEXUS_PASSWORD='cdm12345'
# ---------------------------------参数配置------------------------------------------
cdm_redis_host='192.168.53.60'
cdm_redis_port=26379

cdm_eureka_url='http://192.168.53.65:30000/eureka/'

cdm_eureka_use_prefer_ip=true

cdm_app_db_username='root'
cdm_app_db_password='123456'

if   [ ${IMAGE_TAG} = '' ]
then
    IMAGE_TAG=`date +%Y%m%d%H%M`
fi

if   [ ${cdmservice} = 'szatc-cdm-service-basic-approve' ]
then
    cdm_app_port=30014
	cdm_app_db_url='jdbc:mysql://192.168.53.60:23306/szcdm_appr?useUnicode=true&characterEncoding=utf8&serverTimezone=Asia/Shanghai'
    cdm_xxl_job_addresses='http://192.168.53.74:8080/xxl-job-admin'
elif [ ${cdmservice} = 'szatc-cdm-service-basic-systemmana' ]
then
    cdm_app_port=30013
	cdm_app_db_url='jdbc:mysql://192.168.53.60:23306/szcdm_syst?useUnicode=true&characterEncoding=utf8&serverTimezone=Asia/Shanghai'
elif [ ${cdmservice} = 'szatc-cdm-service-biz-adapter3rd' ]
then
    cdm_app_port=30016
	cdm_app_db_url='jdbc:mysql://192.168.53.60:23306/szcdm_flight?useUnicode=true&characterEncoding=utf8&serverTimezone=Asia/Shanghai'
    cdm_xxl_job_addresses='http://192.168.53.74:8080/xxl-job-admin'
	cdm_rabbitmq_host='192.168.53.74'
	cdm_rabbitmq_port=5672
	cdm_rabbitmq_username='szdc'
	cdm_rabbitmq_password='szdc123'
elif [ ${cdmservice} = 'szatc-cdm-service-biz-flightcollab' ]
then
    cdm_app_port=30011
	cdm_app_db_url='jdbc:mysql://192.168.53.60:23306/szcdm_flight?useUnicode=true&characterEncoding=utf8&serverTimezone=Asia/Shanghai'
    cdm_xxl_job_addresses='http://192.168.53.74:8080/xxl-job-admin'
	cdm_flight_info_url='http://192.168.53.74:40011/openapi/api/auth/baseinfo/queryFlightInfoBySobt'
	cdm_send_middle_url='http://192.168.53.74:40011/openapi/api/auth/sendMsg'
	cdm_running_situation_url='http://192.168.53.74:40011/'
elif [ ${cdmservice} = 'szatc-cdm-service-biz-inmessage' ]
then
    cdm_app_port=30012
	cdm_app_db_url='jdbc:mysql://192.168.53.60:23306/szcdm_inmessage?useUnicode=true&characterEncoding=utf8&serverTimezone=Asia/Shanghai'
elif [ ${cdmservice} = 'szatc-cdm-service-biz-proc' ]
then
    cdm_app_port=30017
	cdm_app_db_url='jdbc:mysql://192.168.53.60:23306/szcdm_proc?useUnicode=true&characterEncoding=utf8&serverTimezone=Asia/Shanghai'
elif [ ${cdmservice} = 'szatc-cdm-service-support-eureka' ]
then
    cdm_app_port=30000
elif [ ${cdmservice} = 'szatc-cdm-service-support-gateway' ]
then
    cdm_app_port=30015
else
    echo 'None of the cdmservice met, service name:' ${cdmservice}
fi

echo 'cdm_app_port='${cdm_app_port}
echo 'cdm_redis_host='${cdm_redis_host}
echo 'cdm_redis_port='${cdm_redis_port}
echo 'cdm_eureka_url='${cdm_eureka_url}
echo 'cdm_eureka_use_prefer_ip='${cdm_eureka_use_prefer_ip}
echo 'cdm_app_db_username='${cdm_app_db_username}
echo 'cdm_app_db_password='${cdm_app_db_password}
echo 'cdm_app_db_url='${cdm_app_db_url}
echo 'cdm_xxl_job_addresses='${cdm_xxl_job_addresses}
echo 'cdm_rabbitmq_host='${cdm_rabbitmq_host}
echo 'cdm_rabbitmq_port='${cdm_rabbitmq_port}
echo 'cdm_rabbitmq_username='${cdm_rabbitmq_username}
echo 'cdm_rabbitmq_password='${cdm_rabbitmq_password}
echo 'cdm_flight_info_url='${cdm_flight_info_url}
echo 'cdm_send_middle_url='${cdm_send_middle_url}
echo 'cdm_running_situation_url='${cdm_running_situation_url}
	
# ---------------------------------运行脚本--------------------------------------------
APP_OPTIONS='$APP_OPTIONS'
cp `ls target/*jar |grep -v "\-sources" ` app.jar
cat > run.sh <<EOF
#!/bin/bash
java  ${APP_OPTIONS}  -Djava.net.preferIPv4Stack=true -Djava.security.egd=file:/dev/./urandom -jar /app.jar
EOF
# ---------------------------------镜像脚本--------------------------------------------
cat > Dockerfile <<EOF
FROM java:8
ENV  TZ="Asia/Shanghai"
ENV  APP_OPTIONS="-Xms512m -Xmx768m -Xss512k"
ENV  GOOS=linux
ENV  CGO_ENABLED=0
ENV  cdm_redis_host=${cdm_redis_host}
ENV  cdm_redis_port=${cdm_redis_port}
ENV  cdm_eureka_url=${cdm_eureka_url}
ENV  cdm_eureka_use_prefer_ip=${cdm_eureka_use_prefer_ip}
ENV  cdm_app_db_username=${cdm_app_db_username}
ENV  cdm_app_db_password=${cdm_app_db_password}
ENV  cdm_app_db_url=${cdm_app_db_url}
ENV  cdm_xxl_job_addresses=${cdm_xxl_job_addresses}
ENV  cdm_rabbitmq_host=${cdm_rabbitmq_host}
ENV  cdm_rabbitmq_port=${cdm_rabbitmq_port}
ENV  cdm_rabbitmq_username=${cdm_rabbitmq_username}
ENV  cdm_rabbitmq_password=${cdm_rabbitmq_password}
ENV  cdm_flight_info_url=${cdm_flight_info_url}
ENV  cdm_send_middle_url=${cdm_send_middle_url}
ENV  cdm_running_situation_url=${cdm_running_situation_url}
ADD app.jar /app.jar
ADD run.sh  /run.sh
RUN chmod 775 /run.sh
EXPOSE ${cdm_app_port}
ENTRYPOINT ["/run.sh"]
EOF

cat Dockerfile
# ---------------------------------构建镜像--------------------------------------------
dir
docker build -t ${NEXUS_IP}'/'${IMAGE_NAME}':'${IMAGE_TAG} . #&>/dev/null
# ---------------------------------推送镜像--------------------------------------------
# ================推送镜像开始================
docker login --username=${NEXUS_USER_NAME} --password=${NEXUS_PASSWORD} ${NEXUS_IP}
docker push ${NEXUS_IP}'/'${IMAGE_NAME}':'${IMAGE_TAG}
# ================删除本地镜像================
docker rmi ${NEXUS_IP}'/'${IMAGE_NAME}':'${IMAGE_TAG}
# ================本次镜像版本================
echo ${IMAGE_NAME}':'${IMAGE_TAG}
# -------------------------------------------------------------------------------------
```

#### 4.jenkins发布指令

```shell
# -> 镜像名称
IMAGE_NAME=${cdmservice}
# -> 镜像标签
IMAGE_TAG=${cdmtag}
#IMAGE_TAG='latest'
# -> 容器名称
CONTAINER_NAME=${cdmservice}
# -> 仓库配置
NEXUS_IP='192.168.53.62:8082'
NEXUS_USER_NAME='jenkins'
NEXUS_PASSWORD='cdm12345'

if   [ ${IMAGE_TAG} = '' ]
then
    IMAGE_TAG=`date +%Y%m%d%H%M`
fi

# ================删除容器及镜像================
# Stop container, and delete the container.
CONTAINER_ID=`docker ps | grep ${CONTAINER_NAME} | awk '{print $1}'`
if [ -n "$CONTAINER_ID" ]; then
    docker stop $CONTAINER_ID
    docker rm $CONTAINER_ID
else #如果容器启动时失败了，就需要docker ps -a才能找到那个容器
    CONTAINER_ID=`docker ps -a | grep ${CONTAINER_NAME} | awk '{print $1}'`
    if [ -n "$CONTAINER_ID" ]; then  # 如果是第一次在这台机器上拉取运行容器，那么docker ps -a也是找不到这个容器的
        docker rm $CONTAINER_ID
    fi
fi

# Delete image early version.
IMAGE_ID=`docker images | grep ${IMAGE_NAME} | awk '{print $3}'`
if [ -n "${IMAGE_ID}" ];then
    docker rmi ${IMAGE_ID}
fi

# Delete unused volume 
# docker volume rm $(docker volume ls -qf dangling=true)

#================获取新镜像及容器================
docker login --username=${NEXUS_USER_NAME} --password=${NEXUS_PASSWORD} ${NEXUS_IP}
docker pull ${NEXUS_IP}'/'${IMAGE_NAME}':'${IMAGE_TAG}

docker run -itd --name ${CONTAINER_NAME} --network host ${NEXUS_IP}'/'${IMAGE_NAME}':'${IMAGE_TAG}
```

## 三、部署至K8s方式

#### 1.application，和方式二相同

#### 2.maven，和方式二相同

#### 3.jenkins打包指令

动态变量无需在脚本里指定，直接到k8s里指定。

```shell
# -------------------------------------------------------------------------------------
# -> 进入项目(结合新的项目规则编写)
echo '================进入工作路径================'
cd ${WORKSPACE}/szatc-cdm/${project}
# -------------------------------------------------------------------------------------
# -> 镜像标签(DEV 固定是latest)
#IMAGE_TAG=`date +%Y%m%d%H%M`
IMAGE_TAG=latest
# -> 镜像名称
IMAGE_NAME=${project}
# -> 仓库配置
NEXUS_IP='192.168.53.62:8082'
NEXUS_USER_NAME='jenkins'
NEXUS_PASSWORD='cdm12345'
# -------------------------------------------------------------------------------------
cp `ls target/*jar |grep -v "\-sources" ` app.jar
APP_OPTIONS='$APP_OPTIONS'
cat > run.sh <<EOF
#!/bin/bash
java  $APP_OPTIONS  -Djava.net.preferIPv4Stack=true -Djava.security.egd=file:/dev/./urandom -jar /app.jar
EOF
# -------------------------------------------------------------------------------------
cat > Dockerfile <<EOF
FROM 192.168.53.62:8082/jdk-maven:v20200515
ENV  TZ="Asia/Shanghai"
ENV  APP_OPTIONS="-Xms512m -Xmx768m -Xss512k"
ENV  GOOS=linux
ENV  CGO_ENABLED=0
ADD app.jar /app.jar
ADD run.sh  /run.sh
RUN chmod 775 /run.sh
EXPOSE 8080
ENTRYPOINT ["/run.sh"]
EOF
# -------------------------------------------------------------------------------------
dir
docker build -t ${NEXUS_IP}'/'${IMAGE_NAME}':'${IMAGE_TAG} . #&>/dev/null
# -------------------------------------------------------------------------------------
echo '================推送镜像开始================'
docker login --username=${NEXUS_USER_NAME} --password=${NEXUS_PASSWORD} ${NEXUS_IP}
docker push ${NEXUS_IP}'/'${IMAGE_NAME}':'${IMAGE_TAG}
echo '================删除本地镜像================'
docker rmi ${NEXUS_IP}'/'${IMAGE_NAME}':'${IMAGE_TAG}
echo '================本次镜像版本================'
echo ${IMAGE_NAME}':'${IMAGE_TAG}
# -------------------------------------------------------------------------------------
```

#### 4.jenkins发布指令

直接用k8s脚本发布，很简单

```shell
# -------------------------------------------------------------------------------------
NEXUS_IP='192.168.53.62:8082'
K8S_NS='cdm-dev-ns'
IMAGE_TAG='latest'
# -------------------------------------------------------------------------------------
kubectl set image  -n ${K8S_NS} deployment/${project}   ${project}=${NEXUS_IP}/${project}:${IMAGE_TAG}
echo kubectl set image  -n ${K8S_NS} deployment/${project}   ${project}=${NEXUS_IP}/${project}:${IMAGE_TAG}
kubectl  -n ${K8S_NS}  get pods
echo kubectl  -n ${K8S_NS}  get pods
kubectl  -n ${K8S_NS}  get pod  | grep ${project} | awk '{print $1}' | xargs kubectl delete pod -n ${K8S_NS} 
kubectl  -n ${K8S-NS}  get pods
```

## 四、前端多环境处理

前端的思路是，一个镜像可以匹配到多个环境。根据自己的ip，判断当前的环境，然后取到对应当前环境的网关地址。

打包则是直接在工程里打好dist，然后把copy到Nginx目录下即可。如果是K8s方式，则是copy到nginx镜像来制作新镜像。

#### 1.前端网关地址处理

```js
if (window.location.host.indexOf('53.64:80') !== -1) {
    baseURL='http://192.xxx.xx.xx:30015';
}
......
```

#### 2.jenkins打包指令

k8s方式：

```shell
# -------------------------------------------------------------------------------------
# -> 进入项目(结合新的项目规则编写)
echo '================进入工作路径================'
cd ${WORKSPACE}
# -------------------------------------------------------------------------------------
# -> 镜像标签(DEV 固定是latest)
#IMAGE_TAG=`date +%Y%m%d%H%M`
IMAGE_TAG=latest
# -> 镜像名称
IMAGE_NAME=${project}
# -> 仓库配置
NEXUS_IP='192.168.53.62:8082'
NEXUS_USER_NAME='jenkins'
NEXUS_PASSWORD='cdm12345'
# -------------------------------------------------------------------------------------
cat > Dockerfile <<EOF
FROM 192.168.53.62:8082/nginx:latest
COPY ./dist /usr/share/nginx/html/
EOF
# -------------------------------------------------------------------------------------
docker build -t ${NEXUS_IP}'/'${IMAGE_NAME}':'${IMAGE_TAG} . #&>/dev/null
# -------------------------------------------------------------------------------------
echo '================推送镜像开始================'
docker login --username=${NEXUS_USER_NAME} --password=${NEXUS_PASSWORD} ${NEXUS_IP}
docker push ${NEXUS_IP}'/'${IMAGE_NAME}':'${IMAGE_TAG}
echo '================删除本地镜像================'
docker rmi ${NEXUS_IP}'/'${IMAGE_NAME}':'${IMAGE_TAG}
echo '================本次镜像版本================'
echo ${IMAGE_NAME}':'${IMAGE_TAG}
```

#### 3.jenkins发布指令

k8s方式：

```shell
# -------------------------------------------------------------------------------------
NEXUS_IP='192.168.53.62:8082'
K8S_NS='cdm-dev-ns'
IMAGE_TAG='latest'
# -------------------------------------------------------------------------------------
kubectl set image  -n ${K8S_NS} deployment/${project}   ${project}=${NEXUS_IP}/${project}:${IMAGE_TAG}
echo kubectl set image  -n ${K8S_NS} deployment/${project}   ${project}=${NEXUS_IP}/${project}:${IMAGE_TAG}
kubectl  -n ${K8S_NS}  get pods
echo kubectl  -n ${K8S_NS}  get pods
kubectl  -n ${K8S_NS}  get pod  | grep ${project} | awk '{print $1}' | xargs kubectl delete pod -n ${K8S_NS} 
kubectl  -n ${K8S-NS}  get pods
```

