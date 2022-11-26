### JDK、JRE和JVM之间的关系
- JDK（Java Development Kit）：是java的核心，运行java必须要有的东西，里面包括java运行环境JRE、java工具包和java基础类库（java开发者使用的功能型类库）；
- JRE（Java Runtime Environment）：运行java所必须的环境，里面包括JVM的实现和java核心类库（JVM工作所需的类库）；
- JVM（Java Virtual Machine）：JVM 是java跨平台的核心，通过JVM屏蔽底层系统的差异，实现一次编译，处处运行。

### Java程序执行流程
- Java文件进行编译得到字节码文件；
- JVM的类加载器加载字节码文件到JVM；
- JVM内对应的执行引擎执行加载到的的字节码文件，并将其解析成对应的二进制可执行文件；
- 执行二进制文件调用操作系统的接口；

### 如何查看编译后的字节码文件
- javap -v className
- 插件：jclasslib Bytecode viewer