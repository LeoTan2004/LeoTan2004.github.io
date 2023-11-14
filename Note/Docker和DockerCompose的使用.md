# Docker和DockerCompose的使用

## Docker

**Docker**需要写**Dockerfile**文件，登录接口的**Dockerfile**示例如下

```dockerfile
FROM openjdk:19
ENV TZ=Asia/Shanghai
COPY *.jar /app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar" ,"-Dspring.profiles.active=dev", "/app.jar"]
```

> 这里有几个需要注意的点
>
> - **Dockerfile**的RUN命令会在构建镜像时执行，而不是启动容器时执行，因此应用的启动命令用`ENTRYPOINT`
> - **Dockerfile**中建议指定运行环境，使用参数`-Dspring.profiles.active=dev`，其中`dev`表示的是配置文件的后缀，本例子中就是是*application-dev.yaml*生效。
> - 建议本地环境和运行环境的配置文件分离，在运行时通过制定环境是不同的配置生效。这样可以一次构建，多环境部署。减少成本。

## Docker-Compose

**Docker-Compose**需要编写**compose.yaml**文件来构建，还时上面的例子

```yaml
services:
  logindb:								# 服务名称
    image: 'mysql:latest'				 # 镜像名称
    environment:						# 环境变量
      - 'MYSQL_ROOT_PASSWORD=root'		  # MYSQL的密码 	
  loginservice:							# web服务名称
    build: ./login						# 构建的路径
    ports:
      - "8080:8080"						# 对外暴露的端口

```

> 这里的几个注意点：
>
> - 在同一个compose构建的各个服务可以通过服务名来互相访问，比如这里的数据库服务名称是***logindb***，那么我们在***loginservice***端就可以通过`jdbc:mysql://logindb:3306?serverTimezone=UTC`来访问数据库。
> - 数据库的端口如果不想对外暴露，就不需要写出来端口映射，**Docker-Compose**为我们在网络内部做了映射，这样子就可以让数据库只能通过内部访问。这不仅仅可用于保护数据库安全，也可以用来做后期的网关代理
> - `build`参数指定需要构建的路径，如果没有`build`，表示使用远程的或者已经构建好的。`build`的路径下的必须要有***Dockerfile***文件，否则无法构建。
> - `ports`参数表示该服务和宿主机的端口映射。
> - 更多配置文件的参数可见官网或者[Docker Compose | 菜鸟教程 (runoob.com)](https://www.runoob.com/docker/docker-compose.html)

编写好compose文件以后，我们就可以在**同一级目录下**使用命令docker-compose命令了（前提是安装docker-compose，教程网上有）

- `docker-compose up -d`：后台构建并且启动compose中的各个服务
- `docker-compose down`：停止并且销毁当前目录下**compose**文件中的所有服务。并且删除网络（实际开发中谨慎使用）。
- `docker-compose stop`：停止当前容器，不销毁
- `docker-compose start`：启动停止了的容器

---

参考

[Docker-Compose常用命令 - 春光牛牛 - 博客园 (cnblogs.com)](https://www.cnblogs.com/yakniu/p/16968792.html)

[Docker Compose | 菜鸟教程 (runoob.com)](https://www.runoob.com/docker/docker-compose.html)

