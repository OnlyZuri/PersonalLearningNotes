# Docker的使用——基于特定语言

## 基于JAVA语言的应用程序使用Docker

## 1）准备工作

* 准备代码文件，下载依赖

```bash
# 测试项目git地址
https://github.com/spring-projects/spring-petclinic.git
# 进入spring-petclinic目录，下载依赖并启动
./mvnw spring-boot:run
# 拉取OpenJDK镜像
docker pull openjdk:8-jdk-alpine
```

* 创建Dockerfile文件

```bash
# syntax=docker/dockerfile:1
FROM openjdk:8-jdk-alpine
# 表示后面的命令都是在/home/app下执行的
WORKDIR /home/app
COPY .mvn/ .mvn
COPY mvnw pom.xml ./
RUN ./mvnw dependency:go-offline
COPY src ./src
CMD ["./mvnw", "spring-boot:run"]
```

* 创建镜像

```bash
docker build -t java-docker .
# 可修改镜像名
docker tag java-docker:latest java-docker:v1.0.0
```

* 启动容器

```bash
# --name 定义容器名
docker run -dp 8080:8080 --name spring-server java-docker
# -a可以查看历史，包括没在运行的容器
docker ps -a
# 关闭容器可以用id，也可以用容器名，同样也可用于重启
docker stop/restart <container_id>/<container_name>
```

## 2）设置本地开发环境

* 使用mysql镜像，并为此分配卷、网络，Java应用程序连接此容器上的mysql

```bash
docker volume create mysql_data
docker volume create mysql_config
docker network create mysqlnet
# 启动mysql容器
docker run -it -d -v mysql_data:/var/lib.mysql -v mysql_config:/etc/mysql/conf.d --network mysqlnet --name mysqlserver -e MYSQL_USER=petclinic -e MYSQL_PASSWORD=petclinic -e MYSQL_ROOT_PASSOWRD=root -e MYSQL_DATABASE=petclinic -p 3306:3306 mysql:8.0.23
# 修改下Dokcerfile使本测试项目使用MYSQL
CMD ["./mvnw", "spring-boot:run", "-Dspring-boot.run.profiles=mysql"]
# 重建Java程序镜像
docker build -t java-docker .
docker run -dp 8080:8080 --name springboot-server --network mysqlnet -e MYSQL_URL=jdbc:mysql://mysqlserver/petclinic java-docker
```

* 使用Docker-compose进行本地开发

```yaml
# docker-compose.dev.yml
version: 3.8
services:
	petclinic:
		build:
			context: .
		ports:
			- 8000:8000
			- 8080:8080
		environment:
			- SERVER_PORT=8080
			- MYSQL_URL=jdbc:mysql://mysqlserver/petclinic
		volumes:
			- ./:/app
		command: ./mvnw spring-boot:run -Dspring-boot.run.profiles=mysql 
	
	mysqlserver:
		image: mysql:8.0.23
		ports:
			- 3306:3306
		environment:
			- MYSQL_ROOT_PASSWORD=root
			- MYSQL_USER=petclinic
			- MYSQL_PASSWORD=petclinic
			- MYSQL_DATABASE=petclinic
		volumes:
			- mysql_data: /var/lib/mysql
			- mysql_config: /etc/mysql/conf.d
volumes:
	mysql_data
	mysql_config
```

```bash
docker-compose -f docker-compose.dev.yml up --build
```

