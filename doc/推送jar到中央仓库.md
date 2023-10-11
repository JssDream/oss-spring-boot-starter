# 推送jar到maven仓库

> https://blog.csdn.net/qq_23501739/article/details/131462588
>
> https://www.codenong.com/cs109097048/
>
> https://blog.csdn.net/wdj_yyds/article/details/132106515
>
> https://xijia.blog.csdn.net/article/details/120728657

## Sonatype JIRA 账号

登录或注册 issues.sonatype.org 社区账号：[注册地址](https://link.juejin.cn/?target=https%3A%2F%2Fissues.sonatype.org%2Fsecure%2FSignup!default.jspa)

密码要求如下：

密码必须至少有 12 个字符。
密码必须至少包含 1 个大写字符。
密码必须至少包含 1 个特殊字符，例如 &、%、™ 或 É。
密码必须包含至少 3 种不同的字符，例如大写字母、小写字母、数字和标点符号。
密码不得与用户名或电子邮件地址相似。

### 创建问题
点击仪表盘面板右上角 ”新建“ 按钮，按照以下步骤向 Sonotype 提交新建项目工单：

项目：Community Support - Open Source Project Repository Hosting (OSSRH)
问题类型：New Project
概要：Github 项目名
Group Id：io.github.[Github 用户名] 或 个人域名（逆序填写）
Project URL: 项目地址
SCM URL: 版本控制地址

> 项目与问题类型使用默认选项即可
>
> 填写完毕后，点击新建，等待审核，管理员会让你在GitHub上建立个项目用于验证身份。
>
> 当Issue的Status变为RESOLVED后，就可以进行下一步操作了。

### 验证 Group Id 所有权
> Group Id 使用 Github 账号方式需按照提示在 Github 上创建临时项目；
>
> Group Id 使用个人域名方式需按照提示解析 DNS TXT 记录

![OPEN](https://img-blog.csdnimg.cn/img_convert/1a27655a0939a41414e924ed6fba21bc.webp?x-oss-process=image/format,png)

![RESOLVED](https://img-blog.csdnimg.cn/img_convert/01030727288c4c9602d420bbcf593c21.webp?x-oss-process=image/format,png)

## 安装gpg

GPG（GNU Privacy Guard） 是基于 OpenPGP 标准实现的加密软件，它提供了对文件的非对称加密和签名验证功能。所有发布到 Maven 仓库的文件都需要进行 GPG 签名，以验证文件的合法性。

[Windows](https://link.juejin.cn/?target=https%3A%2F%2Fwww.gpg4win.org%2F)

在安装完成gpg后，在命令行下通过指令来生成一个秘钥：
```
# 密钥生成命令
gpg --gen-key
# 密钥查看命令 
gpg --list-keys
```

在生成的过程中，首先会要求输入姓名和邮箱地址，在命令行窗口下填完这两个信息后，还会弹窗要求输入一个密码：

Real name: xiaoming
Email address: xiaoming@gmail.com

密码： 1234567890

---------
pub   ed25519 2023-10-11 [SC] [expires: 2026-10-10]
      xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
uid           [ultimate] xiaoming <xiaoming@gmail.com>
sub   cv25519 2023-10-11 [E] [expires: 2026-10-10]


c、上传秘钥

在秘钥生成完后，我们需要把公钥上传到公共服务器供sonatype验证，可以通过下面的命令将公钥上传：
gpg --keyserver hkp://keyserver.ubuntu.com:11371 --send-keys XXXXX
gpg --keyserver hkp://pool.sks-keyservers.net --send-keys 2F19A699

虽然我这里一次就上传成功了，但是在看其他教程的过程中，也可能会出现失败的情况，这种情况可以尝试上传到其他的存放公钥的服务器：
● pool.sks-keyservers.net
● keys.openpgp.org
● pgp.mit.edu

gpg --keyserver keyserver.ubuntu.com --send-keys xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
gpg: sending key xxxxxxxxxxxxx to hkp://keyserver.ubuntu.com


gpg --keyserver keyserver.ubuntu.com --recv-keys xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
gpg: key xxxxxxxxxxxxx: "xiaoming <xiaoming@gmail.com>" not changed
gpg: Total number processed: 1
gpg:              unchanged: 1

## maven 设置

接下来需要修改本地maven的配置，为了保险起见，我建议大家最好同时修改.m2和conf目录下的配置文件，否则有可能出现一些奇怪的问题。

账号密码和上面注册的ossrh一致

a、server

首先在配置文件中添加一个server节点，配置sonatype的用户名及密码：

```
<servers>
    <server>
        <id>ossrh</id>
        <username>${sonatype username}</username>
        <password>${sonatype password}</password>
    </server>
</servers>
```

b、profile

接着添加一个profie节点，配置gpg信息，这里就需要填入在生成gpg秘钥过程中，我们在弹窗中输入的密码了：	

```
<profiles>
    <profile>
        <id>ossrh</id>
        <properties>
            <gpg.executable>gpg</gpg.executable>
            <gpg.passphrase>${弹窗输入的那个密码}</gpg.passphrase>
        </properties>
    </profile>
</profiles>
<activeProfiles>
    <activeProfile>ossrh</activeProfile>
</activeProfiles>
```

## 项目pom修改

distributionManagement
添加distributionManagement信息，声明要打包到sonatype的maven仓库中去。

```
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
```

c、plugins

这里需要添加各种plugin插件，除了常用的maven-compiler和maven-deploy插件外，还需要下面几个关键插件：
● nexus-staging-maven-plugin：sonatype插件，用来将项目发布到中央仓库使用
● maven-source-plugin：生成java source.jar文件
● maven-javadoc-plugin：生成java doc文档
● maven-gpg-plugin：对文件进行自动签名

使用到的全部插件详细配置如下，直接拷到项目中就可以使用：

```
<profiles>
        <profile>
            <id>release</id>
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
        </profile>
    </profiles>

```

d、开源签名证书

添加license信息，使用Apache Licene 2.0 协议就行。

```
<licenses>
    <license>
        <name>The Apache Software License, Version 2.0</name>
        <url>http://www.apache.org/licenses/LICENSE-2.0.txt</url>
        <distribution>repo</distribution>
    </license>
</licenses>
```

e、仓库信息

在这里填写一下项目的地址，把我们的github仓库地址贴上去就可以了。

```
<scm>
    <url>
        https://github.com/mose-x/query-dsl-plus
    </url>
    <connection>
        scm:git@github.com/mose-x/query-dsl-plus.git
    </connection>
    <developerConnection>
        scm:git@github.com/mose-x/query-dsl-plus.git
    </developerConnection>
</scm>
```

f、开发人员信息

补充开发者的个人信息，虽然估计也没什么人会联系我就是了。

```
<developers>
    <developer>
        <name>mose-x</name>
        <email>mose-x@qq.com</email>
        <organization>https://github.com/mose-x</organization>
        <timezone>+8</timezone>
    </developer>
</developers>
```

g. 补充额外信息

```
<name>query-dsl-plus</name>
<description>a tool about bilayer cache</description>
<url>https://github.com/mose-x/query-dsl-plus</url>

```

h. 打包发布

java程序中执行 clean deploy

```shell
mvn clean deploy -P release
```

#### 查看是否发布成功

1. 方式一：[Sonatype Nexus](https://link.juejin.cn/?target=https%3A%2F%2Fs01.oss.sonatype.org%2F%23welcome) 面板上查看https://s01.oss.sonatype.org

![Sonatype Nexus](https://img-blog.csdnimg.cn/img_convert/eaf1bc71bc0443965b47addb15b359f5.webp?x-oss-process=image/format,png)

一般30分钟后能查看
https://repo1.maven.org/maven2/

一般4小时后能查看
https://search.maven.org/

一般24小时后能查看
https://mvnrepository.com/
