把 Spring Boot 项目打成可执行 jar 时，`java -jar` 报 `ClassNotFoundException` 是常事——依赖没打进去。这篇讲一种"依赖外置"的打法：主 jar 只放自己的 class，依赖统一放到 `libs/` 目录。

# 目标

打包后能做到：`java -jar xxx.jar` 直接启动，且**依赖外置**到 `libs/` 目录（不打进主 jar），主 jar 只放自己写的 class。

核心靠两个 Maven 插件配合：

1. **maven-jar-plugin**：配置 `Main-Class` 和 `Class-Path`（告诉 JVM 去哪找依赖）；
2. **maven-dependency-plugin**：把依赖 jar 拷贝到指定目录（如 `target/libs`）。

两者必须配对：`Class-Path` 里 jar 的路径前缀，要等于依赖拷贝的目标目录。

# 配置

## 1. maven-jar-plugin

```xml
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
```

配置项说明：

| 配置 | 作用 |
|---|---|
| `addClasspath` | 是否在 manifest 里写 `Class-Path` 行，默认 `false`；设为 `true` 后启动无需手动指定 classpath |
| `classpathPrefix` | `Class-Path` 里每个 jar 的前缀，本质是依赖 jar 所在文件夹，必须与实际拷贝目录一致 |
| `mainClass` | 指定启动的 Main Class |

`addClasspath` 开启后，生成的 manifest 会多出一行（下面是示例）：

```text
Manifest-Version: 1.0
Archiver-Version: Plexus Archiver
Built-By: michealyang
Class-Path: libs/jetty-server-9.4.7.v20170914.jar libs/javax.servlet-api-3.1.0.jar libs/jetty-http-9.4.7.v20170914.jar
Created-By: Apache Maven 3.3.9
Build-Jdk: 1.8.0_162-ea
Main-Class: com.michealyang.jetty.embeded.EmbeddedJettyServer
```

## 2. maven-dependency-plugin

```xml
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
```

- `outputDirectory`：依赖 jar 拷贝到哪，**必须与 `classpathPrefix` 一致**；
- `phase` 绑定在 `prepare-package`，打包前完成拷贝；
- 配置照抄即可，关键就是让"拷贝目录"和"Class-Path 前缀"对上。

# 打包与运行

```shell
mvn package
java -jar embedded-jetty-1.0.0-SNAPSHOT.jar
```

产物结构：自己的 class 在主 jar 里，依赖 jar 在 `libs/` 目录中。

```text
├── embedded-jetty-1.0.0-SNAPSHOT.jar
├── libs
│   ├── jetty-server-9.4.7.v20170914.jar
│   ├── jetty-http-9.4.7.v20170914.jar
```

# 优点

- 配置项多、很自由，每个步骤都可控；
- 依赖外置，主 jar 小，替换 / 复用依赖方便。

# 对比：Fat Jar 方案

上面的做法依赖外置；如果你想打成"依赖全塞进一个 jar"的 **fat jar（uber jar）**，用下面的插件（二选一）：

| 插件 | 特点 |
|---|---|
| `maven-shade-plugin` | 把依赖解压后合并进一个 jar，支持 `relocation` 解决包冲突，最常用 |
| `maven-assembly-plugin` | 用 `jar-with-dependencies` 描述符打包，简单但不能重定位 |

# 小结

1. **怎么打可执行 jar？** `maven-jar-plugin` 配 `Main-Class` 和 `Class-Path`，`maven-dependency-plugin` 把依赖拷到 `libs/`。
2. **两个插件怎么配合？** `classpathPrefix`（jar 里的路径前缀）必须等于 `copy-dependencies` 的 `outputDirectory`，否则启动找不到依赖。
3. **外置依赖和 fat jar 怎么选？** 外置依赖适合可控 / 复用；fat jar 适合直接分发单文件。
4. **`java -jar` 报 `ClassNotFoundException`？** 多半是 `Class-Path` 和拷贝目录不一致，或 manifest 里没写 `Class-Path`（`addClasspath` 没开）。

# 参考

- [两种插件打可执行 jar](https://www.jianshu.com/p/0d85d0539b1a)
