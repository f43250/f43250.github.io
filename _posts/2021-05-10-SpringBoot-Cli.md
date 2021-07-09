---
layout: post
title: "SpringBoot CLI"
subtitle: 'Some tips on SpringBoot CLI'
author: "FengJiaWen"
header-style: text
tags:
  - SpringBoot
  - Cli
  - 命令行
  - Jwt
---

Update: SpringBoot CLI

---

**SpringBoot Cli**

<p>最近项目需要使用Jar包及私钥直接生成Jwt,不通过接口向外提供,第一次使用CLI生成,记录一下过程</p>

* 方法一:

1.pom.xml build更改打包插件及类加载器为支持动态加载主类的类加载器

```xml
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-jar-plugin</artifactId>
                <configuration>
                    <archive>
                        <manifest>
                            <mainClass>org.springframework.boot.loader.PropertiesLauncher</mainClass>
                        </manifest>
                    </archive>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <mainClass>${default-class}</mainClass>
                    <layout>ZIP</layout>
                </configuration>
                <executions>
                    <execution>
                        <goals>
                            <goal>repackage</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
```

2.pom.xml增加上述mainClass为变量的配置,此步非常重要,如果不通过变量指定默认值,则后续不生成Jwt直接启动项目也需额外指定主类,让原本简洁的启动方式变得繁琐.
```xml
<properties>
    <default-class>xx.xxx.xxxApplication</default-class>
 </properties>
```

3.编写Cli主方法

```java
import cn.yottacloud.core.security.JwtManager;

/**
 * cli生成jwt
 * @author FLE
 * @date 2021-07-01 20:00
 */
public class JwtCli {

    /**
     * cli生成jwt方法,读取java命令第一个参数为私钥
     */
    public static void main(String[] args) throws Exception {
        JwtManager jwtManager = new JwtManager();
        String jwt = jwtManager.generateToken(args[0]);
        System.out.println(jwt);
    }
}
```

4.mvn package打好jar包;处于jar包同级目录下执行以下命令,即可在终端输出jwt,是否引用第三方jar包或jar包是否处于运行状态都不影响Cli执行结果.

```shell
java -jar -Dloader.main=xx.xxx.JwtCli xxxx.jar "私钥字符串"
```

* 方法二:
若不想更改打包插件及主类,可用以下方式,不过不推荐.
解压包含Cli主方法且打好的jar包,进入到BOOT-INFO/classes目录,执行以下命令即可在终端输出jwt
```shell
java -Djava.ext.dirs=../lib xx.xxx.JwtCli "私钥字符串"
```
说明:-Djava.ext.dirs=../lib 为依赖主方法相对路径下lib下的所有jar依赖.

* 建议使用方法一:不仅用于生成Jwt一个场景,所有需要Cli的场景以及jar是否启动都可以终端执行.特别现在基本都在容器中部署,解压代码包脚本部署方式较少,方法二需要解压jar包,有局限性.
