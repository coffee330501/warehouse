

### 第一步 创建Maven项目

自己install生成jar扔到其他项目中测试没问题后修改pom.xml准备上传Maven中央仓库。

> pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
	
    <!-- groupId十分重要，一般个人github项目就写io.github.用户名 (写自己的) -->
    <groupId>io.github.coffee330501</groupId>
    <artifactId>internal-call</artifactId>
    <!-- 版本号，后面加-SNAPSHOT的话会发布到SNAPSHOT仓库，在STAGING仓库中找不到 -->
    <version>1.0</version>

    <properties>
        <maven.compiler.source>8</maven.compiler.source>
        <maven.compiler.target>8</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>
	
    <!-- 项目地址(写自己的) -->
    <url>https://github.com/coffee330501/internal-call</url>

    <!-- 项目在github或其它托管平台的地址(写自己的) -->
    <scm>
        <url>https://github.com/coffee330501/internal-call</url>
        <connection>https://github.com/coffee330501/internal-call.git</connection>
    </scm>

    <!-- 开发者信息(写自己的) -->
    <developers>
        <developer>
            <id>objcfeng</id>
            <name>objcfeng</name>
            <email>objcfeng@qq.com</email>
        </developer>
    </developers>

    <!-- 开源协议 -->
    <licenses>
        <license>
            <name>Apache License, Version 2.0</name>
            <url>http://www.apache.org/licenses/LICENSE-2.0.txt</url>
            <distribution>repo</distribution>
            <comments>A business-friendly OSS license</comments>
        </license>
    </licenses>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-source-plugin</artifactId>
                <version>2.2.1</version>
                <executions>
                    <execution>
                        <id>attach-sources</id>
                        <goals>
                            <goal>jar-no-fork</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-javadoc-plugin</artifactId>
                <version>2.9.1</version>
                <configuration>
                    <additionalJOptions>
                        <additionalJOption>-Xdoclint:none</additionalJOption>
                    </additionalJOptions>
                </configuration>
                <executions>
                    <execution>
                        <id>attach-javadocs</id>
                        <goals>
                            <goal>jar</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-gpg-plugin</artifactId>
                <version>1.5</version>
                <executions>
                    <execution>
                        <id>sign-artifacts</id>
                        <phase>verify</phase>
                        <goals>
                            <goal>sign</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>

    <!-- 重要,定义SNAPSHOT仓库和STAGING仓库 -->
    <distributionManagement>
        <snapshotRepository>
            <id>ossrh</id>
            <url>https://s01.oss.sonatype.org/content/repositories/snapshots</url>
        </snapshotRepository>
        <repository>
            <id>ossrh</id>
            <url>https://s01.oss.sonatype.org/service/local/staging/deploy/maven2/</url>
        </repository>
    </distributionManagement>
</project>
```



### 第二步 打开issues.sonatype.org

1. 注册账号

2. 新建一个项目

   ![image-20230713172919112](C:\Users\lx\AppData\Roaming\Typora\typora-user-images\image-20230713172919112.png)

3. 注意：下图中Project URL以.git结尾，SCM URL没有.git

   ![image-20230713173126881](C:\Users\lx\AppData\Roaming\Typora\typora-user-images\image-20230713173126881.png)

4. 记录地址，在github中创建一个以该地址为名的仓库

   ![image-20230713173426882](C:\Users\lx\AppData\Roaming\Typora\typora-user-images\image-20230713173426882.png)

![image-20230713173518587](C:\Users\lx\AppData\Roaming\Typora\typora-user-images\image-20230713173518587.png)

为什么要建这个仓库呢？原话是这样说的：

> 1. Create a temporary, public repository called https://github.com/coffee330501/OSSRH-93225 to verify github account ownership.

### 第三步 创建密钥

1. 下载安装Gpg4Win 地址：[Gpg4win - Secure email and file encryption with GnuPG for Windows](https://www.gpg4win.org/)

2. 创建密钥

   ![image-20230713174017130](C:\Users\lx\AppData\Roaming\Typora\typora-user-images\image-20230713174017130.png)![image-20230713174137962](C:\Users\lx\AppData\Roaming\Typora\typora-user-images\image-20230713174137962.png)

   ![image-20230713174306646](C:\Users\lx\AppData\Roaming\Typora\typora-user-images\image-20230713174306646.png)

   **这个密码一定要记住！！！**

   

3. 上传密钥

   ![image-20230713174755508](C:\Users\lx\AppData\Roaming\Typora\typora-user-images\image-20230713174755508.png)



### 第四步 Maven Config配置

在setting.xml中添加如下配置

放在servers里面

```xml
<server>
      	<id>ossrh</id>
      	<username>你的issues.sonatype.org账号</username>
      	<password>你的issues.sonatype.org密码</password>
</server>
```

放在profiles里面

```xml
<profile>
      <id>ossrh</id>
      <activation>
        <activeByDefault>true</activeByDefault>
      </activation>
      <properties>
        <!--这里填你安装的GnuPG位置-->
        <gpg.executable>H:\GnuPG\bin\gpg.exe</gpg.executable>
        <gpg.passphrase>刚才你生成密钥时输入的密码，不是指纹！！！</gpg.passphrase>
        <!--这里填你秘钥在磁盘上的位置,可通过上面步骤的 gpg --list-keys找到-->
        <gpg.homedir>C:\Users\30398\AppData\Roaming\gnupg</gpg.homedir>
  </properties>
```



### 第五步 mvn clean deploy

1. ![image-20230713175027135](C:\Users\lx\AppData\Roaming\Typora\typora-user-images\image-20230713175027135.png)

   执行时会让你输密码，输入创建密钥时的密码即可。

   ![image-20230713175135148](C:\Users\lx\AppData\Roaming\Typora\typora-user-images\image-20230713175135148.png)

   最后输出这样就可以了



### 第六步 发布到中央仓库

登录 https://s01.oss.sonatype.org/#stagingRepositories

> 注意：mvn BUILD SUCCESS之后到这里显示出来似乎还要一段时间，很可能进去之后是空的，可以在这个页面等一会。

Release之后会自动Drop

![image-20230713175302214](C:\Users\lx\AppData\Roaming\Typora\typora-user-images\image-20230713175302214.png)



### 最后一步

等mvn同步之后，就可以在maven.org上搜索到你的项目了。

![image-20230713180025800](C:\Users\lx\AppData\Roaming\Typora\typora-user-images\image-20230713180025800.png)

