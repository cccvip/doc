
使用maven-jar-plugin和maven-dependency-plugin
首先，maven-jar-plugin的作用是配置mainClass和指定classpath。

<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-jar-plugin</artifactId>
    <configuration>
        <archive>
            <manifest>
                <addClasspath>true</addClasspath>
                <classpathPrefix>libs/</classpathPrefix>
                <mainClass>
                    org.baeldung.executable.ExecutableMavenJar
                </mainClass>
            </manifest>
        </archive>
    </configuration>
</plugin>
addClasspath: 是否在manifest文件中添加classpath。默认为false。如果为true，则会在manifest文件中添加classpath，这样在启动的时候就不用再手动指定classpath了。如下所示，文件中增加了Class-Path一行

Manifest-Version: 1.0                                                                                                                                         
Archiver-Version: Plexus Archiver
Built-By: michealyang
Class-Path: libs/jetty-server-9.4.7.v20170914.jar lib/javax.servlet-api
 -3.1.0.jar libs/jetty-http-9.4.7.v20170914.jar 
Created-By: Apache Maven 3.3.9
Build-Jdk: 1.8.0_162-ea
Main-Class: com.michealyang.jetty.embeded.EmbeddedJettyServer
classpathPrefix: classpath的前缀。如上面的manifest文件中，Class-Path的值中，每个jar包的前缀都是libs/。本质上，这个配置的值是所依赖jar包所在的文件夹。配置正确了才能找到依赖
mainClass: 指定启动时的Main Class

其次，maven-dependency-plugin会把所依赖的jar包copy到指定目录

<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-dependency-plugin</artifactId>
    <executions>
        <execution>
            <id>copy-dependencies</id>
            <phase>prepare-package</phase>
            <goals>
                <goal>copy-dependencies</goal>
            </goals>
            <configuration>
                <outputDirectory>
                    ${project.build.directory}/libs
                </outputDirectory>
            </configuration>
        </execution>
    </executions>
</plugin>
executions中的配置都很重要，按照上面的配置来就行了。outputDirectory指定了要将所依赖的jar包copy到哪个目录。要与maven-jar-plugin中的classpathPrefix一致。

执行如下命令，即可打包：

mvn package

打包结果是，自己写的Class在jar包中，所依赖的jar包在libs目录中:

├── embedded-jetty-1.0.0-SNAPSHOT.jar
├── lib
│ ├── jetty-server-9.4.7.v20170914.jar
│ ├── jetty-http-9.4.7.v20170914.jar

执行如下命令即可启动jar包：

java -jar embedded-jetty-1.0.0-SNAPSHOT.jar

优点
有诸多配置项，很自由，每个步骤都可控


参考文档地址：https://www.jianshu.com/p/0d85d0539b1a