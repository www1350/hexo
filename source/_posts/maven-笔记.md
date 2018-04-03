---
title: maven 笔记
abbrlink: 6badd4fd
date: 2018-04-03 22:39:19
tags:
categories:
---

原文地址： http://www.infoq.com/cn/author/%E8%AE%B8%E6%99%93%E6%96%8C

这里只是摘录点笔记
## 坐标的原则

```
<dependency>
  <groupId>junit</groupId>
  <artifactId>junit</artifactId>
  <version>4.8.2</version>
  <scope>test</scope>
</dependency>
```

滥用坐标、错用坐标的样例比比皆是，在中央仓库中我们能看到SpringFramework有两种坐标，其一是直接使用springframework作为groupId，如springframework:spring-beans:1.2.6，另一种是用org.springframework作为groupId，如org.springframework:spring-beans:2.5。细心看看，前一种方式显得比较随意，后一种方式则是基于域名衍生出来的，显然后者更合理，因为用户能一眼根据域名联想到其Maven坐标，方便寻找。因此新版本的SpringFramework构件都使用org.springframework作为groupId。由这个例子我们可以看到坐标规划一个原则是基于项目域名衍生。其实很多流行的开源项目都破坏了这个原则，例如JUnit，这是因为Maven社区在最开始接受构件并部署到中央仓库的时候，没有很很严格的限制，而对于这些流行的项目来说，一时间更改坐标会影响大量用户，因此也算是个历史遗留问题了。

还有一个常见的问题是将groupId直接匹配到公司或者组织名称，因为乍一看这是显而易见的。例如组织是zoo.com，有个项目是dog，那有些人就直接使用groupId com.zoo了。如果项目只有一个模块，这是没有什么问题的，但现实世界的项目往往会有很多模块，Maven的一大长处就是通过多模块的方式管理项目。那dog项目可能会有很多模块，我们用坐标的哪个部分来定义模块呢？groupId显然不对，version也不可能是，那只有artifactId。因此要这里有了另外一个原则，用artifactId来定义模块，而不是定义项目。接下来，很显然的，项目就必须用groupId来定义。因此对于dog项目来说，应该使用groupId com.zoo.dog，不仅体现出这是zoo.com下的一个项目，而且可以与该组织下的其他项目如com.zoo.cat区分开来。

除此之外，artifactId的定义也有最佳实践，我们常常可以看到一个项目有很多的模块，例如api，dao，service，web等等。Maven项目在默认情况下生成的构件，其名称不会是基于artifactId，version和packaging生成的，例如api-1.0.jar，dao-1.0.jar等等，他们不会带有groupId的信息，这会造成一个问题，例如当我们把所有这些构件放到Web容器下的时候，你会发现项目dog有api-1.0.jar，项目cat也有api-1.0.jar，这就造成了冲突。更坏的情况是，dog项目有api-1.0.jar，cat项目有api-2.0.jar，其实两者没什么关系，可当放在一起的时候，却很容易让人混淆。为了让坐标更加清晰，又出现了一个原则，即在定义artiafctId时也加入项目的信息，例如dog项目的api模块，那就使用artifactId dog-api，其他就是dog-dao，dao-service等等。虽然连字号是不允许出现在Java的包名中的，但Maven没这个限制。现在dog-api-1.0.jar，cat-2.0.jar被放在一起时，就不容易混淆了。

关于坐标，我们还没谈到version，这里不再详述因为读者可以从Maven: The Complete Guide中找到详细的解释，简言之就是使用这样一个格式：

<主版本>.<次版本>.<增量版本>-<限定符>
其中主版本主要表示大型架构变更，次版本主要表示特性的增加，增量版本主要服务于bug修复，而限定符如alpha、beta等等是用来表示里程碑。当然不是每个项目的版本都要用到这些4个部分，根据需要选择性的使用即可。在此基础上Maven还引入了SNAPSHOT的概念，用来表示活动的开发状态，由于不涉及坐标规划，这里不进行详述。不过有点要提醒的是，由于SNAPSHOT的存在，自己显式地在version中使用时间戳字符串其实没有必要。

2.

```
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-compiler-plugin</artifactId>
  <version>2.3.2</version>
  <configuration>
    <source>1.5</source>
    <target>1.5</target>
  </configuration>
</plugin>
```

 mvn dependency:analyze
分析依赖

最后，还一些重要的POM内容通常被大多数项目所忽略，这些内容不会影响项目的构建，但能方便信息的沟通，它们包括项目URL，开发者信息，SCM信息，持续集成服务器信息等等，这些信息对于开源项目来说尤其重要。对于那些想了解项目的人来说，这些信息能他们帮助找到想要的信息，基于这些信息生成的Maven站点也更有价值。相关的POM配置很简单，如：

```
<project>
  <description>...</description>
  <url>...</url>
  <licenses>...</licenses>
  <organization>...</organization>
  <developers>...</developers>
  <issueManagement>...</issueManagement>
  <ciManagement>...</ciManagement>
  <mailingLists>...</mailingLists>
  <scm>...</scm>
</project>
```

```
<dependency>
  <groupId>org.springframework</groupId>
  <artifactid>spring-beans</artifactId>
  <version>2.5</version>
</dependency>
<dependency>
  <groupId>org.springframework</groupId>
  <artifactid>spring-context</artifactId>
  <version>2.5</version>
</dependency>
<dependency>
  <groupId>org.springframework</groupId>
  <artifactid>spring-core</artifactId>
  <version>2.5</version>
</dependency>
```

你会在一个项目中使用不同版本的SpringFramework组件么？答案显然是不会。因此这里就没必要重复写三次<version>2.5</version>，使用Maven属性将2.5提取出来如下：

```
<properties>
  <spring.version>2.5</spring.version>
</properties>
<depencencies>
  <dependency>
    <groupId>org.springframework</groupId>
    <artifactid>spring-beans</artifactId>
    <version>${spring.version}</version>
  </dependency>
  <dependency>
    <groupId>org.springframework</groupId>
    <artifactid>spring-context</artifactId>
    <version>${spring.version}</version>
  </dependency>
  <dependency>
    <groupId>org.springframework</groupId>
    <artifactid>spring-core</artifactId>
    <version>${spring.version}</version>
  </dependency>
</depencencies>
```

现在2.5只出现在一个地方，虽然代码稍微长了点，但重复消失了，日后升级依赖版本的时候，只需要修改一处，而且也能避免漏掉升级某个依赖。

读者可能已经非常熟悉这个例子了，我这里再啰嗦一遍是为了给后面做铺垫，多模块POM重构的目的和该例一样，也是为了消除重复，模块越多，潜在的重复就越多，重构就越有必要。

dependencyManagement只会影响现有依赖的配置，但不会引入依赖。例如我们可以在父模块中配置如下：

```
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>junit</groupId>
      <artifactid>junit</artifactId>
      <version>4.8.2</version>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>log4j</groupId>
      <artifactid>log4j</artifactId>
      <version>1.2.16</version>
    </dependency>
  </dependencies>
</dependencyManagement>
```

这段配置不会给任何子模块引入依赖，但如果某个子模块需要使用JUnit和Log4j的时候，我们就可以简化依赖配置成这样：

```
  <dependency>
    <groupId>junit</groupId>
    <artifactid>junit</artifactId>
  </dependency>
  <dependency>
    <groupId>log4j</groupId>
    <artifactid>log4j</artifactId>
  </dependency>
```

你可以把dependencyManagement放到单独的专门用来管理依赖的POM中，然后在需要使用依赖的模块中通过import scope依赖，就可以引入dependencyManagement。例如可以写这样一个用于依赖管理的POM：

```
<project>
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.juvenxu.sample</groupId>
  <artifactId>sample-dependency-infrastructure</artifactId>
  <packaging>pom</packaging>
  <version>1.0-SNAPSHOT</version>
  <dependencyManagement>
    <dependencies>
        <dependency>
          <groupId>junit</groupId>
          <artifactid>junit</artifactId>
          <version>4.8.2</version>
          <scope>test</scope>
        </dependency>
        <dependency>
          <groupId>log4j</groupId>
          <artifactid>log4j</artifactId>
          <version>1.2.16</version>
        </dependency>
    </dependencies>
  </dependencyManagement>
</project>
```

然后我就可以通过非继承的方式来引入这段依赖管理配置：

```
  <dependencyManagement>
    <dependencies>
        <dependency>
          <groupId>com.juvenxu.sample</groupId>
          <artifactid>sample-dependency-infrastructure</artifactId>
          <version>1.0-SNAPSHOT</version>
          <type>pom</type>
          <scope>import</scope>
        </dependency>
    </dependencies>
  </dependencyManagement>

  <dependency>
    <groupId>junit</groupId>
    <artifactid>junit</artifactId>
  </dependency>
  <dependency>
    <groupId>log4j</groupId>
    <artifactid>log4j</artifactId>
  </dependency>
```

这样，父模块的POM就会非常干净，由专门的packaging为pom的POM来管理依赖，也契合的面向对象设计中的单一职责原则。此外，我们还能够创建多个这样的依赖管理POM，以更细化的方式管理依赖。这种做法与面向对象设计中使用组合而非继承也有点相似的味道。
## 消除多模块插件配置重复

与dependencyManagement类似的，我们也可以使用pluginManagement元素管理插件。一个常见的用法就是我们希望项目所有模块的使用Maven Compiler Plugin的时候，都使用Java 1.5，以及指定Java源文件编码为UTF-8，这时可以在父模块的POM中如下配置pluginManagement：

```
<build>
  <pluginManagement>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>2.3.2</version>
        <configuration>
          <source>1.5</source>
          <target>1.5</target>
          <encoding>UTF-8</encoding>
        </configuration>
      </plugin>
    </plugins>
  </pluginManagement>
</build>
```

这段配置会被应用到所有子模块的maven-compiler-plugin中，由于Maven内置了maven-compiler-plugin与生命周期的绑定，因此子模块就不再需要任何maven-compiler-plugin的配置了。

持续集成？
- 只维护一个源码仓库
- 让构建自行测试
- 每人每天向主干提交代码
- 每次提交都应在持续集成机器上构建主干
- 保持快速的构建
- 在模拟生产环境中测试
- 让每个人都能轻易获得最新的可执行文件
- 每个人都能看到进度
- 自动化部署

现在，我们希望Maven在integration-test阶段执行所有以IT结尾命名的测试类，配置Maven Surefire Plugin如下：

```
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-surefire-plugin</artifactId>
        <version>2.7.2</version>
        <executions>
          <execution>
            <id>run-integration-test</id>
            <phase>integration-test</phase>
            <goals>
              <goal>test</goal>
            </goals>
            <configuration>
              <includes>
                <include>**/*IT.java</include>
              </includes>
            </configuration>
          </execution>
        </executions>
      </plugin>
```

对应于同样的package生命周期阶段，Maven为jar项目调用了maven-jar-plugin，为war项目调用了maven-war-plugin，换言之，packaging直接影响Maven的构建生命周期。了解这一点非常重要，特别是当你需要自定义打包行为的时候，你就必须知道去配置哪个插件。一个常见的例子就是在打包war项目的时候排除某些web资源文件，这时就应该配置maven-war-plugin如下：

```
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-war-plugin</artifactId>
    <version>2.1.1</version>
    <configuration>
      <webResources>
        <resource>
          <directory>src/main/webapp</directory>
          <excludes>
            <exclude>**/*.jpg</exclude>
          </excludes>
        </resource>
      </webResources>
    </configuration>
  </plugin>
```

本专栏的《坐标规划》一文中曾解释过，一个Maven项目只生成一个主构件，当需要生成其他附属构件的时候，就需要用上classifier。源码包和Javadoc包就是附属构件的极佳例子。它们有着广泛的用途，尤其是源码包，当你使用一个第三方依赖的时候，有时候会希望在IDE中直接进入该依赖的源码查看其实现的细节，如果该依赖将源码包发布到了Maven仓库，那么像Eclipse就能通过m2eclipse插件解析下载源码包并关联到你的项目中，十分方便。由于生成源码包是极其常见的需求，因此Maven官方提供了一个插件来帮助用户完成这个任务：

```
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-source-plugin</artifactId>
    <version>2.1.2</version>
    <executions>
      <execution>
        <id>attach-sources</id>
        <phase>verify</phase>
        <goals>
          <goal>jar-no-fork</goal>
        </goals>
      </execution>
    </executions>
  </plugin>
```

类似的，生成Javadoc包只需要配置插件如下：

```
  <plugin>          
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-javadoc-plugin</artifactId>
    <version>2.7</version>
    <executions>
      <execution>
        <id>attach-javadocs</id>
          <goals>
            <goal>jar</goal>
          </goals>
      </execution>
    </executions>
  </plugin>    
```

为了帮助所有Maven用户更方便的使用Maven中央库中海量的资源，中央仓库的维护者强制要求开源项目提交构件的时候同时提供源码包和Javadoc包。这是个很好的实践，读者也可以尝试在自己所处的公司内部实行，以促进不同项目之间的交流。
## 可执行CLI包

除了前面提到了常规JAR包、WAR包，源码包和Javadoc包，另一种常被用到的包是在命令行可直接运行的CLI（Command Line）包。默认Maven生成的JAR包只包含了编译生成的.class文件和项目资源文件，而要得到一个可以直接在命令行通过java命令运行的JAR文件，还要满足两个条件：

JAR包中的/META-INF/MANIFEST.MF元数据文件必须包含Main-Class信息。
项目所有的依赖都必须在Classpath中。
Maven有好几个插件能帮助用户完成上述任务，不过用起来最方便的还是maven-shade-plugin，它可以让用户配置Main-Class的值，然后在打包的时候将值填入/META-INF/MANIFEST.MF文件。关于项目的依赖，它很聪明地将依赖JAR文件全部解压后，再将得到的.class文件连同当前项目的.class文件一起合并到最终的CLI包中，这样，在执行CLI JAR文件的时候，所有需要的类就都在Classpath中了。下面是一个配置样例：

```
  <plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-shade-plugin</artifactId>
    <version>1.4</version>
    <executions>
      <execution>
        <phase>package</phase>
        <goals>
          <goal>shade</goal>
        </goals>
        <configuration>
          <transformers>
            <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
              <mainClass>com.juvenxu.mavenbook.HelloWorldCli</mainClass>
            </transformer>
          </transformers>
        </configuration>
      </execution>
    </executions>
  </plugin>
```

上述例子中的，我的Main-Class是com.juvenxu.mavenbook.HelloWorldCli，构建完成后，对应于一个常规的hello-world-1.0.jar文件，我还得到了一个hello-world-1.0-cli.jar文件。细心的读者可能已经注意到了，这里用的是cli这个classifier。最后，我可以通过java -jar hello-world-1.0-cli.jar命令运行程序。

自定义格式包

实际的软件项目常常会有更复杂的打包需求，例如我们可能需要为客户提供一份产品的分发包，这个包不仅仅包含项目的字节码文件，还得包含依赖以及相关脚本文件以方便客户解压后就能运行，此外分发包还得包含一些必要的文档。这时项目的源码目录结构大致是这样的：

pom.xml
src/main/java/
src/main/resources/
src/test/java/
src/test/resources/
src/main/scripts/
src/main/assembly/
README.txt
除了基本的pom.xml和一般Maven目录之外，这里还有一个src/main/scripts/目录，该目录会包含一些脚本文件如run.sh和run.bat，src/main/assembly/会包含一个assembly.xml，这是打包的描述文件，稍后介绍，最后的README.txt是份简单的文档。

我们希望最终生成一个zip格式的分发包，它包含如下的一个结构：

bin/
lib/
README.txt
其中bin/目录包含了可执行脚本run.sh和run.bat，lib/目录包含了项目JAR包和所有依赖JAR，README.txt就是前面提到的文档。

描述清楚需求后，我们就要搬出Maven最强大的打包插件：maven-assembly-plugin。它支持各种打包文件格式，包括zip、tar.gz、tar.bz2等等，通过一个打包描述文件（该例中是src/main/assembly.xml），它能够帮助用户选择具体打包哪些文件集合、依赖、模块、和甚至本地仓库文件，每个项的具体打包路径用户也能自由控制。如下就是对应上述需求的打包描述文件src/main/assembly.xml：

```
<assembly>
  <id>bin</id>
  <formats>
    <format>zip</format>
  </formats>
  <dependencySets>
    <dependencySet>
      <useProjectArtifact>true</useProjectArtifact>
      <outputDirectory>lib</outputDirectory>
    </dependencySet>
  </dependencySets>
  <fileSets>
    <fileSet>
      <outputDirectory>/</outputDirectory>
      <includes>
        <include>README.txt</include>
      </includes>
    </fileSet>
    <fileSet>
      <directory>src/main/scripts</directory>
      <outputDirectory>/bin</outputDirectory>
      <includes>
        <include>run.sh</include>
        <include>run.bat</include>
      </includes>
    </fileSet>
  </fileSets>
</assembly>
```

首先这个assembly.xml文件的id对应了其最终生成文件的classifier。
其次formats定义打包生成的文件格式，这里是zip。因此结合id我们会得到一个名为hello-world-1.0-bin.zip的文件。（假设artifactId为hello-world，version为1.0）
dependencySets用来定义选择依赖并定义最终打包到什么目录，这里我们声明的一个depenencySet默认包含所有所有依赖，而useProjectArtifact表示将项目本身生成的构件也包含在内，最终打包至输出包内的lib路径下（由outputDirectory指定）。
fileSets允许用户通过文件或目录的粒度来控制打包。这里的第一个fileSet打包README.txt文件至包的根目录下，第二个fileSet则将src/main/scripts下的run.sh和run.bat文件打包至输出包的bin目录下。
打包描述文件所支持的配置远超出本文所能覆盖的范围，为了避免读者被过多细节扰乱思维，这里不再展开，读者若有需要可以去参考这份文档。

最后，我们需要配置maven-assembly-plugin使用打包描述文件，并绑定生命周期阶段使其自动执行打包操作：

```
  <plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-assembly-plugin</artifactId>
    <version>2.2.1</version>
    <configuration>
      <descriptors>
        <descriptor>src/main/assembly/assembly.xml</descriptor>
      </descriptors>
    </configuration>
    <executions>
      <execution>
        <id>make-assembly</id>
        <phase>package</phase>
        <goals>
          <goal>single</goal>
        </goals>
      </execution>
    </executions>
  </plugin>
```

运行mvn clean package之后，我们就能在target/目录下得到名为hello-world-1.0-bin.zip的分发包了。