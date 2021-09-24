#Description: tomcat image build by sunyongtao
FROM centos:latest
MAINTAINER "sunyongtao@netfinworks.com <18217754133>"
COPY jdk1.8.0_60  /usr/java/jdk1.8.0_60/
ADD tomcat  /opt/app/tomcat/ 
ENV CATALINA_HOME=/opt/app/tomcat \ 
    JAVA_HOME=/usr/java/jdk1.8.0_60 \ 
#    JAVA_MEM="-Xmx1024m -Xms1024m -XX:NewSize=256m -XX:MaxNewSize=512m" \
    TZ=Asia/Shanghai \ 
    LANG=en_US.UTF-8
#RUN yum -y install  less lrzsz zip  unzip && yum clean all
EXPOSE 8080
#以CMD方式启动docker容器的时候,如果要想动态传入JVM参数,需要-e JAVA_MEM动态传入,如下:
#docker run -d -P --name tomcat1  -e JAVA_MEM="-Xmx1024m -Xms1024m -XX:NewSize=128m -XX:MaxNewSize=256"  -v  /mnt/tomcat/webapps/cmf:/opt/app/tomcat/webapps/cmf -v /opt/pay/config/basis:/opt/pay/config/basis  tomcat:v0.1
CMD ["/opt/app/tomcat/bin/catalina.sh","run"]


#ENTRYPOINT ["/opt/app/tomcat/bin/catalina.sh","run"]
#可以在启动tomcat容器时,在命令的最后面动态补加JVM启动参数，如下:
#docker run -d -P --name tomcat1    -v  /mnt/tomcat/webapps/cmf:/opt/app/tomcat/webapps/cmf -v /opt/pay/config/basis:/opt/pay/config/basis tomcat:v0.2  -Xmx1024m -Xms1024m -XX:NewSize=128m -XX:MaxNewSize=256

#################################ENTRYPOINT与CMD的区别##############################
#a、 类似CMD指令功能,用于为容器指定默认运行程序,从而使容器像是一个单独的可执行程序
#b、与CMD不同的是,由ENTRYPOINT启动的程序不会被docker run命令行指定参数所覆盖,而且这些命令行参数会被当作参数传递给ENTRYPOINT指定的程序,不过
#docker run命令的--entrypoint选项的参数可覆盖ENTRYPOINT指令指定的程序

#Syntax
#  ENTRYPOINT <command>
#  ENTRYPOINT ["executable","param1","param2"]

#docker run命令行传入的参数会覆盖CMD指令的内容,并且附加到ENTRYPOINT命令最后作为其参数使用

#Dockerfile文件中可以使用多个ENTRYPOINT指令,但是仅有最后一个会生效
#Dockerfile文件中可以使用多个CMD指令,但是仅有最后一个会生效
#Dockerfile文件中同时定义了CMD和ENTRYPOINT,那么CMD的所有内容将被作为参数传递给ENTRYPOINT

#############################HEALTHCHECK#####################################################
HEALTHCHECK  --interval=120s --timeout=30s  --start-period=20s  --retries=3  \
   CMD curl -I - -q http://localhost:8080/cmf/_health_check



############################Docker容器的重启策略如下
#no，默认策略，在容器退出时不重启容器
#on-failure，在容器非正常退出时（退出状态非0），才会重启容器

#    on-failure:3，在容器非正常退出时重启容器，最多重启3次

#always，在容器退出时总是重启容器
#unless-stopped，在容器退出时总是重启容器，但是不考虑在Docker守护进程启动时就已经停止了的容器
[root@node5 tomcat]# 


%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%多阶段构建%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%

FROM maven:3.8.1-jdk-11 as builder
WORKDIR /usr/src/star-user
COPY . .
RUN  mvn clean package -P 'prd,!dev'  #指定引用哪个环境的配置文件


# 不能使用slim这个jdk，需要生成验证码的时候缺少必要的linux依赖
FROM adoptopenjdk/openjdk11:latest as prod
WORKDIR /root
COPY --from=builder  /usr/src/star-user/target/*.jar /root/app.jar    #多阶段构建，将第一阶段生成的文件拷贝到第二阶段镜像中
ENTRYPOINT  ["java","-server","-Xms512m","-Xmx1024m","-Dcom.sun.management.jmxremote", "-Dfile.encoding=UTF-8", "-Xrunjdwp:server=y,transport=dt_socket,address=8000,suspend=n","-jar","/root/app.jar"]

