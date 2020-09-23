# 入行 10 年总结，作为开发必须知道的 Maven 实用技巧

![](http://img.uprogrammer.cn/static/20200921144458.png)

## Maven 介绍

### 什么是 Maven

Maven 是基于项目对象模型(POM project object model)，可以通过一小段描述信息（配置）来管理项目的构建，报告和文档的软件项目管理工具，简单的说就是用来管理项目所需要的依赖且管理项目构建的工具。

### Maven 的安装与配置

1. 从 [Maven 官网](http://maven.apache.org/download.cgi)下载压缩包，解压到本地。
2. 配置环境变量 `MAVEN_HOME` 为 Maven 的解压目录。
3. 添加 Maven 目录下的 bin 目录到环境变量 `PATH` 中。
4. 可以在 Maven 目录下的 conf/setting.xml 文件中，通过 `<localRepository />` 来指定本地仓库路径。
5. 打开终端，输入 `mvn -version` 验证时是否成功。

### Idea 中配置本地安装的 Maven

![](http://img.uprogrammer.cn/static/20200909144745.png)

打开 Idea 的配置面板，找到 Maven 配置页。

1. `Maven home directory`：设置为本地的 Maven 路径
2. `User settings file`：勾选后面的 Override 可以自定义 settings 文件，可以指向 Maven 路径下的 `conf/settings.xml`
3. `Local repository`：勾选 Override 同样可以自定义仓库路径，默认会从配置的 settings 文件中读取。

### Maven 坐标与依赖

#### 坐标

Maven 通过 groupId、artifactId、version 三个变量来唯一确定一个具体的依赖，俗称 GAV。

#### 依赖

在 pom.xml 中我们通过 dependency 来声明坐标信息(GAV)，如我们需要声明对 4.2.6.RELEASE 版本 spring-core 包的依赖。

```xml
<dependency>
  <groupId>org.springframework</groupId>
  <artifactId>spring-core</artifactId>
  <version>4.2.6.RELEASE</version>
</dependency>
```

#### 依赖 scope

1. compile：编译依赖范围，在编译，测试，运行时都需要，依赖范围默认值
2. test：测试依赖范围，测试时需要。编译和运行不需要，如 junit
3. provided：已提供依赖范围，编译和测试时需要。运行时不需要，如 servlet-api
4. runtime：运行时依赖范围，测试和运行时需要。编译不需要，例如面向接口编程，JDBC 驱动实现 jar
5. system：系统依赖范围。本地依赖，不在 Maven 中央仓库，结合 systemPath 标签使用

####  依赖排除

使用 `<exclusions>` 标签下的 `<exclusion>` 标签指定 GA 信息来排除，例如：排除 xxx.jar 传递依赖过来的 yyy.jar

```xml
<dependency>
  <groupId>com.xxx</groupId>
  <artifactId>xxx</artifactId>
  <version>x.version</version>
  <exclusions>
    <exclusion>
      <groupId>com.xxx</groupId>
      <artifactId>yyy</artifactId>
    </exclusion>
  </exclusions>
</dependency>
```

#### 依赖关系查看

进入工程根目录，在命令行中运行

1. `mvn dependency:tree` 命令会列出依赖关系树及各级依赖关系
2. `mvn dependency:analyze` 分析依赖关系

### Maven 项目的结构

![](http://img.uprogrammer.cn/static/20200909145713.png)

Maven 项目有其他标准的目录组织结构，如上图所示：

```
|- project: 项目目录
  |- src: 源码目录
    |- src/main/java: 项目 java 源码文件存放目录
    |- src/main/resources: 项目资源文件存放目录
    |- src/test/java: 项目单元测试源码文件存放目录
    |- src/test/resources: 项目单元测试资源文件存放目录
  |- target: 项目编译/打包后存放的目标路径
  |- pom.xml: 项目依赖管理配置文件
```

### Maven 的生命周期

![](http://img.uprogrammer.cn/static/20200909192047.png)

Maven 有 3 套生命周期，每套生命周期都包含了一些阶段，这些阶段是有序的，后面的阶段依赖前面的阶段。这三套生命周期是相互独立的，可以仅仅调用 clean 生命周期的某个阶段， 或者调用 default 生命周期的某个阶段，而不会对其他生命周期产生任何影响。

1. clean：清理项目
2. default：构建项目
   + compile：编译项目的源代码
   + package：打包编译好的代码
   + install：将包安装至本地仓库，提供给其他项目依赖
   + deploy：将最终的包复制到远程的仓库，提供给其他开发人员和项目共享。
3. site：建立项目站点

生命周期的执行：

```shell
# 清理项目
mvn clean
# 打包项目
mvn package
# 打包并安装到本地仓库
mvn install
```

可以组合各阶段进行执行：

```shell
# 清理项目后打包并发布到远程仓库
mvn clean package deploy
```

## settings 文件详解

### settings 文件的作用

settings 是用来设置 Maven 参数的配置文件，并且，settings.xml 是 Maven 的全局配置文件。settings.xml 中包含类似本地仓库、远程仓库和联网使用的代理信息等配置。

### settings 文件的位置

全局配置：`${MAVEN_HOME}/conf/settings.xml`

用户配置：`${user.home}/.m2/settings.xml`

### settings 文件配置优先级

其实相对于多用户的 PC 机而言，在 Maven 安装目录的 conf 子目录下面的 settings.xml 才是真正的全局的配置。而用户目录的 .m2 子目录下面的 settings.xml 的配置只是针对当前用户的。当这两个文件同时存在的时候，那么对于相同的配置信息用户目录下面的 settings.xml 中定义的会覆盖 Maven 安装目录下面的 settings.xml 中的定义。用户目录下的 settings.xml 文件一般是不存在的，但是 Maven 允许我们在这里定义我们自己的 settings.xml，如果需要在这里定义我们自己的 settings.xml 的时候就可以把 Maven 安装目录下面的 settings.xml 文件拷贝到用户目录的 .m2 目录下，然后改成自己想要的样子。

### settings.xml 元素

#### 顶级元素概览

```xml
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0
                          https://maven.apache.org/xsd/settings-1.0.0.xsd">
  <localRepository/>
  <interactiveMode/>
  <usePluginRegistry/>
  <offline/>
  <pluginGroups/>
  <servers/>
  <mirrors/>
  <proxies/>
  <profiles/>
  <activeProfiles/>
</settings>
```

#### LocalRepository

该值表示构建系统本地仓库的路径。其默认值：`~/.m2/repository`。

#### Servers

一般，仓库的下载和部署是在 pom.xml 文件中的 repositories 和 distributionManagement 元素中定义的。然而，一般类似用户名、密码（有些仓库访问是需要安全认证的）等信息不应该在 pom.xml 文件中配置，这些信息可以配置在 settings.xml 中。

```xml
<!--配置服务端的一些设置。一些设置如安全证书不应该和pom.xml一起分发。这种类型的信息应该存在于构建服务器上的settings.xml文件中。 -->
<servers>
  <!--服务器元素包含配置服务器时需要的信息 -->
  <server>
    <!--这是server的id（注意不是用户登陆的id），该id与distributionManagement中repository元素的id相匹配。 -->
    <id>server001</id>
    <!--鉴权用户名。鉴权用户名和鉴权密码表示服务器认证所需要的登录名和密码。 -->
    <username>my_login</username>
    <!--鉴权密码 。鉴权用户名和鉴权密码表示服务器认证所需要的登录名和密码。密码加密功能已被添加到2.1.0 +。详情请访问密码加密页面 -->
    <password>my_password</password>
    <!--鉴权时使用的私钥位置。和前两个元素类似，私钥位置和私钥密码指定了一个私钥的路径（默认是${user.home}/.ssh/id_dsa）以及如果需要的话，一个密语。将来passphrase和password元素可能会被提取到外部，但目前它们必须在settings.xml文件以纯文本的形式声明。 -->
    <privateKey>${usr.home}/.ssh/id_dsa</privateKey>
    <!--鉴权时使用的私钥密码。 -->
    <passphrase>some_passphrase</passphrase>
    <!--文件被创建时的权限。如果在部署的时候会创建一个仓库文件或者目录，这时候就可以使用权限（permission）。这两个元素合法的值是一个三位数字，其对应了unix文件系统的权限，如664，或者775。 -->
    <filePermissions>664</filePermissions>
    <!--目录被创建时的权限。 -->
    <directoryPermissions>775</directoryPermissions>
  </server>
</servers>
```

#### Mirrors

用于定义一系列的远程仓库的镜像。对于一个 Maven 项目，如果没有特别声明，默认使用 Maven 的 central 库，url 为 http://repo.maven.apache.org/maven2/。但是这些远程库往往需要连接互联网访问，由于访问互联网的限制或安全控制的需要，在企业内部往往需要建立对远程库的镜像，即远程库的 mirror。

> **注意**：
>
> 1. 定义多个远程仓库镜像时，只有当前一个 mirror 无法连接的时候，才会去找后一个，类似于备份和容灾。
> 2. mirror 也不是按 settings.xml 中写的那样的顺序来查询的。所谓的第一个并不一定是最上面的那个。当有 id 为 B、A、C 的顺序的 mirror 在 mirrors 节点中，Maven 会根据字母排序来指定第一个，所以不管怎么排列，一定会找到 A 这个mirror来进行查找，当A无法连接，出现意外的情况下，才会去B查询。

##### Mirror

当 Maven 需要到的依赖 jar 包不在本地仓库时，就需要到远程仓库下载。这个时候如果 settings.xml 中配置了镜像，而且镜像配置的规则中匹配到目标仓库时，Maven 认为目标仓库被镜像了，不会再去被镜像仓库下载依赖 jar包，而是直接去镜像仓库下载。简单而言，mirror 可以拦截对远程仓库的请求，改变对目标仓库的下载地址。

```xml
<mirrors>
  <!-- 给定仓库的下载镜像。 -->
  <mirror>
    <!-- 该镜像的唯一标识符。id 用来区分不同的 mirror元素。 -->
    <id>mirrorId</id>
    <!-- 镜像名称 -->
    <name>PlanetMirror Australia</name>
    <!-- 该镜像的URL。构建系统会优先考虑使用该URL，而非使用默认的服务器URL。 -->
    <url>http://downloads.planetmirror.com/pub/maven2</url>
    <!-- 被镜像的服务器的id。例如，如果我们要设置了一个 Maven 中央仓库（http://repo.maven.apache.org/maven2/）的镜像，就需要将该元素设置成 central。这必须和中央仓库的id central完全一致。 -->
    <mirrorOf>repositoryId</mirrorOf>
  </mirror>
</mirrors>
```

##### 加速远程依赖的下载

使用镜像可以解决远程依赖下载慢的问题。

```xml
<mirrors>
  <!--国内阿里云提供的镜像，非常不错-->
  <mirror>
    <!--This sends everything else to /public -->
    <id>aliyun_nexus</id>
    <mirrorOf>central</mirrorOf> 
    <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
  </mirror>
</mirrors>
```

在 settings.xml 中配置如上 mirrors，远程的依赖就会从阿里云镜像中下载。

##### 高级镜像配置

为了满足一些复杂的需求，Maven 还支持更高级的镜像配置：

1. `<mirrorOf>*</mirrorOf>`：匹配所有远程仓库。
2. `<mirrorOf>external:*</mirrorOf>`：匹配所有远程仓库，使用 localhost 的除外，使用 `file://` 协议的除外。也就是说，匹配所有不在本机上的远程仓库。
3. `<mirrorOf>repo1,repo2</mirrorOf>`：匹配仓库 repo1 和 repo2，使用逗号分隔多个远程仓库。
4. `<mirrorOf>*,!repo1</miiroOf>`：匹配所有远程仓库，repo1 除外，使用感叹号将仓库从匹配中排除。

##### 案例

个人的 Maven 配置了阿里的镜像，而项目中需要使用到一些第三方 jar 包，为了方便引入，已上传到192.168.0.201 的 nexus 私服下。但由于个人 Maven 阿里的镜像配置为 `<mirrorOf>*</mirrorOf>`，所有的仓库都被镜像，不会再去 192.168.0.201 下下载第三方 jar 包。

上传的第三方 jar 包目标路径：
http://192.168.0.201:8081/nexus/content/groups/public/com/alipay/sdk-java/20170615110434/sdk-java-20170615110434.pom
被镜像后路径：
http://maven.aliyun.com/nexus/content/groups/public/com/alipay/sdk-java/20170615110434/sdk-java-20170615110434.pom

所以需要修改镜像的 mirrorOf 规则，避免默认从镜像中下载。

Maven的 conf/settings.xml

```xml
<mirrors>
  <!--国内阿里云提供的镜像，非常不错-->
  <mirror>
    <!--This sends everything else to /public -->
    <id>aliyun_nexus</id>
    <!--对所有仓库使用该镜像,除了一个名为maven_nexus_201的仓库除外-->
    <!--这个名为maven_nexus_201的仓库可以在javamaven项目中配置一个repository-->
    <mirrorOf>*,!maven_nexus_201</mirrorOf> 
    <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
  </mirror>
</mirrors>
```

Maven 项目下的 pom.xml 配置一个远程仓库

```xml
<repositories>
  <!-- 192.168.0.201远程仓库 -->
  <repository>
    <id>maven_nexus_201</id>
    <name>maven_nexus_201</name>
    <layout>default</layout>
    <url>http://192.168.0.201:8081/nexus/content/groups/public/</url>
    <snapshots>  
      <enabled>true</enabled>  
    </snapshots>
  </repository>
</repositories>
```

## pom 文件详解

#### 顶级元素概览

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" 
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd ">
  <parent />
  <!-- 声明项目描述符遵循哪一个 POM 模型版本。模型本身的版本很少改变，虽然如此，但它仍然是必不可少的，
         这是为了当 Maven 引入了新的特性或者其他模型变更的时候，确保稳定性。 -->
  <modelVersion />
  <groupId />
  <artifactId />
  <version />
  <packaging />
  <name />
  <url />
  <description />
  <prerequisites />
  <issueManagement />
  <ciManagement />
  <inceptionYear />
  <mailingLists />
  <developers />
  <contributors />
  <licenses />
  <scm />
  <organization />
  <build />
  <profiles />
  <modules />
  <repositories />
  <pluginRepositories />
  <dependencies />
  <reports />
  <reporting />
  <dependencyManagement />
  <distributionManagement />
  <properties />
</project>  
```

#### parent

```xml
<!-- 父项目的坐标。如果项目中没有规定某个元素的值，那么父项目中的对应值即为项目的默认值。
         坐标包括group ID，artifact ID和 version。 --> 
<parent> 
  <!-- 被继承的父项目的构件标识符 --> 
  <artifactId>xxx</artifactId>

  <!-- 被继承的父项目的全球唯一标识符 -->
  <groupId>xxx</groupId> 

  <!-- 被继承的父项目的版本 --> 
  <version>xxx</version>

  <!-- 父项目的pom.xml文件的相对路径。相对路径允许你选择一个不同的路径。默认值是../pom.xml。
             Maven首先在构建当前项目的地方寻找父项目的pom，其次在文件系统的这个位置（relativePath位置），
             然后在本地仓库，最后在远程仓库寻找父项目的pom。 --> 
  <relativePath>xxx</relativePath> 
</parent>
```

#### groudId、artifactId、version、packaging

```xml
<!-- 项目的全球唯一标识符，通常使用全限定的包名区分该项目和其他项目。并且构建时生成的路径也是由此生成， 
         如com.mycompany.app生成的相对路径为：/com/mycompany/app --> 
<groupId>xxx</groupId> 

<!-- 构件的标识符，它和group ID一起唯一标识一个构件。换句话说，你不能有两个不同的项目拥有同样的artifact ID
和groupID；在某个特定的group ID下，artifact ID也必须是唯一的。构件是项目产生的或使用的一个东西，Maven
为项目产生的构件包括：JARs，源码，二进制发布和WARs等。 --> 
<artifactId>xxx</artifactId> 

<!-- 项目当前版本，格式为:主版本.次版本.增量版本-限定版本号 --> 
<version>1.0-SNAPSHOT</version>

<!-- 项目产生的构件类型，例如jar、war、ear、pom。插件可以创建他们自己的构件类型，所以前面列的不是全部构件类型 --> 
<packaging>jar</packaging> 
```

`<packaging>` 默认构件类型为 jar，当作为父级 Maven 项目的时候，构件类型为 pom。

#### modules

```xml
<!-- 模块（有时称作子项目） 被构建成项目的一部分。列出的每个模块元素是指向该模块的目录的相对路径 --> 
<modules>
  <!--子项目相对路径-->
  <module></module>
</modules>
```

通过 `<modules>` Maven 项目之间可以形成父子关系，可以有多个层级。

#### repositories

```xml
<!-- 发现依赖和扩展的远程仓库列表。 --> 
<repositories> 
  <!-- 包含需要连接到远程仓库的信息 --> 
  <repository> 
    <!-- 如何处理远程仓库里发布版本的下载 --> 
    <releases> 
      <!-- true 或者 false 表示该仓库是否为下载某种类型构件（发布版，快照版）开启。 --> 
      <enabled><enabled> 

      <!-- 该元素指定更新发生的频率。Maven 会比较本地 POM 和远程 POM 的时间戳。
             这里的选项是：always（一直），daily（默认，每日），interval：X（这里X是以分钟为单位的时间间隔），或者never（从不）。 --> 
      <updatePolicy></updatePolicy> 

        <!-- 当Maven验证构件校验文件失败时该怎么做：ignore（忽略），fail（失败），或者warn（警告）。 --> 
      <checksumPolicy></checksumPolicy> 
    </releases> 

    <!-- 如何处理远程仓库里快照版本的下载。有了 releases 和 snapshots 这两组配置，POM 就可以在每个单独的仓库中，
         为每种类型的构件采取不同的策略。例如，可能有人会决定只为开发目的开启对快照版本下载的支持。 --> 
    <snapshots> 
      <enabled><enabled>
      <updatePolicy></updatePolicy>
      <checksumPolicy></checksumPolicy> 
    </snapshots> 

    <!-- 远程仓库唯一标识符。可以用来匹配在 settings.xml 文件里配置的远程仓库 --> 
    <id>banseon-repository-proxy</id> 

    <!-- 远程仓库名称 --> 
    <name>banseon-repository-proxy</name> 

    <!-- 远程仓库 URL，按 protocol://hostname/path 形式 --> 
    <url>http://192.168.1.169:9999/repository/</url> 

    <!-- 用于定位和排序构件的仓库布局类型-可以是 default（默认）或者legacy（遗留）。Maven 2为其仓库提供了一个默认
         的布局；然而，Maven 1.x 有一种不同的布局。我们可以使用该元素指定布局是 default（默认）还是 legacy（遗留）。 --> 
    <layout> default </layout> 
  </repository> 
</repositories>
```

`<id>` 可以配合 settings.xml 中的远程仓库配置进行使用，见上一节中的**案例**。

#### properties

```xml
<!-- 以值替代名称，Properties 可以在整个 POM 中使用，也可以作为触发条件。格式是<name>value</name>。 --> 
<properties>
  <name>value</name>
</properties>
```

#### dependencies

```xml
<!-- 该元素描述了项目相关的所有依赖。 这些依赖组成了项目构建过程中的一个个环节。它们自动从项目定义的仓库中下载。 --> 
<dependencies> 
  <dependency> 
    <!-- 依赖的group ID --> 
    <groupId>org.apache.maven</groupId> 

    <!-- 依赖的artifact ID --> 
    <artifactId>maven-artifact</artifactId> 

    <!-- 依赖的版本号。 在 Maven 2 里，也可以配置成版本号的范围。 --> 
    <version>3.8.1</version> 

    <!-- 依赖类型，默认类型是jar。它通常表示依赖的文件的扩展名，但也有例外。一个类型可以被映射成另外一个扩展
                 名或分类器。类型经常和使用的打包方式对应，尽管这也有例外。一些类型的例子：jar，war，ejb-client和test-jar。
                 如果设置extensions为 true，就可以在plugin里定义新的类型。所以前面的类型的例子不完整。 --> 
    <type>jar</type> 

    <!-- 依赖的分类器。分类器可以区分属于同一个POM，但不同构建方式的构件。分类器名被附加到文件名的版本号后面。例如，
                 如果你想要构建两个单独的构件成JAR，一个使用Java 1.4编译器，另一个使用Java 6编译器，你就可以使用分类器来生
                 成两个单独的JAR构件。 --> 
    <classifier></classifier> 

    <!-- 依赖范围。在项目发布过程中，帮助决定哪些构件被包括进来。欲知详情请参考依赖机制。 
                - compile ：默认范围，用于编译 
                - provided：类似于编译，但支持你期待jdk或者容器提供，类似于classpath 
                - runtime: 在执行时需要使用 
                - test: 用于test任务时使用 
                - system: 需要外在提供相应的元素。通过systemPath来取得 
                - systemPath: 仅用于范围为system。提供相应的路径 
                - optional: 当项目自身被依赖时，标注依赖是否传递。用于连续依赖时使用 --> 
    <scope>test</scope> 

    <!-- 仅供system范围使用。注意，不鼓励使用这个元素，并且在新的版本中该元素可能被覆盖掉。该元素为依赖规定了文件
                 系统上的路径。需要绝对路径而不是相对路径。推荐使用属性匹配绝对路径，例如${java.home}。 --> 
    <systemPath></systemPath> 

    <!-- 当计算传递依赖时，从依赖构件列表里，列出被排除的依赖构件集。即告诉 Maven 你只依赖指定的项目，不依赖项目的
                 依赖。此元素主要用于解决版本冲突问题 --> 
    <exclusions> 
      <exclusion> 
        <artifactId>spring-core</artifactId> 
        <groupId>org.springframework</groupId> 
      </exclusion> 
    </exclusions> 

    <!-- 可选依赖，如果你在项目 B 中把 C 依赖声明为可选，你就需要在依赖于 B 的项目（例如项目 A）中显式的引用对 C 的依赖。
                 可选依赖阻断依赖的传递性。 --> 
    <optional>true</optional> 
  </dependency> 
</dependencies>
```

#### dependencyManagement

1. 只能出现在父 pom 里
2. 用于统一版本号
3. 只是依赖声明，并不直接依赖，需要时在子项目中在声明要使用依赖的 GA 信息，V 信息可以省略。

```xml
<!-- 继承自该项目的所有子项目的默认依赖信息。这部分的依赖信息不会被立即解析，而是当子项目声明一个依赖
     （必须描述 groupId 和 artifactId 信息），如果 group ID 和 artifact ID 以外的一些信息没有
     描述，则通过 group ID 和 artifact ID 匹配到这里的依赖，并使用这里的依赖信息。 --> 
<dependencyManagement> 
  <dependencies> 
    <!-- 参见dependencies/dependency元素 --> 
    <dependency> 
    </dependency> 
  </dependencies> 
</dependencyManagement>
```

#### build

```xml
<build> 
  <!-- 该元素设置了项目源码目录，当构建项目的时候，构建系统会编译目录里的源码。
       该路径是相对于 pom.xml 的相对路径。 --> 
  <sourceDirectory></sourceDirectory> 

  <!-- 该元素设置了项目单元测试使用的源码目录，当测试项目的时候，构建系统会编译目录里的源码。
       该路径是相对于 pom.xml 的相对路径。 --> 
  <testSourceDirectory></testSourceDirectory> 

  <!-- 被编译过的应用程序 class 文件存放的目录。 --> 
  <outputDirectory></outputDirectory> 

  <!-- 被编译过的测试 class 文件存放的目录。 --> 
  <testOutputDirectory></testOutputDirectory> 

  <!-- 这个元素描述了项目相关的所有资源路径列表，例如和项目相关的属性文件，
       这些资源被包含在最终的打包文件里。 --> 
  <resources> 
    <!-- 这个元素描述了项目相关或测试相关的所有资源路径 --> 
    <resource> 

      <!-- 是否使用参数值代替参数名。参数值取自 properties 元素或者文件里配置的属性，文件在 filters 元素里列出。 --> 
      <filtering></filtering>

      <!-- 描述存放资源的目录，该路径相对 pom.xml 路径 --> 
      <directory></directory>

      <!-- 包含的模式列表，例如**/*.xml. --> 
      <includes>
        <include></include>
      </includes>

      <!-- 排除的模式列表，例如**/*.xml -->
      <excludes>
        <exclude></exclude>
      </excludes>
    </resource> 
  </resources> 

  <!-- 子项目可以引用的默认插件信息。该插件配置项直到被引用时才会被解析或绑定到生命周期。
       给定插件的任何本地配置都会覆盖这里的配置 --> 
  <pluginManagement> 
    <!-- 参见 dependencies 元素 -->
    <plugins>  
    </plugins> 
  </pluginManagement> 

  <!-- 该项目使用的插件列表 。 --> 
  <plugins> 
    <!-- plugin 元素包含描述插件所需要的信息。 --> 
    <plugin> 
      <!-- 插件在仓库里的 group ID --> 
      <groupId></groupId> 

      <!-- 插件在仓库里的 artifact ID --> 
      <artifactId></artifactId> 

      <!-- 被使用的插件的版本（或版本范围） --> 
      <version></version> 

      <!-- 在构建生命周期中执行一组目标的配置。每个目标可能有不同的配置。 --> 
      <executions> 
        <!-- execution元素包含了插件执行需要的信息 --> 
        <execution> 
          <!-- 执行目标的标识符，用于标识构建过程中的目标，或者匹配继承过程中需要合并的执行目标 --> 
          <id></id>

          <!-- 绑定了目标的构建生命周期阶段，如果省略，目标会被绑定到源数据里配置的默认阶段 --> 
          <phase></phase>

          <!-- 配置的执行目标 --> 
          <goals></goals> 

          <!-- 作为DOM对象的配置 --> 
          <configuration></configuration>
        </execution> 
      </executions> 

      <!-- 项目引入插件所需要的额外依赖 --> 
      <dependencies>
        <!-- 参见dependencies/dependency元素 --> 
        <dependency> 
        </dependency> 
      </dependencies> 

      <!-- 作为 DOM 对象的配置 --> 
      <configuration></configuration> 
    </plugin> 
  </plugins>
</build> 
```

## 使用 Nexus 搭建 Maven 私服

Nexus 是一个强大的 Maven 仓库管理器，它极大的简化了本地内部仓库的维护和外部仓库的访问。

### Docker 搭建 Nexus

我们采用 Docker 方式来安装：

1\. 拉取 nexus3 镜像

```shell
# 拉取 nexus3 镜像
docker pull sonatype/nexus3:3.16.0
```

2\. 启动 nexus3 容器

```shell
# 通过镜像启动容器
docker run -d --name nexus -p 8081:8081 -v /Users/linfuyan/Develop/nexus-data:/nexus-data sonatype/nexus3:3.16.0
```

> -d：后台模式运行容器
>
> --name：容器命名为 nexus
>
> -p：映射本地 8081 端口到容器内的 8081 端口
>
> -v：将容器内的 /nexus-data 目录挂载到本地目录

3\. 浏览器中访问 localhost:8081 就可以看到 Nexus 页面了。

![](http://img.uprogrammer.cn/static/20200910093211.png)

> **注意**：
>
> 3.6.0 版本的 sonatype/nexus3 无法查看到 upload 页面。
>
> 3.27.0 版本的 sonatype/nexus3 在我电脑上跑不起来，而且初始密码存在 admin.password 文件中。
>
> 最终选择了 3.16.0 版本。

### Nexus 基础使用

通过右上角的 sign in 按钮，输入默认账号密码 `admin/admin123` 可以对仓库等进行管理。

#### 创建远程仓库

![](http://img.uprogrammer.cn/static/20200910093817.png)

可选 Maven2 group、hosted、proxy 类型。

hosted：本地仓库，通常我们会部署自己的构件到这一类型的仓库。比如公司的第二方库。

proxy：代理仓库，它们被用来代理远程的公共仓库，如 Maven 中央仓库。

group：仓库组，用来合并多个hosted/proxy仓库，当你的项目希望在多个 repository 使用资源时就不需要多次引用了，只需要引用一个 group 即可。

![](http://img.uprogrammer.cn/static/20200910094301.png)

新建 hosted 类型仓库

![](http://img.uprogrammer.cn/static/20200910094820.png)

把 craft4j 添加到 maven-public 中。

![](http://img.uprogrammer.cn/static/20200910100425.png)

#### 创建具有上传权限的角色

![](http://img.uprogrammer.cn/static/20200910115006.png)

#### 创建具有上传权限的角色的用户

![](http://img.uprogrammer.cn/static/20200910115142.png)

### 上传 jar 组件到 Nexus

#### 通过网页手动上传第三方 jar 到 Nexus

![](http://img.uprogrammer.cn/static/20200910142651.png)

![](http://img.uprogrammer.cn/static/20200910142822.png)

上传成功以后，就可以在对应的远程仓库中查看到上传完成的组件信息。

#### 通过 mvn deploy:deploy-file 上传到 Nexus

```shell
➜  ron-jwt git:(master) ✗ mvn deploy:deploy-file -DgroupId=io.craft4j -DartifactId=checkstyle -Dversion=1.0.1-SNAPSHOT -Dpackaging=jar -Dfile=/Users/linfuyan/Code/java-lab/airplan-java-server/codestyle/checkstyle-7.0-all.jar -DrepositoryId=craft4j -Durl=http://127.0.0.1:8081/repository/craft4j/
[INFO] Scanning for projects...
[INFO]
[INFO] ------------------< org.apache.maven:standalone-pom >-------------------
[INFO] Building Maven Stub Project (No POM) 1
[INFO] --------------------------------[ pom ]---------------------------------
[INFO]
[INFO] --- maven-deploy-plugin:2.7:deploy-file (default-cli) @ standalone-pom ---
Downloading from remote-repository: http://127.0.0.1:8081/repository/craft4j/io/craft4j/checkstyle/1.0.1-SNAPSHOT/maven-metadata.xml
Downloaded from remote-repository: http://127.0.0.1:8081/repository/craft4j/io/craft4j/checkstyle/1.0.1-SNAPSHOT/maven-metadata.xml (770 B at 3.7 kB/s)
Uploading to remote-repository: http://127.0.0.1:8081/repository/craft4j/io/craft4j/checkstyle/1.0.1-SNAPSHOT/checkstyle-1.0.1-20200910.081245-2.jar
Uploading to remote-repository: http://127.0.0.1:8081/repository/craft4j/io/craft4j/checkstyle/1.0.1-SNAPSHOT/checkstyle-1.0.1-20200910.081245-2.pom
```

repositoryId 依赖 settings.xml 中的 server 配置，主要是用户名与密码。

#### 通过 mvn deploy 方式上传到 Nexus

在上面的步骤中，已经将新建的 craft4j 仓库添加到 maven-public 仓库组中。 

在 settings.xml 中修改配置：

```xml
<servers>
  <server>
    <id>nexus-releases</id>
    <username>dev</username>
    <password>dev123</password>
  </server>
  <server>
    <id>craft4j</id>
    <username>dev</username>
    <password>dev123</password>
  </server>
</servers>
```

在项目 pom.xml 中修改配置：

```xml
<distributionManagement>
  <repository>
    <id>maven-releases</id>
    <name>Nexus Release Repository</name>
    <url>http://127.0.0.1:8081/repository/maven-releases/</url>
  </repository>
  <snapshotRepository>
    <id>craft4j</id>
    <name>Nexus Snapshot Repository</name>
    <url>http://127.0.0.1:8081/repository/craft4j/</url>
  </snapshotRepository>
</distributionManagement>
```

pom.xml 中的 repository id 与 settings.xml 中的 server id 相对应，需要保持一致。

在需要发布的项目中执行 `mvn deploy`，看到如下日志，就说明发布成功了，同样可以在 Nexus 的仓库浏览页中查看。

```shell
➜  ron-jwt git:(master) ✗ mvn clean deploy
[INFO] Scanning for projects...
[INFO] 
[INFO] ---------------------------< io.ron:ron-jwt >---------------------------
[INFO] Building ron-jwt 1.0-SNAPSHOT
[INFO] --------------------------------[ jar ]---------------------------------
[INFO] ...
[INFO] --- maven-deploy-plugin:2.7:deploy (default-deploy) @ ron-jwt ---
Downloading from craft4j: http://127.0.0.1:8081/repository/craft4j/io/ron/ron-jwt/1.0-SNAPSHOT/maven-metadata.xml
Uploading to craft4j: http://127.0.0.1:8081/repository/craft4j/io/ron/ron-jwt/1.0-SNAPSHOT/ron-jwt-1.0-20200910.071640-1.jar
Uploaded to craft4j: http://127.0.0.1:8081/repository/craft4j/io/ron/ron-jwt/1.0-SNAPSHOT/ron-jwt-1.0-20200910.071640-1.jar (15 kB at 21 kB/s)
Uploading to craft4j: http://127.0.0.1:8081/repository/craft4j/io/ron/ron-jwt/1.0-SNAPSHOT/ron-jwt-1.0-20200910.071640-1.pom
Uploaded to craft4j: http://127.0.0.1:8081/repository/craft4j/io/ron/ron-jwt/1.0-SNAPSHOT/ron-jwt-1.0-20200910.071640-1.pom (1.3 kB at 2.6 kB/s)
Downloading from craft4j: http://127.0.0.1:8081/repository/craft4j/io/ron/ron-jwt/maven-metadata.xml
Uploading to craft4j: http://127.0.0.1:8081/repository/craft4j/io/ron/ron-jwt/1.0-SNAPSHOT/maven-metadata.xml
Uploaded to craft4j: http://127.0.0.1:8081/repository/craft4j/io/ron/ron-jwt/1.0-SNAPSHOT/maven-metadata.xml (757 B at 2.0 kB/s)
Uploading to craft4j: http://127.0.0.1:8081/repository/craft4j/io/ron/ron-jwt/maven-metadata.xml
Uploaded to craft4j: http://127.0.0.1:8081/repository/craft4j/io/ron/ron-jwt/maven-metadata.xml (271 B at 1.0 kB/s)
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 5.243 s
[INFO] Finished at: 2020-09-10T15:16:42+08:00
[INFO] ------------------------------------------------------------------------
```

## Maven 构件的版本管理

使用 Maven 作为依赖管理工具，一般我们对于依赖的版本号，常见两种类型：一种以“-RELEASE”结尾，另一种以“-SNAPSHOT”结尾。

私服中，会存在 snapshot 快照仓库和 release 发布仓库，snapshot 快照仓库用于保存开发过程中的不稳定版本，release 正式仓库则是用来保存稳定的发行版本。

Maven 会根据模块的版本号（pom 文件中的 version）中是否带有“-SNAPSHOT”（注意这里必须是全部大写）来判断是快照版本还是正式版本。如果是快照版本，那么在 mvn deploy 时会自动发布到私服的快照版本库中；如果是正式发布版本，那么在 mvn deploy 时会自动发布到正式版本库中。

快照版本的依赖，Maven 编译打包的时候无论本地是否存在，都会去私服拉取最新的，而正式版本的依赖，如果本地仓库已经存在，Maven 不会去私服拉取最新的版本，所以我们要基于快照版本进行开发，但是上线的时候一定记得变成正式版，否则如果本地正在进行开发另一个功能，提交到私服的代码有可能会被误上线。

那我们有什么好的方法来避免这种情况呢？

1\. 在 settings.xml 中修改私服配置，通过 updatePolicy 为 always 强制更新。

```xml
<profile>
    <id>nexus</id>
    <repositories>
        <repository>
            <id>nexus</id>
            <name>Nexus</name>
            <url>http://127.0.0.1:8081/repository/groups/maven-public/</url>
            <releases>
                <enabled>true</enabled>
                <updatePolicy>always</updatePolicy>
            </releases>
            <snapshots>
                <enabled>true</enabled>
                <updatePolicy>always</updatePolicy>
            </snapshots>
        </repository>
    </repositories>
    <pluginRepositories>
        <pluginRepository>
        </pluginRepository>
    </pluginRepositories>
</profile>
```

2\. 在构建的时候加上“-U”参数，强制拉取所有依赖的最新代码

```shell
mvn clean install -U
```

3\. 语义化版本

首先，我们在团队协作时，要定义好开发中的依赖一定不要忘记升级版本号，然后开发的过程中还要保持版本号以“-SNAPSHOT”结尾，来把该依赖作为快照版本进行开发，这样每次别人更新完上传到私服以后，你本地打包时会自动拉取最新代码，从而方便我们的开发和维护。

**参考与致谢**

[带你深度解析Maven](https://www.cnblogs.com/hafiz/p/8119964.html)

[史上最全的maven的pom.xml文件详解](https://www.cnblogs.com/hafiz/p/5360195.html)

[setting.xml 配置详解](https://www.cnblogs.com/yangxia-test/p/4409736.html)

[上传jar包到nexus私服](https://blog.csdn.net/u013409283/article/details/78854875)

[Maven私服:Docker安装nexus3](https://www.jianshu.com/p/09a6cab3785a)

[Maven版本号中隐藏的惊天大秘密](https://www.cnblogs.com/hafiz/p/8124741.html)

[Maven全局配置文件settings.xml详解](https://www.cnblogs.com/soupk/p/9303611.html)

---

如果你看完本文有收获，欢迎关注微信公众号：精进Java(ID: **craft4j**)，更多 Java 后端与架构的干货等你一起学习与交流。

