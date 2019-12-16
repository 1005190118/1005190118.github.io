---
title：Spring Boot打包成docker镜像
---



# pom加入依赖

``` xml
<properties>
	<docker.image.prefix>xdclass</docker.image.prefix>
</properties>

<build>
	<finalName>docker-demo</finalName>
	 <plugins>
		<plugin>
			<groupId>com.spotify</groupId>
			<artifactId>dockerfile-maven-plugin</artifactId>
			<version>1.3.6</version>
			<configuration>
			<repository>${docker.image.prefix}/${project.artifactId}</repository>
			       <buildArgs>
			              <JAR_FILE>target/${project.build.finalName}.jar</JAR_FILE>
			        </buildArgs>
			  </configuration>
		</plugin>
	 </plugins>
</build>
```

# 根目录新建Dockerfile

```dockerfile
FROM openjdk:8-jdk-alpine
VOLUME /tmp
ARG JAR_FILE
COPY ${JAR_FILE} app.jar
ENTRYPOINT ["java","-jar","/app.jar"]
```

# 执行打包命令（再有docker的机器上）

> mvn install dockerfile:build

# 推送阿里云镜像仓库

> 阿里云镜像仓库：https://dev.aliyun.com/search.html

