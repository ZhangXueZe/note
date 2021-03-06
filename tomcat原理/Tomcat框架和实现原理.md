<!-- TOC -->

- [Tomcat框架和实现原理](#tomcat框架和实现原理)
    - [自己的思考和学习环境搭建：](#自己的思考和学习环境搭建)
        - [Tomcat的功能描述：](#tomcat的功能描述)
        - [Tomcat的编译和调试：](#tomcat的编译和调试)
            - [获取源代码，完成编译：](#获取源代码完成编译)
            - [本地调试：](#本地调试)
                - [添加Eclipse支持：](#添加eclipse支持)
                - [导入Eclipse：](#导入eclipse)
                - [修改调试入口脚本：](#修改调试入口脚本)
                - [开始调试：](#开始调试)
            - [偷懒的调试方式：](#偷懒的调试方式)
    - [Tomcat的启动逻辑：](#tomcat的启动逻辑)
        - [从启动脚本开始：](#从启动脚本开始)
        - [Tomcat的启动入口代码：](#tomcat的启动入口代码)
    - [Tomcat的启动流程代码分析](#tomcat的启动流程代码分析)
        - [实例化Boostrap对象：](#实例化boostrap对象)
        - [Boostrap调用init初始化方法：](#boostrap调用init初始化方法)
            - [初始化类加载器：](#初始化类加载器)
                - [JVM的父类委托机制类加载：](#jvm的父类委托机制类加载)
                - [java的双亲委派机制类加载：](#java的双亲委派机制类加载)
                - [tomcat的类加载机制：](#tomcat的类加载机制)
            - [设置当前线程的上下文类加载器：](#设置当前线程的上下文类加载器)
            - [使用这个加载器来加载自己的类：](#使用这个加载器来加载自己的类)
            - [最后将这个实例作为catalinaDaemon：](#最后将这个实例作为catalinadaemon)
        - [Boostrap调用setAwait方法：](#boostrap调用setawait方法)
        - [Boostrap调用load方法实现服务器的初始化：](#boostrap调用load方法实现服务器的初始化)
            - [初始化操作：](#初始化操作)
            - [配置文件server.xml的读入：](#配置文件serverxml的读入)
            - [设置标准输出和错误输出为 SystemLogHandler 接管：](#设置标准输出和错误输出为-systemloghandler-接管)
            - [服务的启动：](#服务的启动)
            - [启动时间计算：](#启动时间计算)
        - [load代码实现细节：](#load代码实现细节)
            - [脚本启动和关闭的是同一个服务实例：](#脚本启动和关闭的是同一个服务实例)
            - [配置文件解析和服务实例的生成详解：](#配置文件解析和服务实例的生成详解)
            - [实例化的server启动：](#实例化的server启动)
                - [super.initInternal()流程分析：](#superinitinternal流程分析)
                - [globalNamingResources.init()流程分析：](#globalnamingresourcesinit流程分析)
                - [遍历类加载器，对加载jar包校验：](#遍历类加载器对加载jar包校验)
                - [对Service的初始化调用：](#对service的初始化调用)
                - [编写测试程序模拟getServer().init()调用流程：](#编写测试程序模拟getserverinit调用流程)
                - [完整的Server.init()调用流程：](#完整的serverinit调用流程)
            - [Service.init()的调用：](#serviceinit的调用)
                - [Service的构成：](#service的构成)
                - [父类的initInternal()调用：](#父类的initinternal调用)
                - [Engine实例和Service的绑定：](#engine实例和service的绑定)
                - [Engine的初始化：](#engine的初始化)
                    - [StandardEngine调用getRealm：](#standardengine调用getrealm)
                    - [ContainerBase调用reconfigureStartStopExecutor：](#containerbase调用reconfigurestartstopexecutor)
                    - [LifecycleMBeanBase执行initInternal：](#lifecyclembeanbase执行initinternal)
                    - [StandardEngine初始化小结：](#standardengine初始化小结)
                - [Executor的初始化：](#executor的初始化)
                    - [Executer生成：](#executer生成)
                    - [Executer初始化：](#executer初始化)
                    - [StandardThreadExecutor的作用](#standardthreadexecutor的作用)
                - [MapperListener的初始化：](#mapperlistener的初始化)
        - [Boostrap调用start方法启动服务器：](#boostrap调用start方法启动服务器)
            - [Engine的启动：](#engine的启动)
    - [Tomcat的总体设计框架：](#tomcat的总体设计框架)
        - [总体结构：](#总体结构)
            - [类和对应的概念：](#类和对应的概念)
            - [tomcat启动时序关系：](#tomcat启动时序关系)
    - [Tomcat的功能和实现细节：](#tomcat的功能和实现细节)
        - [ContainerBase中的ContainerBackgroundProcessor：](#containerbase中的containerbackgroundprocessor)
        - [Tomcat容器的生命周期管理：](#tomcat容器的生命周期管理)
            - [生命周期管理：LifeCycle接口](#生命周期管理lifecycle接口)
            - [LifecycleMBeanBase的实现：](#lifecyclembeanbase的实现)
            - [Registry的作用：](#registry的作用)
    - [Tomcat编译脚本和整体呈现形式分析：](#tomcat编译脚本和整体呈现形式分析)
        - [Ant构建脚本分析：](#ant构建脚本分析)

<!-- /TOC -->


# Tomcat框架和实现原理

java web开发始终离不开servlet容器，其中最出名的就是Apache的顶级开源项目Tomcat了。
并且各大厂商只要有技术能力和资金的都会对这个开源项目进行定制，完成个性化的需求，例如：安全补丁，稳定性，内部分析等等。
作为一个后端开发来讲，如果不理解Tomcat的运行原理，就无法正确的配合物理资源进行开发。



## 自己的思考和学习环境搭建：
软件工程最终的产品就是这个经过工程管理最后得到的软件，还有这些软件的升级调整等业务。所以对于开源代码的学习是非常重要的。
只有看过的代码足够多，并且认真分析这些代码的组成过程，才能让自己的知识面没有盲点。才能支持自己的业务向上发展。

### Tomcat的功能描述：
Tomcat是一种web容器，用来接收http请求，并将请求得到的结果返回。既然要满足这个需求，一般的思路为：
要有一个网络连接通信维护者和一个请求处理者。通信维护者能够监听到请求并返回数据；请求处理者能够将请求进行分解处理，得到请求应该返回的数据。
Tomcat的设计核心就是这样的，由两大组件组成：Connector和Container。Connector负责的是底层的网络通信的实现，而Container负责的是上层servlet业务的实现。

### Tomcat的编译和调试：
能够拿到tomcat的最新代码，通过编译和调试，完成对这个开源软件的动态分析。

#### 获取源代码，完成编译：
访问apache的官方git托管仓库：http://git.apache.org/
然后下载最新的代码：
```shell
git clone git://git.apache.org/tomcat.git
```
拷贝源代码下的build.properties.default为build.properties，然后将其中的base.path修改为当前系统下，对tomcat进行编译需要的第三方库的缓存路径。
然后搭建ant编译环境，使用ant对tomcat源代码进行编译，只有通过这个编译，后续的eclipse才能够正确的对tomcat进行调试了。
进入tomcat源代码目录，执行：
```shell
ant
ant ide-eclipse
```
这一句命令就完成了编译。最后会在当前目录的output下生成build文件夹，下面就是tomcat打包解压缩之后的目录结构了。

参考：
> - [Building Tomcat](https://tomcat.apache.org/tomcat-9.0-doc/building.html)

#### 本地调试：
为了方便自己更加方便的对tomcat进行理解，本地搭建调试环境就非常重要了。但是tomcat默认使用ant进行编译管理，不支持Maven和Gradle编译。
但是官方给出了一个使用Eclipse进行调试的方式，但是不提供任何保证，经过测试，确实可以测试，方便很多。

##### 添加Eclipse支持：
tomcat现在只支持eclipse进行调试，IDEA还无法完成调试。
为了支持eclipse，需要使用ant来生成工程需要的基本文件和依赖。进入tomcat源代码目录，执行：
```shell
ant ide-eclipse
```
##### 导入Eclipse：
然后打开eclipse，进入：windows -> Preferences，然后选择： Java->Build Path->Classpath Variables
设置两个变量：
> - TOMCAT_LIBS_BASE：将之前build.properties中的base.path路径放在这儿
> - ANT_HOME：将ant的安装路径放在这儿

然后使用：File->Import and choose Existing Projects into Workspace 导入tomcat工程。
##### 修改调试入口脚本：
为了开始调试，最关键的一点开始了，需要修改res/ide-support/eclipse/目录下的两个文件：start-tomcat.launch和stop-tomcat.launch。将其中的：
```xml
<listEntry value="/tomcat/java/org/apache/catalina/startup/Bootstrap.java"/>
<stringAttribute key="org.eclipse.jdt.launching.PROJECT_ATTR" value="tomcat"/>
<stringAttribute key="org.eclipse.jdt.launching.VM_ARGUMENTS" value="-Dcatalina.home=${project_loc:/tomcat/java/org/apache/catalina/startup/Bootstrap.java}/output/build"/>
```
里面的project起始路径设置为当前的工程名称，否则会找不到Bootstrap这个类。
##### 开始调试：
在Eclipse中右击start-tomcat.launch这个文件，点Run As启动Tomcat，点Debug As可以调试Tomcat。

需要注意的是，如果没有对上述入口脚本修改，就需要在运行配置中对这两个脚本进行配置检查。
打开run as configuration，然后在Argument中添加属性：
```shell
Program arguments : start
VM arguments : -Dcatalina.home=${project_loc:/tomcat/java/org/apache/catalina/startup/Bootstrap.java}/output/build
```
对于stop-tomcat.launch也是同样操作。
然后就可以正常的点击debug和run来启动tomcat来进行调试和运行了。

参考：
> - [源码调试tomcat](http://www.cnblogs.com/lixiaojiao-hit/p/5096743.html)
> - [Tomcat 8 源码学习一之导入到IDEA](https://emacsist.github.io/2016/06/27/Tomcat-8-%E6%BA%90%E7%A0%81%E5%AD%A6%E4%B9%A0%E4%B8%80%E4%B9%8B%E5%AF%BC%E5%85%A5%E5%88%B0IDEA/)
> - [调试跟进 tomcat 源码](http://alphahinex.github.io/2015/10/14/how-to-debug-into-tomcat-sources)
> - [idea中编译tomcat源码(译)](http://foolchild.cn/2015/12/28/import_tomcat_to_idea)
> - [编写自己的tomcat, 并运行tomcat源码于eclipse中](https://my.oschina.net/xpbug/blog/53610)

#### 偷懒的调试方式：
因为java的类加载机制，会优先加载src下的java文件（编译出的class），而不是jar包中的class。
所以我们可以将tomcat的源代码放在自己的web工程下，然后直接进行调试的。
具体方式后续补充。


## Tomcat的启动逻辑：
只有知道了Tomcat启动过程，才能对Connector和Container是怎么初始化的了然于胸，知道了Connector和Container的初始化才能准确的把握其结构。

### 从启动脚本开始：
分析从当前tomcat的bin目录下的执行脚本startup.bat开始，最关键的步骤为：
```shell
set "EXECUTABLE=%CATALINA_HOME%\bin\catalina.bat"
call "%EXECUTABLE%" start %CMD_LINE_ARGS%
```
总体上：tomcat的startup.sh脚本主要用来判断环境，找到catalina.sh脚本源路径，将启动命令参数传递给catalina.bat执行。
继续进入catalina.bat查看关键代码：
```shell
if "%CLASSPATH%" == "" goto emptyClasspath
set "CLASSPATH=%CLASSPATH%;"
:emptyClasspath
set "CLASSPATH=%CLASSPATH%%CATALINA_HOME%\bin\bootstrap.jar"

if not "%CATALINA_TMPDIR%" == "" goto gotTmpdir
set "CATALINA_TMPDIR=%CATALINA_BASE%\temp"
:gotTmpdir

rem Add tomcat-juli.jar to classpath
rem tomcat-juli.jar can be over-ridden per instance
if not exist "%CATALINA_BASE%\bin\tomcat-juli.jar" goto juliClasspathHome
set "CLASSPATH=%CLASSPATH%;%CATALINA_BASE%\bin\tomcat-juli.jar"
goto juliClasspathDone
:juliClasspathHome
set "CLASSPATH=%CLASSPATH%;%CATALINA_HOME%\bin\tomcat-juli.jar"
:juliClasspathDone
```
这个代码将当前bin路径下的jar包添加到classpath中，供后续程序运行时查找对应的库。然后看看，具体从哪个库开始执行：
```shell
echo Using CATALINA_BASE:   "%CATALINA_BASE%"
echo Using CATALINA_HOME:   "%CATALINA_HOME%"
echo Using CATALINA_TMPDIR: "%CATALINA_TMPDIR%"
if ""%1"" == ""debug"" goto use_jdk
echo Using JRE_HOME:        "%JRE_HOME%"
goto java_dir_displayed
:use_jdk
echo Using JAVA_HOME:       "%JAVA_HOME%"
:java_dir_displayed
echo Using CLASSPATH:       "%CLASSPATH%"

set _EXECJAVA=%_RUNJAVA%
set MAINCLASS=org.apache.catalina.startup.Bootstrap
set ACTION=start
set SECURITY_POLICY_FILE=
set DEBUG_OPTS=
set JPDA=
```
通过环境变量设置了catalina.bat执行的参数，然后指定入口的类。然后开始执行从startup.bat中传入的start启动命令脚本：
```shell
:doStart
shift
if "%TITLE%" == "" set TITLE=Tomcat
set _EXECJAVA=start "%TITLE%" %_RUNJAVA%
if not ""%1"" == ""-security"" goto execCmd
shift
echo Using Security Manager
set "SECURITY_POLICY_FILE=%CATALINA_BASE%\conf\catalina.policy"
goto execCmd
```
这儿又指定了启动的配置文件。
知道了启动的jar包位置，jar包中的入口class名称，还有对应的配置文件，那么就可以进入：
apache-tomcat-9.0.0.M22-src\java\org\apache\catalina\startup\Bootstrap.java
这个代码文件进行查看了。

### Tomcat的启动入口代码：
根据上述脚本，我们可以将tomcat的启动入口定位，并且得到入口调用的具体参数：
org.apache.catalina.startup.Boostrap start
进入tomcat代码，查看对应包下的内容为：
```java
/**
* Main method and entry point when starting Tomcat via the provided
* scripts.
*
* @param args Command line arguments to be processed
*/
public static void main(String args[]) {

    if (daemon == null) {
        // Don't set daemon until init() has completed
        Bootstrap bootstrap = new Bootstrap();
        try {
            bootstrap.init();
        } catch (Throwable t) {
            handleThrowable(t);
            t.printStackTrace();
            return;
        }
        daemon = bootstrap;
    } else {
        // When running as a service the call to stop will be on a new
        // thread so make sure the correct class loader is used to prevent
        // a range of class not found exceptions.
        Thread.currentThread().setContextClassLoader(daemon.catalinaLoader);
    }

    try {
        String command = "start";
        if (args.length > 0) {
            command = args[args.length - 1];
        }

        if (command.equals("startd")) {
            args[args.length - 1] = "start";
            daemon.load(args);
            daemon.start();
        } else if (command.equals("stopd")) {
            args[args.length - 1] = "stop";
            daemon.stop();
        } else if (command.equals("start")) {
            daemon.setAwait(true);
            daemon.load(args);
            daemon.start();
        } else if (command.equals("stop")) {
            daemon.stopServer(args);
        } else if (command.equals("configtest")) {
            daemon.load(args);
            if (null==daemon.getServer()) {
                System.exit(1);
            }
            System.exit(0);
        } else {
            log.warn("Bootstrap: command \"" + command + "\" does not exist.");
        }
    } catch (Throwable t) {
        // Unwrap the Exception for clearer error reporting
        if (t instanceof InvocationTargetException &&
                t.getCause() != null) {
            t = t.getCause();
        }
        handleThrowable(t);
        t.printStackTrace();
        System.exit(1);
    }
}
```
也可以用jd-gui-windows-1.4.0这个工具来打开bin目录下的Bootstrap.jar文件来查看是否对应的就是这个代码。

通过代码，这个main方法主要做了4件事：
> - 实例化Boostrap对象，并调用其init方法；
> - 调用setAwait方法；
> - 调用load方法；
> - 调用start方法；

通过上述步骤，就完成了tomcat服务器的初始化和启动，放在webapp下的war包应用就可以在浏览器中被访问了。

## Tomcat的启动流程代码分析
Tomcat的启动流程是整个软件的核心功能的集中体现。

### 实例化Boostrap对象：
在早期的tomcat中，boostrap的实例创建中，是没有初始化操作的，是放在init调用的时候的进行的。
但是在最新的tomcat9中，使用了类的静态代码块来完成这个初始化操作。这样对于一些基本环境的设置都在类的构造之前完成了。
具体的静态初始化代码块内容为：
```java
static {
    // Will always be non-null
    String userDir = System.getProperty("user.dir");

    // Home first
    String home = System.getProperty(Globals.CATALINA_HOME_PROP);
    File homeFile = null;

    if (home != null) {
        File f = new File(home);
        try {
            homeFile = f.getCanonicalFile();
        } catch (IOException ioe) {
            homeFile = f.getAbsoluteFile();
        }
    }

    if (homeFile == null) {
        // First fall-back. See if current directory is a bin directory
        // in a normal Tomcat install
        File bootstrapJar = new File(userDir, "bootstrap.jar");

        if (bootstrapJar.exists()) {
            File f = new File(userDir, "..");
            try {
                homeFile = f.getCanonicalFile();
            } catch (IOException ioe) {
                homeFile = f.getAbsoluteFile();
            }
        }
    }

    if (homeFile == null) {
        // Second fall-back. Use current directory
        File f = new File(userDir);
        try {
            homeFile = f.getCanonicalFile();
        } catch (IOException ioe) {
            homeFile = f.getAbsoluteFile();
        }
    }

    catalinaHomeFile = homeFile;
    System.setProperty(
            Globals.CATALINA_HOME_PROP, catalinaHomeFile.getPath());

    // Then base
    String base = System.getProperty(Globals.CATALINA_BASE_PROP);
    if (base == null) {
        catalinaBaseFile = catalinaHomeFile;
    } else {
        File baseFile = new File(base);
        try {
            baseFile = baseFile.getCanonicalFile();
        } catch (IOException ioe) {
            baseFile = baseFile.getAbsoluteFile();
        }
        catalinaBaseFile = baseFile;
    }
    System.setProperty(
            Globals.CATALINA_BASE_PROP, catalinaBaseFile.getPath());
}
```
上述代码块的主要作用就是检查当前路径下的配置文件是否存在，然后读入配置文件，设置系统的环境变量。然后调用默认的构造函数来完成类的初始化操作。

关于静态代码块，一些基本的概念为：
> - 它是随着类的加载而执行，只执行一次，并优先于主函数。具体说，静态代码块是由类调用的。类调用时，先执行静态代码块，然后才执行主函数的。
> - 静态代码块其实就是给类初始化的，而构造代码块是给对象初始化的。
> - 静态代码块中的变量是局部变量，与普通函数中的局部变量性质没有区别。
> - 一个类中可以有多个静态代码块

### Boostrap调用init初始化方法：
创建一个Boostrap对象之后，然后是调用init方法来进行初始化设置，具体内容为：
```java
/**
* Initialize daemon.
* @throws Exception Fatal initialization error
*/
public void init() throws Exception {

    initClassLoaders();

    Thread.currentThread().setContextClassLoader(catalinaLoader);

    SecurityClassLoad.securityClassLoad(catalinaLoader);

    // Load our startup class and call its process() method
    if (log.isDebugEnabled())
        log.debug("Loading startup class");
    Class<?> startupClass =
        catalinaLoader.loadClass
        ("org.apache.catalina.startup.Catalina");
    Object startupInstance = startupClass.newInstance();

    // Set the shared extensions class loader
    if (log.isDebugEnabled())
        log.debug("Setting startup class properties");
    String methodName = "setParentClassLoader";
    Class<?> paramTypes[] = new Class[1];
    paramTypes[0] = Class.forName("java.lang.ClassLoader");
    Object paramValues[] = new Object[1];
    paramValues[0] = sharedLoader;
    Method method =
        startupInstance.getClass().getMethod(methodName, paramTypes);
    method.invoke(startupInstance, paramValues);

    catalinaDaemon = startupInstance;
}
```
这是一个统一的调用入口，包含很多其他的初始化操作，主要有：
> - 初始化类加载器:
```java
initClassLoaders();
```
> - 设置当前线程的类加载器：
```java
Thread.currentThread().setContextClassLoader(catalinaLoader);
```
> - 然后有一个奇怪名字的类加载器：SecurityClassLoad.securityClassLoad：
```java
SecurityClassLoad.securityClassLoad(catalinaLoader);
```
> - 用自己的类加载器加载这个类：org.apache.catalina.startup.Catalina：
```java
 // Load our startup class and call its process() method
if (log.isDebugEnabled())
    log.debug("Loading startup class");
Class<?> startupClass =
    catalinaLoader.loadClass
    ("org.apache.catalina.startup.Catalina");
Object startupInstance = startupClass.newInstance();
```
> - 然后获取参数，用这些参数调用刚刚加载器获取的类的实例的方法：
```java
// Set the shared extensions class loader
if (log.isDebugEnabled())
    log.debug("Setting startup class properties");
String methodName = "setParentClassLoader";
Class<?> paramTypes[] = new Class[1];
paramTypes[0] = Class.forName("java.lang.ClassLoader");
Object paramValues[] = new Object[1];
paramValues[0] = sharedLoader;
Method method =
    startupInstance.getClass().getMethod(methodName, paramTypes);
method.invoke(startupInstance, paramValues);
```
> - 最后将这个类返回：
```java
catalinaDaemon = startupInstance;
```

其中比较关键的内容有：
1 初始化一个类加载器；
2 使用了自定义的类加载器来完成整个servlet容器的类加载；
3 SecurityClassLoad的使用。
下面我们来逐一过一遍。

#### 初始化类加载器：
所谓的类加载器，他的主要作用是：
新建一个Java对象的时候，JVM要将这个对象对应的字节码加载到内存中，这个字节码的原始信息存放在编译生成的.class文件中，类加载器需要将这些存储在硬盘（或者网络）的.class文件经过一些处理之后变成字节码在加载到内存中。
要创建用户自己的类加载器，只需要扩展java.lang.ClassLoader类，然后覆盖它的findClass(String name)方法即可，该方法根据参数指定类的名字，返回对应的Class对象的引用。

##### JVM的父类委托机制类加载：
在JVM中并不是一次性把所有的文件都加载到，而是一步一步的，按照需要来加载。因此使用哪种类加载器、在什么位置加载类都是JVM中重要的内容，也是理解整个类加载的核心所在。
因为我们在启动tomcat之前，java的运行环境已经可用了，然后通过java命令来对tomcat进行启动，那么首先完成的必然是JVM对当前java执行环境的类加载，并且提供系统类加载给java应用使用。

JVM的初始化类加载器是最特殊的，首先需要从native环境启动JVM，然后通过JVM的类加载器初始化执行环境。
JVM的类加载采用了父类委托机制。JVM的ClassLoader通过Parent属性定义父子关系，可以形成树状结构。其中引导类、扩展类、系统类三个加载器是JVM内置的。
它们的作用分别是：
> - 1）启动（Bootstrap）类加载器：是用本地代码实现的类装入器，它负责将 <Java_Runtime_Home>/lib下面的类库加载到内存中（比如rt.jar）。由于引导类加载器涉及到虚拟机本地实现细节，开发者无法直接获取到启动类加载器的引用，所以不允许直接通过引用进行操作。
> - 2）标准扩展（Extension）类加载器：是由 Sun 的 ExtClassLoader（sun.misc.Launcher$ExtClassLoader）实现的。它负责将< Java_Runtime_Home >/lib/ext或者由系统变量 java.ext.dir指定位置中的类库加载到内存中。开发者可以直接使用标准扩展类加载器。
> - 3）系统（System）类加载器：是由 Sun 的 AppClassLoader（sun.misc.Launcher$AppClassLoader）实现的。它负责将系统类路径（CLASSPATH）中指定的类库加载到内存中。开发者可以直接使用系统类加载器。

Java虚拟机的第一个类加载器是Bootstrap，这个加载器很特殊，它不是Java类，因此它不需要被别人加载，它嵌套在Java虚拟机内核里面，也就是JVM启动的时候Bootstrap就已经启动，它是用C++写的二进制代码（不是字节码），它可以去加载别的类。
这也是我们在测试时为什么发现System.class.getClassLoader()结果为null的原因，这并不表示System这个类没有类加载器，而是它的加载器比较特殊，是BootstrapClassLoader，由于它不是Java类，因此获得它的引用肯定返回null。

总结父类委托机制为：
> - 1 用户自己的类加载器，把加载请求传给父加载器，父加载器再传给其父加载器，一直到加载器树的顶层。
> - 2 最顶层的类加载器首先针对其特定的位置加载，如果加载不到就转交给子类。
> - 3 如果一直到底层的类加载（引导类加载器）都没有加载到，那么就会抛出异常ClassNotFoundException。

##### java的双亲委派机制类加载：
在完成了JVM的启动和环境初始化之后，对于java应用的加载，使用双亲委派机制进行加载。
因为JVM默认的类加载器之后就是用户自定义的加载器了，这样就从系统加载器开始形成了类加载的树形结构，那加载应用类时就需要定义类到底由当前加载器还是父加载器去搜索加载。

所谓双亲委派是指子类加载器如果没有加载过该目标类，就先委托父类加载器加载该目标类，只有在父类加载器找不到字节码文件的情况下才从自己的类路径中查找并装载目标类。“双亲委派”机制加载Class的具体过程是：
> - 源ClassLoader先判断该Class是否已加载，如果已加载，则返回Class对象；如果没有则委托给父类加载器。
> - 父类加载器判断是否加载过该Class，如果已加载，则返回Class对象；如果没有则委托给祖父类加载器。
> - 依此类推，直到始祖类加载器（引用类加载器）。
> - 始祖类加载器判断是否加载过该Class，如果已加载，则返回Class对象；如果没有则尝试从其对应的类路径下寻找class字节码文件并载入。如果载入成功，则返回Class对象；如果载入失败，则委托给始祖类加载器的子类加载器。
> - 始祖类加载器的子类加载器尝试从其对应的类路径下寻找class字节码文件并载入。如果载入成功，则返回Class对象；如果载入失败，则委托给始祖类加载器的孙类加载器。
> - > - 依此类推，直到源ClassLoader。
> - 源ClassLoader尝试从其对应的类路径下寻找class字节码文件并载入。如果载入成功，则返回Class对象；如果载入失败，源ClassLoader不会再委托其子类加载器，而是抛出异常。

需要注意的是，“双亲委派”机制只是Java推荐的机制，并不是强制的机制。

##### tomcat的类加载机制：
因为tomcat作为java应用，如果要完成功能，首先能依赖的就只有系统类加载器，然后自己在构建自己的类加载体系。

我们进入tomcat的boostrap看看他自己的类加载器初始化在干什么：
```java
private void initClassLoaders() {
    try {
        commonLoader = createClassLoader("common", null);
        if( commonLoader == null ) {
            // no config file, default to this loader - we might be in a 'single' env.
            commonLoader=this.getClass().getClassLoader();
        }
        catalinaLoader = createClassLoader("server", commonLoader);
        sharedLoader = createClassLoader("shared", commonLoader);
    } catch (Throwable t) {
        handleThrowable(t);
        log.error("Class loader creation threw exception", t);
        System.exit(1);
    }
}
```


上述代码中，实现了三个自定义类加载器的初始化：
```java
commonLoader = createClassLoader("common", null);
catalinaLoader = createClassLoader("server", commonLoader);
sharedLoader = createClassLoader("shared", commonLoader);
```
通过参数来设置类加载器的名称和父加载器，并且使用了createClassLoader函数来简化类加载器的生成操作，我们进入createClassLoader看看是如何实现的：
```java
private ClassLoader createClassLoader(String name, ClassLoader parent)
    throws Exception {

    String value = CatalinaProperties.getProperty(name + ".loader");
    if ((value == null) || (value.equals("")))
        return parent;

    value = replace(value);

    List<Repository> repositories = new ArrayList<>();

    String[] repositoryPaths = getPaths(value);

    for (String repository : repositoryPaths) {
        // Check for a JAR URL repository
        try {
            @SuppressWarnings("unused")
            URL url = new URL(repository);
            repositories.add(
                    new Repository(repository, RepositoryType.URL));
            continue;
        } catch (MalformedURLException e) {
            // Ignore
        }

        // Local repository
        if (repository.endsWith("*.jar")) {
            repository = repository.substring
                (0, repository.length() - "*.jar".length());
            repositories.add(
                    new Repository(repository, RepositoryType.GLOB));
        } else if (repository.endsWith(".jar")) {
            repositories.add(
                    new Repository(repository, RepositoryType.JAR));
        } else {
            repositories.add(
                    new Repository(repository, RepositoryType.DIR));
        }
    }

    return ClassLoaderFactory.createClassLoader(repositories, parent);
}
```
首先，通过CatalinaProperties获取当前的配置文件，然后获取要创建的类加载器的名称对应的值；
然后，将这个值从相对路径，使用replace转换为绝对路径（这个值是当前需要加载的所有lib的路径）；
接着，对这些需要加载的库路径进行检查，按照路径和对应的类型放在Repository的列表中；
最后，调用ClassLoaderFactory来对当前的所有Repository对象创建类加载器。

可以看到是在读取配置文件，然后交给一个工厂来完成类加载器的生成。

具体方式为：
> - 1. 从CatalinaProperties中获取属性值：
进入java\org\apache\catalina\startup\CatalinaProperties.java
在静态代码段中，对配置文件进行了设置：
```java
static {
        loadProperties();
}

/**
* Load properties.
*/
private static void loadProperties() {

    InputStream is = null;
    String fileName = "catalina.properties";

    try {
        String configUrl = System.getProperty("catalina.config");
        if (configUrl != null) {
            if (configUrl.indexOf('/') == -1) {
                // No '/'. Must be a file name rather than a URL
                fileName = configUrl;
            } else {
                is = (new URL(configUrl)).openStream();
            }
        }
    } catch (Throwable t) {
        handleThrowable(t);
    }

    if (is == null) {
        try {
            File home = new File(Bootstrap.getCatalinaBase());
            File conf = new File(home, "conf");
            File propsFile = new File(conf, fileName);
            is = new FileInputStream(propsFile);
        } catch (Throwable t) {
            handleThrowable(t);
        }
    }

    if (is == null) {
        try {
            is = CatalinaProperties.class.getResourceAsStream
                ("/org/apache/catalina/startup/catalina.properties");
        } catch (Throwable t) {
            handleThrowable(t);
        }
    }

    if (is != null) {
        try {
            properties = new Properties();
            properties.load(is);
        } catch (Throwable t) {
            handleThrowable(t);
            log.warn(t);
        } finally {
            try {
                is.close();
            } catch (IOException ioe) {
                log.warn("Could not close catalina properties file", ioe);
            }
        }
    }

    if ((is == null)) {
        // Do something
        log.warn("Failed to load catalina properties file");
        // That's fine - we have reasonable defaults.
        properties = new Properties();
    }

    // Register the properties as system properties
    Enumeration<?> enumeration = properties.propertyNames();
    while (enumeration.hasMoreElements()) {
        String name = (String) enumeration.nextElement();
        String value = properties.getProperty(name);
        if (value != null) {
            System.setProperty(name, value);
        }
    }
}
```
读取当前包下的catalina.properties这个文件，然后获取common.loader这个属性的值：
```xml
common.loader="${catalina.base}/lib","${catalina.base}/lib/*.jar","${catalina.home}/lib","${catalina.home}/lib/*.jar"
```
这儿出现的${catalina.home}是通过启动脚本获取，然后传递进入的：
```bat
-Dcatalina.home="%CATALINA_HOME%"
```
也就是当前tomcat的根目录，所以上述配置文件设置的路径就是当前tomcat运行环境的根目录下的lib，对应的也就是out/build目录下的lib文件夹。

> - 2. 将这个这个属性的值做一些处理（因为有分好分割的多个，最终转换为一个list），得到一个Repository的列表，然后交给类加载器工厂处理，简化后的流程代码为：
```java
List<Repository> repositories = new ArrayList<>();
repositories.add(new Repository(repository, RepositoryType.URL));
ClassLoaderFactory.createClassLoader(repositories, parent);
```
其中，Repository这个类型是在ClassLoaderFactory内部定义的子类，进入java\org\apache\catalina\startup\ClassLoaderFactory.java，查看代码为：
```java
public static class Repository {
    private final String location;
    private final RepositoryType type;

    public Repository(String location, RepositoryType type) {
        this.location = location;
        this.type = type;
    }

    public String getLocation() {
        return location;
    }

    public RepositoryType getType() {
        return type;
    }
}

/**
* Create and return a new class loader, based on the configuration
* defaults and the specified directory paths:
*
* @param repositories List of class directories, jar files, jar directories
*                     or URLS that should be added to the repositories of
*                     the class loader.
* @param parent Parent class loader for the new class loader, or
*  <code>null</code> for the system class loader.
* @return the new class loader
*
* @exception Exception if an error occurs constructing the class loader
*/
public static ClassLoader createClassLoader(List<Repository> repositories,
                                            final ClassLoader parent)
    throws Exception {

    if (log.isDebugEnabled())
        log.debug("Creating new class loader");

    // Construct the "class path" for this class loader
    Set<URL> set = new LinkedHashSet<>();

    if (repositories != null) {
        for (Repository repository : repositories)  {
            if (repository.getType() == RepositoryType.URL) {
                URL url = buildClassLoaderUrl(repository.getLocation());
                if (log.isDebugEnabled())
                    log.debug("  Including URL " + url);
                set.add(url);
            } else if (repository.getType() == RepositoryType.DIR) {
                File directory = new File(repository.getLocation());
                directory = directory.getCanonicalFile();
                if (!validateFile(directory, RepositoryType.DIR)) {
                    continue;
                }
                URL url = buildClassLoaderUrl(directory);
                if (log.isDebugEnabled())
                    log.debug("  Including directory " + url);
                set.add(url);
            } else if (repository.getType() == RepositoryType.JAR) {
                File file=new File(repository.getLocation());
                file = file.getCanonicalFile();
                if (!validateFile(file, RepositoryType.JAR)) {
                    continue;
                }
                URL url = buildClassLoaderUrl(file);
                if (log.isDebugEnabled())
                    log.debug("  Including jar file " + url);
                set.add(url);
            } else if (repository.getType() == RepositoryType.GLOB) {
                File directory=new File(repository.getLocation());
                directory = directory.getCanonicalFile();
                if (!validateFile(directory, RepositoryType.GLOB)) {
                    continue;
                }
                if (log.isDebugEnabled())
                    log.debug("  Including directory glob "
                        + directory.getAbsolutePath());
                String filenames[] = directory.list();
                if (filenames == null) {
                    continue;
                }
                for (int j = 0; j < filenames.length; j++) {
                    String filename = filenames[j].toLowerCase(Locale.ENGLISH);
                    if (!filename.endsWith(".jar"))
                        continue;
                    File file = new File(directory, filenames[j]);
                    file = file.getCanonicalFile();
                    if (!validateFile(file, RepositoryType.JAR)) {
                        continue;
                    }
                    if (log.isDebugEnabled())
                        log.debug("    Including glob jar file "
                            + file.getAbsolutePath());
                    URL url = buildClassLoaderUrl(file);
                    set.add(url);
                }
            }
        }
    }

    // Construct the class loader itself
    final URL[] array = set.toArray(new URL[set.size()]);
    if (log.isDebugEnabled())
        for (int i = 0; i < array.length; i++) {
            log.debug("  location " + i + " is " + array[i]);
        }

    return AccessController.doPrivileged(
        new PrivilegedAction<URLClassLoader>() {
            @Override
            public URLClassLoader run() {
                if (parent == null)
                    return new URLClassLoader(array);
                else
                    return new URLClassLoader(array, parent);
            }
        });
}
```
主要的流程就是，对当前传入的参数列表遍历，然后根据每一个参数的路径和类型，通过不同的方式进行类加载器的构建，主要的方式有：
```java
public enum RepositoryType {
    DIR,
    GLOB,
    JAR,
    URL
}
```
也就是Repository中的属性的类型，总体上，支持从：路径，全局设置，jar包和URL创建类加载器。
这个Repository作为ClassLoaderFactory的内部类，用于创建类加载器的时候给予属性记录的辅助功能。

createClassLoader的主要流程为：
> - 首先创建一个Set容器，用来存放要创建的类加载器的URL：
```java
Set<URL> set = new LinkedHashSet<>();
```
> - 然后对传入的列表遍历，根据当前输入参数的类型做参数检查，并且获取每一个变量的location属性，最后都是用buildClassLoaderUrl这个函数来生成一个URL对象，加入到容器中。例如：
```java
if (repository.getType() == RepositoryType.URL) {
    URL url = buildClassLoaderUrl(repository.getLocation());
    if (log.isDebugEnabled())
        log.debug("  Including URL " + url);
    set.add(url);
}
```
> - 最后用Set中的URL来构建出类加载器：
```java
return AccessController.doPrivileged(
    new PrivilegedAction<URLClassLoader>() {
        @Override
        public URLClassLoader run() {
            if (parent == null)
                return new URLClassLoader(array);
            else
                return new URLClassLoader(array, parent);
        }
    });
```
这个是对java.security.AccessController的使用，也就是用了java的安全模式。
doPrivileged 方法能够使一段受信任代码获得更大的权限，甚至比调用它的应用程序还要多，可做到临时访问更多的资源。
例如，应用程序可能无法直接访问某些系统资源，但这样的应用程序必须得到这些资源才能够完成功能。针对这种情况，Java SDK 给域提供了 doPrivileged 方法，让程序突破当前域权限限制，临时扩大访问权限。下面内容会详细讲解一下安全相关的方法使用。

上述函数最后返回的时候，在doPrivileged中重载了run方法，其中就包含了构造类加载器的方法调用：URLClassLoader。
这儿有一个问题，在创建common这个类加载器的时候，parent为null，但是所有的java类加载器都是有父的，这儿也说明了当父加载器没有设置的时候，使用 new URLClassLoader(array) 来构建这个加载器，也就是说最终会以系统类加载器(AppClassLoader)作为父类加载器。
如果指定了父加载器，就会在URLClassLoader中设置其父加载器，完成构造。

从JDK源码上来看其实是URLClassLoader继承了ClassLoader，也就是说URLClassLoader把ClassLoader扩展了一下，所以可以理解成URLClassLoader功能要多点。ClassLoader只能加载classpath下面的类，而URLClassLoader可以加载任意路径下的类。他们的继承关系如下：
```java
public class URLClassLoader extends SecureClassLoader {}
public class SecureClassLoader extends ClassLoader {}
```

返回到initClassLoader中：
catalinaLoader = createClassLoader("server", commonLoader);
sharedLoader = createClassLoader("shared", commonLoader);
说明这两个自定义类加载器的父是commonLoader这个自定义类加载器，到此为止，自定义的类加载器构造就完毕了。


参考：
> - [Tomcat源码解读：ClassLoader的设计](http://www.cnblogs.com/f1194361820/p/4186232.html)
> - [java安全模型介绍](https://www.ibm.com/developerworks/cn/java/j-lo-javasecurity/)
> - [java与tomcat7类加载机制](http://blog.csdn.net/czmacd/article/details/54017027)


#### 设置当前线程的上下文类加载器：
完成了自定义类加载器的初始化之后，就可以使用这些类加载器来完成类的加载了。

类 java.lang.Thread中的方法 getContextClassLoader()和setContextClassLoader(ClassLoader cl)用来获取和设置线程的上下文类加载器。如果没有通过 setContextClassLoader(ClassLoader cl)方法进行设置的话，线程将继承其父线程的上下文类加载器。Java 应用运行的初始线程的上下文类加载器是系统类加载器(appClassLoader)。在线程中运行的代码可以通过此类加载器来加载类和资源。

也就是说，如果没有将自定义的类加载器明确的设置为线程的上下文类加载器，那么就会使用系统类加载器，这样做的话就无法对不同的servlet容器实例提供不同的运行环境了，所以必然会用自定义的类加载器来替换系统的：
```java
Thread.currentThread().setContextClassLoader(catalinaLoader);
```
设置的是catalinaLoader这个类加载器，是commonloader的子加载器。

#### 使用这个加载器来加载自己的类：
使用自定义的类加载器加载指定路径的类：
```java
if (log.isDebugEnabled())
    log.debug("Loading startup class");
    Class<?> startupClass = catalinaLoader.loadClass("org.apache.catalina.startup.Catalina");
```
接着，调用Catalina的setParentClassLoader方法：
```java
Object startupInstance = startupClass.newInstance();

// Set the shared extensions class loader
if (log.isDebugEnabled())
    log.debug("Setting startup class properties");
String methodName = "setParentClassLoader";
Class<?> paramTypes[] = new Class[1];
paramTypes[0] = Class.forName("java.lang.ClassLoader");
Object paramValues[] = new Object[1];
paramValues[0] = sharedLoader;
Method method = startupInstance.getClass().getMethod(methodName, paramTypes);
method.invoke(startupInstance, paramValues);
```

#### 最后将这个实例作为catalinaDaemon：
```java
catalinaDaemon = startupInstance;
```
这个catalinaDaemon后续用来实现对Catalina的函数调用，完成后续操作。


参考：
[ClassLoader，Thread.currentThread().setContextClassLoader，tomcat的ClassLoader](http://www.cnblogs.com/549294286/p/3714692.html)


### Boostrap调用setAwait方法：
完成了类加载的初始化之后，获取到了当前Boostrap的实例，然后在start参数下，第一个调用的方式就是setAwait方法。
调用的方式也是通过类加载器返回的实例进行调用的：
```java
/**
* Set flag.
* @param await <code>true</code> if the daemon should block
* @throws Exception Reflection error
*/
public void setAwait(boolean await)
    throws Exception {

    Class<?> paramTypes[] = new Class[1];
    paramTypes[0] = Boolean.TYPE;
    Object paramValues[] = new Object[1];
    paramValues[0] = Boolean.valueOf(await);
    Method method =
        catalinaDaemon.getClass().getMethod("setAwait", paramTypes);
    method.invoke(catalinaDaemon, paramValues);

}
```
进入apache-tomcat-9.0.0.M22-src\java\org\apache\catalina\startup\Catalina.java，看一下setAwait方法实现：
```java
public void setAwait(boolean b) {
    await = b;
}
```
非常简单，就是设置了一个标志位，问题的关键是这个标志位是用来干什么的？

### Boostrap调用load方法实现服务器的初始化：
接下来就是调用load方法来获取启动的参数：
```java
/**
* Load daemon.
*/
private void load(String[] arguments)
    throws Exception {

    // Call the load() method
    String methodName = "load";
    Object param[];
    Class<?> paramTypes[];
    if (arguments==null || arguments.length==0) {
        paramTypes = null;
        param = null;
    } else {
        paramTypes = new Class[1];
        paramTypes[0] = arguments.getClass();
        param = new Object[1];
        param[0] = arguments;
    }
    Method method =
        catalinaDaemon.getClass().getMethod(methodName, paramTypes);
    if (log.isDebugEnabled())
        log.debug("Calling startup class " + method);
    method.invoke(catalinaDaemon, param);
}
```
也是通过类加载器获取的实例来完成调用的，参数为使用catalina.bat启动时传入的参数。
然后看看实际被调用的load函数的实现：
```java
/*
* Load using arguments
*/
public void load(String args[]) {

    try {
        if (arguments(args)) {
            load();
        }
    } catch (Exception e) {
        e.printStackTrace(System.out);
    }
}
```
这个函数中使用arguments函数处理了传入的参数：
```java
/**
* Process the specified command line arguments.
*
* @param args Command line arguments to process
* @return <code>true</code> if we should continue processing
*/
protected boolean arguments(String args[]) {

    boolean isConfig = false;

    if (args.length < 1) {
        usage();
        return false;
    }

    for (int i = 0; i < args.length; i++) {
        if (isConfig) {
            configFile = args[i];
            isConfig = false;
        } else if (args[i].equals("-config")) {
            isConfig = true;
        } else if (args[i].equals("-nonaming")) {
            setUseNaming(false);
        } else if (args[i].equals("-help")) {
            usage();
            return false;
        } else if (args[i].equals("start")) {
            // NOOP
        } else if (args[i].equals("configtest")) {
            // NOOP
        } else if (args[i].equals("stop")) {
            // NOOP
        } else {
            usage();
            return false;
        }
    }

    return true;
}
```
这个函数将输入的参数进行检查，然后根据输入将开关进行打开关闭设置，控制后续调用流程，并且打印使用说明等。
最终调用了无参数的load方法：
```java
/**
* Start a new server instance.
*/
public void load() {

    long t1 = System.nanoTime();

    initDirs();

    // Before digester - it may be needed
    initNaming();

    // Create and execute our Digester
    Digester digester = createStartDigester();

    InputSource inputSource = null;
    InputStream inputStream = null;
    File file = null;
    try {
        try {
            file = configFile();
            inputStream = new FileInputStream(file);
            inputSource = new InputSource(file.toURI().toURL().toString());
        } catch (Exception e) {
            if (log.isDebugEnabled()) {
                log.debug(sm.getString("catalina.configFail", file), e);
            }
        }
        if (inputStream == null) {
            try {
                inputStream = getClass().getClassLoader()
                    .getResourceAsStream(getConfigFile());
                inputSource = new InputSource
                    (getClass().getClassLoader()
                        .getResource(getConfigFile()).toString());
            } catch (Exception e) {
                if (log.isDebugEnabled()) {
                    log.debug(sm.getString("catalina.configFail",
                            getConfigFile()), e);
                }
            }
        }

        // This should be included in catalina.jar
        // Alternative: don't bother with xml, just create it manually.
        if (inputStream == null) {
            try {
                inputStream = getClass().getClassLoader()
                        .getResourceAsStream("server-embed.xml");
                inputSource = new InputSource
                (getClass().getClassLoader()
                        .getResource("server-embed.xml").toString());
            } catch (Exception e) {
                if (log.isDebugEnabled()) {
                    log.debug(sm.getString("catalina.configFail",
                            "server-embed.xml"), e);
                }
            }
        }


        if (inputStream == null || inputSource == null) {
            if  (file == null) {
                log.warn(sm.getString("catalina.configFail",
                        getConfigFile() + "] or [server-embed.xml]"));
            } else {
                log.warn(sm.getString("catalina.configFail",
                        file.getAbsolutePath()));
                if (file.exists() && !file.canRead()) {
                    log.warn("Permissions incorrect, read permission is not allowed on the file.");
                }
            }
            return;
        }

        try {
            inputSource.setByteStream(inputStream);
            digester.push(this);
            digester.parse(inputSource);
        } catch (SAXParseException spe) {
            log.warn("Catalina.start using " + getConfigFile() + ": " +
                    spe.getMessage());
            return;
        } catch (Exception e) {
            log.warn("Catalina.start using " + getConfigFile() + ": " , e);
            return;
        }
    } finally {
        if (inputStream != null) {
            try {
                inputStream.close();
            } catch (IOException e) {
                // Ignore
            }
        }
    }

    getServer().setCatalina(this);
    getServer().setCatalinaHome(Bootstrap.getCatalinaHomeFile());
    getServer().setCatalinaBase(Bootstrap.getCatalinaBaseFile());

    // Stream redirection
    initStreams();

    // Start the new server
    try {
        getServer().init();
    } catch (LifecycleException e) {
        if (Boolean.getBoolean("org.apache.catalina.startup.EXIT_ON_INIT_FAILURE")) {
            throw new java.lang.Error(e);
        } else {
            log.error("Catalina.start", e);
        }
    }

    long t2 = System.nanoTime();
    if(log.isInfoEnabled()) {
        log.info("Initialization processed in " + ((t2 - t1) / 1000000) + " ms");
    }
}
```
这段代码比较长，核心功能是启动了一个server，我们快接近tomcat的container实现了！
下面我们来认真看一下这段代码的实现过程。

参考：
> - [Tomcat6源码解析--Bootstrap.java](http://blog.csdn.net/wzy26816812/article/details/40371809)

#### 初始化操作：
这个函数执行的开始，首先记录一下当前的时间点：
```java
long t1 = System.nanoTime();
```
然后开始初始化目录：
```java
initDirs();
```
具体实现为：
```java
protected void initDirs() {
    String temp = System.getProperty("java.io.tmpdir");
    if (temp == null || (!(new File(temp)).isDirectory())) {
        log.error(sm.getString("embedded.notmp", temp));
    }
}
```
通过 System.getProperty("java.io.tmpdir") 是获取操作系统的缓存临时目录，然后判断这个目录是否存在。
接着调用 initNaming() 检查是否需要开启命名服务支持：
```java
protected boolean useNaming = true;

protected void initNaming() {
    // Setting additional variables
    if (!useNaming) {
        log.info( "Catalina naming disabled");
        System.setProperty("catalina.useNaming", "false");
    } else {
        System.setProperty("catalina.useNaming", "true");
        String value = "org.apache.naming";
        String oldValue =
            System.getProperty(javax.naming.Context.URL_PKG_PREFIXES);
        if (oldValue != null) {
            value = value + ":" + oldValue;
        }
        System.setProperty(javax.naming.Context.URL_PKG_PREFIXES, value);
        if( log.isDebugEnabled() ) {
            log.debug("Setting naming prefix=" + value);
        }
        value = System.getProperty
            (javax.naming.Context.INITIAL_CONTEXT_FACTORY);
        if (value == null) {
            System.setProperty
                (javax.naming.Context.INITIAL_CONTEXT_FACTORY,
                    "org.apache.naming.java.javaURLContextFactory");
        } else {
            log.debug( "INITIAL_CONTEXT_FACTORY already set " + value );
        }
    }
}
```
默认是命名服务支持的开关是打开的，将系统属性添加 "catalina.useNaming" ，设置这个属性值为true。

这个命名服务是干什么用的？
该方法主要用于设置javax.naming的一些需要用于的属性值,如javax.naming.Context.INITIAL_CONTEXT_FACTORY及javax.naming.Context.URL_PKG_PREFIXES

例如：
在具体的解析中，类StandardContext中进行启动时，只需要判断System.getProperty("catalina.useNaming")即可判断出是否启动命名上下文。如果启用命名服务，则会自动将NamingContextListener注册到监听器中，余下的工作就是监听器去完成了。
也就是说这个命名服务支持是用来实现tomcat的JNDI支持的。

具体的内容为：
initNaming方法给系统设置java.naming.factory.url.pkgs和java.naming.factory.initial。
在创建JNDI上下文时，使用：
Context.INITIAL_CONTEXT_FACTORY（”java.naming.factory.initial”）属性，来指定创建JNDI上下文的工厂类；
Context.URL_PKG_PREFIXES(“java.naming.factory.url.pkgs”)用在查询url中包括scheme方法id时创建对应的JNDI上下文，例如查询”java:/jdbc/test1″等类似查询上，即以冒号”:”标识的shceme。
Context.URL_PKG_PREFIXES属性值有多个java 包(package)路径，其中以冒号”:”分隔各个包路径，这些包路径中包括JNDI相关实现类。当在JNDI上下文中查找”java:”这类包括scheme方案ID的url时，InitialContext类将优先查找Context.URL_PKG_PREFIXES属性指定的包路径中是否存在 scheme+”.”+scheme + “URLContextFactory”工厂类（需要实现ObjectFactory接口），如果存在此工厂类，则调用此工厂类的getObjectInstance方法获得此scheme方案ID对应的jndi上下文，再在此上下文中继续查找对应的url。

参考：
> - [在embed tomcat中使用jndi命名服务](https://www.iflym.com/index.php/code/201208080003.html)
> - [Tomcat7.0源码分析——server.xml文件的加载与解析](http://www.milletblog.com/2016/07/Tomcat70%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90serverxml%E6%96%87%E4%BB%B6%E7%9A%84%E5%8A%A0%E8%BD%BD%E4%B8%8E%E8%A7%A3%E6%9E%90-43/)

#### 配置文件server.xml的读入：
我们在启动tomcat之前，往往会设置conf文件夹下的server.xml来定制tomcat服务器的行为，所以对于server.xml的获取非常重要。
创建一个digester来获取配置文件，并且解析：
```java
// Create and execute our Digester
Digester digester = createStartDigester();

InputSource inputSource = null;
InputStream inputStream = null;
File file = null;
try {
    try {
        file = configFile();
        inputStream = new FileInputStream(file);
        inputSource = new InputSource(file.toURI().toURL().toString());
    } catch (Exception e) {
        if (log.isDebugEnabled()) {
            log.debug(sm.getString("catalina.configFail", file), e);
        }
    }
    if (inputStream == null) {
        try {
            inputStream = getClass().getClassLoader()
                .getResourceAsStream(getConfigFile());
            inputSource = new InputSource
                (getClass().getClassLoader()
                    .getResource(getConfigFile()).toString());
        } catch (Exception e) {
            if (log.isDebugEnabled()) {
                log.debug(sm.getString("catalina.configFail",
                        getConfigFile()), e);
            }
        }
    }

    // This should be included in catalina.jar
    // Alternative: don't bother with xml, just create it manually.
    if (inputStream == null) {
        try {
            inputStream = getClass().getClassLoader()
                    .getResourceAsStream("server-embed.xml");
            inputSource = new InputSource
            (getClass().getClassLoader()
                    .getResource("server-embed.xml").toString());
        } catch (Exception e) {
            if (log.isDebugEnabled()) {
                log.debug(sm.getString("catalina.configFail",
                        "server-embed.xml"), e);
            }
        }
    }


    if (inputStream == null || inputSource == null) {
        if  (file == null) {
            log.warn(sm.getString("catalina.configFail",
                    getConfigFile() + "] or [server-embed.xml]"));
        } else {
            log.warn(sm.getString("catalina.configFail",
                    file.getAbsolutePath()));
            if (file.exists() && !file.canRead()) {
                log.warn("Permissions incorrect, read permission is not allowed on the file.");
            }
        }
        return;
    }

    try {
        inputSource.setByteStream(inputStream);
        digester.push(this);
        digester.parse(inputSource);
    } catch (SAXParseException spe) {
        log.warn("Catalina.start using " + getConfigFile() + ": " +
                spe.getMessage());
        return;
    } catch (Exception e) {
        log.warn("Catalina.start using " + getConfigFile() + ": " , e);
        return;
    }
} finally {
    if (inputStream != null) {
        try {
            inputStream.close();
        } catch (IOException e) {
            // Ignore
        }
    }
}

getServer().setCatalina(this);
getServer().setCatalinaHome(Bootstrap.getCatalinaHomeFile());
getServer().setCatalinaBase(Bootstrap.getCatalinaBaseFile());
```
总体上，均是TOMCAT服务器读取配置文件${catalina.home}/conf/server.xml的过程,这个过程也是整个服务器启动的最重要的一个过程，这个过程比较复杂，暂时不看了。

参考：
[tomcat解析(五)Digester(一)](http://blog.csdn.net/holly2k/article/details/5258829)
[tomcat解析(六)Digester(二)startElement](http://blog.csdn.net/holly2k/article/details/5258840)
[tomcat解析(七)Digester(四)characters,endElement.endDocument](http://blog.csdn.net/holly2k/article/details/5261156)
[tomcat解析(八)Catalina.createStartDigester](http://blog.csdn.net/holly2k/article/details/5258849)

#### 设置标准输出和错误输出为 SystemLogHandler 接管：
initStreams() 这个函数将System.out和System.error进行了重定向：
```java
protected void initStreams() {
    // Replace System.out and System.err with a custom PrintStream
    System.setOut(new SystemLogHandler(System.out));
    System.setErr(new SystemLogHandler(System.err));
}
```
这儿使用了System.setOut和setErr函数来做输出的重定向。重定向到自定义的SystemLogHandler类中做处理。

#### 服务的启动：
在完成了从server.xml配置文件中获取参数对server构造，并且重定向当前输出之后，就开始对这个server启动了：
```java
// Start the new server
try {
    getServer().init();
} catch (LifecycleException e) {
    if (Boolean.getBoolean("org.apache.catalina.startup.EXIT_ON_INIT_FAILURE")) {
        throw new java.lang.Error(e);
    } else {
        log.error("Catalina.start", e);
    }
}
```
这儿的getServer获取的是通过server.xml配置文件来动态生成的Server接口的一个实现，然后调用其init方法来启动。
可以看到这一步其实是很从配置文件实例化一个server密切相关的，只是中间因为需要接管标准输出多加入了一步。

#### 启动时间计算：
最后这一步就是在命令行中启动tomcat最常见的内容了，整个服务的启动，开销的时间：
```java
long t2 = System.nanoTime();
if(log.isInfoEnabled()) {
    log.info("Initialization processed in " + ((t2 - t1) / 1000000) + " ms");
}
```

至此，tomcat的服务启动过程就结束了。

### load代码实现细节：
上述整个流程从启动脚本到内部实现，总体上完成了对tomcat服务启动的描述，但是其中包含很多的细节还缺少，导致很多内容并没出现在启动分析中。
最重要的server的实例生成依赖于对xml的解析，使用的工具是：
[commons-digester](https://commons.apache.org/proper/commons-digester/)。

#### 脚本启动和关闭的是同一个服务实例：
我们在使用脚本的时候，可以在任意时候对当前的tomcat实例进行开启和关闭，如何保证操作的是同一个实例？
我们使用脚本启动和关闭tomcat的时候，实际上最终都是执行bootstrap的main方法，正因为daemon是static的，所以，我们start和stop的时候，实际上操作的是同一个bootstrap对象，才能对同一个tomcat的启动和关闭。 

#### 配置文件解析和服务实例的生成详解：
再来回过头看看createStartDigester这个函数创建digester的具体实现：
```java
/**
* Create and configure the Digester we will be using for startup.
* @return the main digester to parse server.xml
*/
protected Digester createStartDigester() {
    long t1=System.currentTimeMillis();
    // Initialize the digester
    Digester digester = new Digester();
    digester.setValidating(false);
    digester.setRulesValidation(true);
    Map<Class<?>, List<String>> fakeAttributes = new HashMap<>();
    List<String> attrs = new ArrayList<>();
    attrs.add("className");
    fakeAttributes.put(Object.class, attrs);
    digester.setFakeAttributes(fakeAttributes);
    digester.setUseContextClassLoader(true);

    // Configure the actions we will be using
    digester.addObjectCreate("Server",
                                "org.apache.catalina.core.StandardServer",
                                "className");
    digester.addSetProperties("Server");
    digester.addSetNext("Server",
                        "setServer",
                        "org.apache.catalina.Server");

    digester.addObjectCreate("Server/GlobalNamingResources",
                                "org.apache.catalina.deploy.NamingResourcesImpl");
    digester.addSetProperties("Server/GlobalNamingResources");
    digester.addSetNext("Server/GlobalNamingResources",
                        "setGlobalNamingResources",
                        "org.apache.catalina.deploy.NamingResourcesImpl");

    digester.addObjectCreate("Server/Listener",
                                null, // MUST be specified in the element
                                "className");
    digester.addSetProperties("Server/Listener");
    digester.addSetNext("Server/Listener",
                        "addLifecycleListener",
                        "org.apache.catalina.LifecycleListener");

    digester.addObjectCreate("Server/Service",
                                "org.apache.catalina.core.StandardService",
                                "className");
    digester.addSetProperties("Server/Service");
    digester.addSetNext("Server/Service",
                        "addService",
                        "org.apache.catalina.Service");

    digester.addObjectCreate("Server/Service/Listener",
                                null, // MUST be specified in the element
                                "className");
    digester.addSetProperties("Server/Service/Listener");
    digester.addSetNext("Server/Service/Listener",
                        "addLifecycleListener",
                        "org.apache.catalina.LifecycleListener");

    //Executor
    digester.addObjectCreate("Server/Service/Executor",
                        "org.apache.catalina.core.StandardThreadExecutor",
                        "className");
    digester.addSetProperties("Server/Service/Executor");

    digester.addSetNext("Server/Service/Executor",
                        "addExecutor",
                        "org.apache.catalina.Executor");


    digester.addRule("Server/Service/Connector",
                        new ConnectorCreateRule());
    digester.addRule("Server/Service/Connector", new SetAllPropertiesRule(
            new String[]{"executor", "sslImplementationName", "protocol"}));
    digester.addSetNext("Server/Service/Connector",
                        "addConnector",
                        "org.apache.catalina.connector.Connector");

    digester.addObjectCreate("Server/Service/Connector/SSLHostConfig",
                                "org.apache.tomcat.util.net.SSLHostConfig");
    digester.addSetProperties("Server/Service/Connector/SSLHostConfig");
    digester.addSetNext("Server/Service/Connector/SSLHostConfig",
            "addSslHostConfig",
            "org.apache.tomcat.util.net.SSLHostConfig");

    digester.addRule("Server/Service/Connector/SSLHostConfig/Certificate",
                        new CertificateCreateRule());
    digester.addRule("Server/Service/Connector/SSLHostConfig/Certificate",
                        new SetAllPropertiesRule(new String[]{"type"}));
    digester.addSetNext("Server/Service/Connector/SSLHostConfig/Certificate",
                        "addCertificate",
                        "org.apache.tomcat.util.net.SSLHostConfigCertificate");

    digester.addObjectCreate("Server/Service/Connector/Listener",
                                null, // MUST be specified in the element
                                "className");
    digester.addSetProperties("Server/Service/Connector/Listener");
    digester.addSetNext("Server/Service/Connector/Listener",
                        "addLifecycleListener",
                        "org.apache.catalina.LifecycleListener");

    digester.addObjectCreate("Server/Service/Connector/UpgradeProtocol",
                                null, // MUST be specified in the element
                                "className");
    digester.addSetProperties("Server/Service/Connector/UpgradeProtocol");
    digester.addSetNext("Server/Service/Connector/UpgradeProtocol",
                        "addUpgradeProtocol",
                        "org.apache.coyote.UpgradeProtocol");

    // Add RuleSets for nested elements
    digester.addRuleSet(new NamingRuleSet("Server/GlobalNamingResources/"));
    digester.addRuleSet(new EngineRuleSet("Server/Service/"));
    digester.addRuleSet(new HostRuleSet("Server/Service/Engine/"));
    digester.addRuleSet(new ContextRuleSet("Server/Service/Engine/Host/"));
    addClusterRuleSet(digester, "Server/Service/Engine/Host/Cluster/");
    digester.addRuleSet(new NamingRuleSet("Server/Service/Engine/Host/Context/"));

    // When the 'engine' is found, set the parentClassLoader.
    digester.addRule("Server/Service/Engine",
                        new SetParentClassLoaderRule(parentClassLoader));
    addClusterRuleSet(digester, "Server/Service/Engine/Cluster/");

    long t2=System.currentTimeMillis();
    if (log.isDebugEnabled()) {
        log.debug("Digester for server.xml created " + ( t2-t1 ));
    }
    return digester;

}
```
其中add开头的方法就是在给 Digester 配置一些规则：
```java
//创建一个创建server对象的规则
digester.addObjectCreate("Server","org.apache.catalina.core.StandardServer","className");
//创建一个设置属性的规则，那么创建完了server对象之后就会设置配置文件后面的属性
digester.addSetProperties("Server");
//当server元素执行完了之后，执行的操作。。。这里其实是调用前面那个对象的方法
digester.addSetNext("Server","setServer","org.apache.catalina.Server");

digester.addObjectCreate("Server/Listener",null,"className");
digester.addSetProperties("Server/Listener");
digester.addSetNext("Server/Listener","addLifecycleListener","org.apache.catalina.LifecycleListener");

digester.addObjectCreate("Server/Service","org.apache.catalina.core.StandardService","className");
digester.addSetProperties("Server/Service");
digester.addSetNext("Server/Service","addService","org.apache.catalina.Service");

digester.addObjectCreate("Server/Service/Listener",null,"className");
digester.addSetProperties("Server/Service/Listener");
digester.addSetNext("Server/Service/Listener","addLifecycleListener","org.apache.catalina.LifecycleListener");
```
有了这些规则之后，在执行parse的过程中，就会根据这些规则来做一些操作。比如，解析到server.xml中的Server节点的时候:
```xml
<Server port="8005" shutdown="SHUTDOWN">
  ...
  <Service name="Catalina">
  ...
  ...
```
就会根据上面的规则，首先通过代码关联创建一个org.apache.catalina.core.StandardService对象：
```java
digester.addObjectCreate("Server",
                            "org.apache.catalina.core.StandardServer",
                            "className");
digester.addSetProperties("Server");
digester.addSetNext("Server",
                    "setServer",
                    "org.apache.catalina.Server");
```
并且调用当前的setServer，将生成的Server赋值给当前类的server变量。
然后使用如下代码：
```java
digester.addObjectCreate("Server/Service",
                            "org.apache.catalina.core.StandardService",
                            "className");
digester.addSetProperties("Server/Service");
digester.addSetNext("Server/Service",
                    "addService",
                    "org.apache.catalina.Service");
```
调用 StandardServer 的 addService 方法将StandardService添加到StandardServer中。

总体上：**createStartDigester方法返回的是一个Digester类的实例，它的作用就是根据设置的一系列规则，解析config/server.xml文件，组织创建关系描述中所有具有相互依赖关系的对象。**

其他的组件也是按类似的方式完成创建和关联的。这样经过对xml文件的解析将会产生：
```java
org.apache.catalina.core.StandardServer
org.apache.catalina.core.StandardService
org.apache.catalina.connector.Connector
org.apache.catalina.core.StandardEngine
org.apache.catalina.core.StandardHost
org.apache.catalina.core.StandardContext
```
等等一系列对象，这些对象从前到后前一个包含后一个对象的引用。


参考：
[【Tomcat学习笔记】4-启动流程分析](https://fdx321.github.io/2017/05/18/%E3%80%90Tomcat%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E3%80%914-%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90/)
[Apache Commons Digester 一 （基础内容、核心API）](http://www.cnblogs.com/chenpi/p/6930730.html)
[Apache Commons Digester 二（规则模块绑定-RulesModule、异步解析-asyncParse、xml变量Substitutor、带参构造方法）](http://www.cnblogs.com/chenpi/p/6947441.html)
[Apache Commons Digester 三（规则注解）](http://www.cnblogs.com/chenpi/p/6959691.html)
[XML 解析（二） JDOM, DOM4J,Digester](http://goalietang.iteye.com/blog/2030255)
[Tomcat源码的catalina中利用Digester解析conf/server.xml](http://blog.csdn.net/wgw335363240/article/details/5869660)


#### 实例化的server启动：
既然通过配置文件创建了需要启动的server和service，那么回到 getServer().init() 这里看看是如何启动的。

解析完xml之后关闭文件流，接着设置StandardServer对象（该对象在上面解析xml时）的catalina的引用为当前对象。接下来将调用StandardServer对象的init方法。

也就是说，现在的Server是StandardServer，那么执行的init方法也只StandardServer中的init方法，但是这个类中没有这个方法，那么这个调用就会去找他的父类的init方法：
```java
public final class StandardServer extends LifecycleMBeanBase implements Server 
```
而LifecycleMBeanBase也没有这个方法，继续向上找其父类的init方法，也就是LifecycleBase的init方法：
```java
@Override
public final synchronized void init() throws LifecycleException {
    if (!state.equals(LifecycleState.NEW)) {
        invalidTransition(Lifecycle.BEFORE_INIT_EVENT);
    }

    try {
        setStateInternal(LifecycleState.INITIALIZING, null, false);
        initInternal();
        setStateInternal(LifecycleState.INITIALIZED, null, false);
    } catch (Throwable t) {
        handleSubClassException(t, "lifecycleBase.initFail", toString());
    }
}
```
调用initInternal方法，这儿需要注意，因为实际上还是从StandardServer进行调用的，所以这儿的initInternal方法实际上执行的是StandardServer中的initInternal方法：
```java
/**
* Invoke a pre-startup initialization. This is used to allow connectors
* to bind to restricted ports under Unix operating environments.
*/
@Override
protected void initInternal() throws LifecycleException {

    super.initInternal();

    // Register global String cache
    // Note although the cache is global, if there are multiple Servers
    // present in the JVM (may happen when embedding) then the same cache
    // will be registered under multiple names
    onameStringCache = register(new StringCache(), "type=StringCache");

    // Register the MBeanFactory
    MBeanFactory factory = new MBeanFactory();
    factory.setContainer(this);
    onameMBeanFactory = register(factory, "type=MBeanFactory");

    // Register the naming resources
    globalNamingResources.init();

    // Populate the extension validator with JARs from common and shared
    // class loaders
    if (getCatalina() != null) {
        ClassLoader cl = getCatalina().getParentClassLoader();
        // Walk the class loader hierarchy. Stop at the system class loader.
        // This will add the shared (if present) and common class loaders
        while (cl != null && cl != ClassLoader.getSystemClassLoader()) {
            if (cl instanceof URLClassLoader) {
                URL[] urls = ((URLClassLoader) cl).getURLs();
                for (URL url : urls) {
                    if (url.getProtocol().equals("file")) {
                        try {
                            File f = new File (url.toURI());
                            if (f.isFile() &&
                                    f.getName().endsWith(".jar")) {
                                ExtensionValidator.addSystemResource(f);
                            }
                        } catch (URISyntaxException e) {
                            // Ignore
                        } catch (IOException e) {
                            // Ignore
                        }
                    }
                }
            }
            cl = cl.getParent();
        }
    }
    // Initialize our defined Services
    for (int i = 0; i < services.length; i++) {
        services[i].init();
    }
}
```
在开始执行的时候，首先执行的是：
```java
super.initInternal();
```
也就是StandardServer的父类的initInternal方法，也就是LifecycleMBeanBase中的initInternal方法。

总体上，StandardServer.initInternal()方法中主要完成的功能有：
> - 调用父类LifecycleMBeanBase.initInternal方法进行注册MBean
> - 从低向上逐级验证tomcat类加载器
> - 使用循环逐个初始化service（在解析serverx.xml的时候已经实例化StandardService并调用StandardServer.addService()添加到StandardServer.services变量中）。在标准server.xml配置中只有一个service——StandardService，所以就是只调用StandardService.init()这一个service的方法了。

需要注意的是这儿完成的调用流程：
调用getServer().init()方法后，会进入 LifecycleBase#init，这个方法主要是设置生命周期以及触发相应的事件，之后会调用 StandardServer#init()，它首先会调用 LifecycleMBeanBase#init 把自己注册到MBeanServer中(JMX后面会具体说)，然后完成 StandardServer 自己初始化需要做的事情，最后在遍历数组，依次调用各个service的init方法。

下面我们仔细看一下StandardServer的initInternal方法调用的细节。

##### super.initInternal()流程分析：
每次执行Server的init调用的时候都会进入到父类的这个方法中进行执行，这个方法调用父类LifecycleMBeanBase的initInternal()方法，具体方法实现为：
```java
@Override
protected void initInternal() throws LifecycleException {

    // If oname is not null then registration has already happened via
    // preRegister().
    if (oname == null) {
        mserver = Registry.getRegistry(null, null).getMBeanServer();
        oname = register(this, getObjectNameKeyProperties());
    }
}
```
LifecycleMBeanBase这个类，是Tomcat提供的对MBeanRegistration的抽象实现类，运用抽象模板模式将所有容器统一注册到JMX。
所以在StandardServer中调用这个方法，就实现了将StandardServer类的实例注册到JMX的服务器的过程。
关于JMX的更多内容，后续章节再进行补充。

在StandardServer的initInternal方法中，后续的调用：
```java
// Register global String cache
// Note although the cache is global, if there are multiple Servers
// present in the JVM (may happen when embedding) then the same cache
// will be registered under multiple names
onameStringCache = register(new StringCache(), "type=StringCache");

// Register the MBeanFactory
MBeanFactory factory = new MBeanFactory();
factory.setContainer(this);
onameMBeanFactory = register(factory, "type=MBeanFactory");
```
也是在对JMX进行注册。

参考：
> - [Tomcat源码分析-JMX之Registry类（中）](http://blog.csdn.net/wojiushiwo945you/article/details/73648405)
> - [TOMCAT源码分析——生命周期管理](http://www.cnblogs.com/jiaan-geng/p/4864501.html)


##### globalNamingResources.init()流程分析：
globalNamingResources这个变量是类NamingResourcesImpl的一个实例，这个类也是继承自LifecycleMBeanBase，所以这儿的init方法调用，做的工作和上述super.initInternal()方法中的方式一样，功能也是一样。
GlobalNamingResources是支持JNDI资源配置的类，具体的作用后续再补充。

##### 遍历类加载器，对加载jar包校验：
```java
if (getCatalina() != null) {
    ClassLoader cl = getCatalina().getParentClassLoader();
    // Walk the class loader hierarchy. Stop at the system class loader.
    // This will add the shared (if present) and common class loaders
    while (cl != null && cl != ClassLoader.getSystemClassLoader()) {
        if (cl instanceof URLClassLoader) {
            URL[] urls = ((URLClassLoader) cl).getURLs();
            for (URL url : urls) {
                if (url.getProtocol().equals("file")) {
                    try {
                        File f = new File (url.toURI());
                        if (f.isFile() && f.getName().endsWith(".jar")) {
                            ExtensionValidator.addSystemResource(f);
                        }
                    } catch (URISyntaxException e) {
                        // Ignore
                    } catch (IOException e) {
                        // Ignore
                    }
                }
            }
        }
        cl = cl.getParent();
    }
}
```
从 Catalina 的 parentClassLoader 开始，向上一直遍历到 ExtClassLoader，把它们会加载的 jar 包都用ExtensionValidator记录下来，后面再 StandardContext 启动的时候，会用 ExtensionValidator 来校验 StandardContext 对应的 Web App 依赖的一些 jar 包是否已经被加进来了。

从common ClassLoader开始往上查看，直到SystemClassLoader，遍历各个classLoader对应的查看路径，找到jar结尾的文件，读取Manifest信息，加入到ExtensionValidator#containerManifestResources属性中。

参考：
[【Tomcat学习笔记】7-分析各个组件的init和start](https://fdx321.github.io/2017/05/21/%E3%80%90Tomcat%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E3%80%917-%E5%88%86%E6%9E%90%E5%90%84%E4%B8%AA%E7%BB%84%E4%BB%B6%E7%9A%84init%E5%92%8Cstart/)

##### 对Service的初始化调用：
```java
// Initialize our defined Services
for (int i = 0; i < services.length; i++) {
    services[i].init();
}
```
完成上述操作之后，就可以对当前Server下的多个service进行init调用了。
对于Service调用init的过程在后续章节中继续。


##### 编写测试程序模拟getServer().init()调用流程：
首先，编写Lifecycle接口：
```java
public interface Lifecycle {
    public void init() throws Exception;
}
```
然后，编写一个抽象类LifecycleBase来实现这个接口：
```java
public abstract class LifecycleBase implements Lifecycle {

    @Override
    public void init() throws Exception {
        System.out.println("LifecycleBase init start");
        initInternal();
        System.out.println("LifecycleBase init end");
    }


    protected abstract void initInternal() throws Exception;
}
```
接着，再编写抽象类LifecycleMBeanBase来继承上述抽象类：
```java
public abstract class LifecycleMBeanBase extends LifecycleBase{
    @Override
    protected void initInternal() throws Exception {
        System.out.println("in LifecycleMBeanBase initInternal");
    }
}
```
最后，编写一个不允许被继承的类StandardServer，来继承LifecycleMBeanBase：
```java
public final class StandardServer extends LifecycleMBeanBase{
    @Override
    protected void initInternal() throws Exception {
        super.initInternal();
        System.out.println("in StandardServer initInternal");
    }
}
```
上述代码就完成了tomcat中，从Boostrap中使用反射调用Catalina的load函数中，getServer().init()的函数调用流程。
现在我们来测试一下：
```java
public class CallerTest {
    public static void main(String[] args){
        StandardServer standardServer = new StandardServer();
        try {
            standardServer.init();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```
最后返回的结果为：
```shell
LifecycleBase init start
in LifecycleMBeanBase initInternal
in StandardServer initInternal
LifecycleBase init end
```
通过上述简化，可以非常明白的看懂，在tomcat代码中的整个调用流程是如何进行的。

这个可以理解为new出来的对象保存了实际上这个对象的“指针”，通过这个“指针”进行的调用按照继承的方式进行：
```java
public class CallerTest {
    public static void main(String[] args){
        try {
            StandardServer standardServer = new StandardServer();
            standardServer.init();

            LifecycleMBeanBase lserver = new StandardServer();
            lserver.init();

            Lifecycle lcserver = new StandardServer();
            lcserver.init();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```
可以看到返回的结果是还是一样的：
```shell
LifecycleBase init start
in LifecycleMBeanBase initInternal
in StandardServer initInternal
LifecycleBase init end
LifecycleBase init start
in LifecycleMBeanBase initInternal
in StandardServer initInternal
LifecycleBase init end
LifecycleBase init start
in LifecycleMBeanBase initInternal
in StandardServer initInternal
LifecycleBase init end
```

参考：
> - [Tomcat启动部署](http://www.jianshu.com/p/150bb9bffab9)
> - [tomcat 7 源码分析-4 server初始化背后getServer().init()](http://smartvessel.iteye.com/blog/716492)


##### 完整的Server.init()调用流程：
将上述整体业务流程代码简化为：
```java
Digester digester = createStartDigester();
inputSource.setByteStream(inputStream);
digester.push(this);
digester.parse(inputSource);
getServer().setCatalina(this);
getServer().init();
```

然后这个getServer().init()调用的完整流程图为：
![tupian](tomcat_StandardService_init()调用.png)

参考：
> - [How Tomcat works — 三、tomcat启动（2）](http://www.cnblogs.com/sunshine-2015/p/5745868.html)

#### Service.init()的调用：
继续上述Server.init()调用中，最后调用的是Service的init方法。

各个Service调用init方法，总体上完成的主要功能有：
> - 调用父类LifecycleMBeanBase.initInternal方法进行注册MBeang
> - 如果container（这里的container是StandardEngine）不是null，则调用container的init方法进行初始化
> - 如果有Executor则逐个初始化
> - 最后使用循环逐个在初始化Connector，这里connector有两个，分别是用来处理两种协议：http和ajp

而通过默认server.xml配置生成的Service是StandardService的一个实例。
分析对StandardService的init()的调用和上述Server.init()的调用是一样的，都是通过父类来完成了中转调用：
StandardService中也没有init方法，只能找其父类，也就是LifecycleMBeanBase的init方法，也没有，继续向上找其父类，也就是LifecycleBase的init方法，有这个方法，其中又调用了StandardService自己的initInternal()方法。

我们来看看在StandardService中调用这个initInternal()方法实现：
```java
/**
* Invoke a pre-startup initialization. This is used to allow connectors
* to bind to restricted ports under Unix operating environments.
*/
@Override
protected void initInternal() throws LifecycleException {

    super.initInternal();

    if (engine != null) {
        engine.init();
    }

    // Initialize any Executors
    for (Executor executor : findExecutors()) {
        if (executor instanceof JmxEnabled) {
            ((JmxEnabled) executor).setDomain(getDomain());
        }
        executor.init();
    }

    // Initialize mapper listener
    mapperListener.init();

    // Initialize our defined Connectors
    synchronized (connectorsLock) {
        for (Connector connector : connectors) {
            connector.init();
        }
    }
}
```
在这儿，Connector第一次完整的出现了。

Service接口的调用在整个tomcat中非常重要，因为涉及到的就是服务启动的细节内容了。
下面我们分小节仔细看一下整个过程。

##### Service的构成：
Service 只是在 Connector 和 Container 外面多包一层，把它们组装在一起，向外面提供服务，一个 Service 可以设置多个 Connector，但是只能有一个 Container 容器。
我们看看Service这个接口的定义：
```java
package org.apache.catalina;
import java.io.File;
import org.apache.catalina.deploy.NamingResourcesImpl;
import org.apache.catalina.startup.Catalina;
public interface Server extends Lifecycle {
    // ------------------------------------------------------------- Properties
    public NamingResourcesImpl getGlobalNamingResources();
    public void setGlobalNamingResources
        (NamingResourcesImpl globalNamingResources);
    public javax.naming.Context getGlobalNamingContext();
    public int getPort();
    public void setPort(int port);
    public String getAddress();
    public void setAddress(String address);
    public String getShutdown();
    public void setShutdown(String shutdown);
    public ClassLoader getParentClassLoader();
    public void setParentClassLoader(ClassLoader parent);
    public Catalina getCatalina();
    public void setCatalina(Catalina catalina);
    public File getCatalinaBase();
    public void setCatalinaBase(File catalinaBase);
    public File getCatalinaHome();
    public void setCatalinaHome(File catalinaHome);
    // --------------------------------------------------------- Public Methods
    public void addService(Service service);
    public void await();
    public Service findService(String name);
    public Service[] findServices();
    public void removeService(Service service);
    public Object getNamingToken();
}
```

##### 父类的initInternal()调用：
```java
super.initInternal();
```
执行的是父类LifecycleMBeanBase的initInternal方法，和Server中调用的一样，这个函数的作用是用来完成注册。

##### Engine实例和Service的绑定：
代码中的engine是已经被初始化的，要找到是在哪里初始化的。
一切的来源都是从Catalina中的createStartDigester中进行创建的，先看看其中的内容：
```java
digester.addRuleSet(new EngineRuleSet("Server/Service/"));   //engine元素的定义的处理，这里主要是创建eingie
```
这儿将当前的Service和EngineRuleSet绑定。

digester对server.xml设置的标签动作有5种调用：
> - addObjectCreate：遇到起始标签的元素，初始化一个实例对象入栈
> - addSetProperties：遇到某个属性名，使用setter来赋值
> - addSetNext：遇到结束标签的元素，调用相应的方法
> - addRule：调用rule的begin 、body、end、finish方法来解析xml，入栈和出栈给对象赋值
> - addRuleSet：调用addRuleInstances来解析xml标签

根据上述规则，上述代码就是使用"Server/Service/"作为prefix，生成一个EngineRuleSet实例，然后调用这个实例的addRuleInstances方法。
我们进入EngineRuleSet看看：
```java
package org.apache.catalina.startup;
import org.apache.tomcat.util.digester.Digester;
import org.apache.tomcat.util.digester.RuleSetBase;
public class EngineRuleSet extends RuleSetBase {
    // ----------------------------------------------------- Instance Variables
    protected final String prefix;
    // ------------------------------------------------------------ Constructor
    public EngineRuleSet() {
        this("");
    }
    public EngineRuleSet(String prefix) {
        this.namespaceURI = null;
        this.prefix = prefix;
    }
    // --------------------------------------------------------- Public Methods
    @Override
    public void addRuleInstances(Digester digester) {
        digester.addObjectCreate(prefix + "Engine",
                                 "org.apache.catalina.core.StandardEngine",
                                 "className");
        digester.addSetProperties(prefix + "Engine");
        digester.addRule(prefix + "Engine",
                         new LifecycleListenerRule
                         ("org.apache.catalina.startup.EngineConfig",
                          "engineConfigClass"));
        digester.addSetNext(prefix + "Engine",
                            "setContainer",
                            "org.apache.catalina.Engine");
        //Cluster configuration start
        digester.addObjectCreate(prefix + "Engine/Cluster",
                                 null, // MUST be specified in the element
                                 "className");
        digester.addSetProperties(prefix + "Engine/Cluster");
        digester.addSetNext(prefix + "Engine/Cluster",
                            "setCluster",
                            "org.apache.catalina.Cluster");
        //Cluster configuration end
        digester.addObjectCreate(prefix + "Engine/Listener",
                                 null, // MUST be specified in the element
                                 "className");
        digester.addSetProperties(prefix + "Engine/Listener");
        digester.addSetNext(prefix + "Engine/Listener",
                            "addLifecycleListener",
                            "org.apache.catalina.LifecycleListener");
        digester.addRuleSet(new RealmRuleSet(prefix + "Engine/"));
        digester.addObjectCreate(prefix + "Engine/Valve",
                                 null, // MUST be specified in the element
                                 "className");
        digester.addSetProperties(prefix + "Engine/Valve");
        digester.addSetNext(prefix + "Engine/Valve",
                            "addValve",
                            "org.apache.catalina.Valve");
    }
}
```
其中和Service进行关联的主要设置为：
```java
digester.addObjectCreate(prefix + "Engine",
                            "org.apache.catalina.core.StandardEngine",
                            "className");
digester.addSetProperties(prefix + "Engine");
digester.addRule(prefix + "Engine",
                    new LifecycleListenerRule
                    ("org.apache.catalina.startup.EngineConfig",
                    "engineConfigClass"));
digester.addSetNext(prefix + "Engine",
                    "setContainer",
                    "org.apache.catalina.Engine");
```
其中，prefix是"Server/Service/"。
首先，使用addObjectCreate创建的对象，当遇到：
> - 名称为："Server/Service/Engine"
时创建一个：
> - 名称为："org.apache.catalina.core.StandardEngine"
的对象，并将其放在栈顶；
然后，根据：
> - 名称为："Server/Service/Engine"
元素的属性(attribute)，对刚创建的对象的属性(property)进行设置；
接着，调用addRule()方法，来注册一个特定的元素匹配模板以及相应的一个Rule类的实例；
最后，当再次遇到：
> - 名称为："Server/Service/Engine"
的元素时创建一个：
> - 名称为："org.apache.catalina.Engine"
的对象，并将其放在栈顶，同时调用第二栈顶元素(Service对象)的setContainer方法。

通过上述这些配置，就将StandardEngine这个类，通过调用Service的setContainer方法进行了绑定。
我们进入StandardService的setContainer方法看看具体实现：
```java
@Override
public void setContainer(Engine engine) {
    Engine oldEngine = this.engine;
    if (oldEngine != null) {
        oldEngine.setService(null);
    }
    this.engine = engine;
    if (this.engine != null) {
        this.engine.setService(this);
    }
    if (getState().isAvailable()) {
        if (this.engine != null) {
            try {
                this.engine.start();
            } catch (LifecycleException e) {
                log.warn(sm.getString("standardService.engine.startFailed"), e);
            }
        }
        // Restart MapperListener to pick up new engine.
        try {
            mapperListener.stop();
        } catch (LifecycleException e) {
            log.warn(sm.getString("standardService.mapperListener.stopFailed"), e);
        }
        try {
            mapperListener.start();
        } catch (LifecycleException e) {
            log.warn(sm.getString("standardService.mapperListener.startFailed"), e);
        }
        if (oldEngine != null) {
            try {
                oldEngine.stop();
            } catch (LifecycleException e) {
                log.warn(sm.getString("standardService.engine.stopFailed"), e);
            }
        }
    }

    // Report this property change to interested listeners
    support.firePropertyChange("container", oldEngine, this.engine);
}
```
首先，获取当前service的旧Engine，如果不为null，就将这个旧的Engine和当前的service解绑定；
然后，就用传入的这个新Engine，也就是StandardEngine覆盖当前service的engine，并且将engine和当前的service绑定；
接着，调用父父类LifecycleBase的getState，获取当前的Service的生命周期状态，如果当前的生命周期有效，就进入条件代码段进行配置：
（1）如果当前的engine不是null，就调用start启动，这个start是当前engine的父类的父类的父类，也即是LifecycleBase中的start方法；
（2）重启当前Service的MapperListener，将新的engine加载；
（3）停止旧的engine；
最后，调用support.firePropertyChange对当前的engine变更进行通知。

其中的support是PropertyChangeSupport的一个实例，用于属性变化的时候对变更进行通知。当bean的属性发生变化时，使用PropertyChangeSupport对象的firePropertyChange方法，它会将一个事件发送给所有已经注册的监听器。该方法有三个参数：属性的名字、旧的值以及新的值。属性的值必须是对象，如果是简单数据类型，则必须进行包装。
而对这个监听的注册是通过Service的函数addPropertyChangeListener进行的。

参考：
[使用PropertyChangeSupport来监听变量的变化](http://blog.csdn.net/yoier/article/details/25891077)

##### Engine的初始化：
StandardEngine和StandardService绑定之后，使用了setContainer进行调用，因为这个时候服务还在初始化阶段，没有执行start启动，所以getState().isAvailable()返回是false的，只进行engine变更的通知，然后返回。
继续看StandardEngine的initInternal中对Engine的初始化。
因为StandardEngine中没有实现init接口，一直向上找父类中的实现，只到LifecycleBase中的init方法：
```java
@Override
public final synchronized void init() throws LifecycleException {
    if (!state.equals(LifecycleState.NEW)) {
        invalidTransition(Lifecycle.BEFORE_INIT_EVENT);
    }

    try {
        setStateInternal(LifecycleState.INITIALIZING, null, false);
        initInternal();
        setStateInternal(LifecycleState.INITIALIZED, null, false);
    } catch (Throwable t) {
        handleSubClassException(t, "lifecycleBase.initFail", toString());
    }
}
```
对当前的生命周期做了设置，然后调用initInternal，这个时候返回到StandardEngine的initInternal方法：
```java
@Override
protected void initInternal() throws LifecycleException {
    // Ensure that a Realm is present before any attempt is made to start
    // one. This will create the default NullRealm if necessary.
    getRealm();
    super.initInternal();
}
```
除去getRealm这个调用，他的初始化很简单，就是调用父的内部初始化函数了，进入ContainerBase查看：
```java
@Override
protected void initInternal() throws LifecycleException {
    reconfigureStartStopExecutor(getStartStopThreadsInternal());
    super.initInternal();
}
```
继续向上，进入LifecycleMBeanBase查看：
```java
@Override
protected void initInternal() throws LifecycleException {

    // If oname is not null then registration has already happened via
    // preRegister().
    if (oname == null) {
        mserver = Registry.getRegistry(null, null).getMBeanServer();

        oname = register(this, getObjectNameKeyProperties());
    }
}
```
现在我们来梳理一下整个init过程。

###### StandardEngine调用getRealm：
所谓Realm，官方解释为：
A Realm is a "database" of usernames and passwords that identify valid users of a web application (or set of web applications), plus an enumeration of the list of roles associated with each valid user. You can think of roles as similar to groups in Unix-like operating systems, because access to specific web application resources is granted to all users possessing a particular role (rather than enumerating the list of associated usernames).

Realm（安全域）其实就是一个存储用户名和密码的“数据库”再加上一个枚举列表。“数据库”中的用户名和密码是用来验证 Web 应用（或 Web 应用集合）用户合法性的，而每一合法用户所对应的角色存储在枚举列表中。可以把这些角色看成是类似 UNIX 系统中的 group（分组），因为只有能够拥有特定角色的用户才能访问特定的 Web 应用资源（而不是通过对用户名列表进行枚举适配）。特定用户的用户名下可以配置多个角色。
虽然 Servlet 规范描述了一个可移植机制，使应用可以在 web.xml 部署描述符中声明它们的安全需求，但却没有提供一种可移植 API 来定义出 Servlet 容器与相应用户及角色信息的接口。然而，在很多情况下，非常适于将 Servlet 容器与一些已有的验证数据库或者生产环境中已存在的机制“连接”起来。因此，Tomcat 定义了一个 Java 接口（org.apache.catalina.Realm），通过“插入”组件来建立连接。
Tomcat提供了 6 种标准插件，支持与各种验证信息源的连接：
> - JDBCRealm——通过 JDBC 驱动器来访问保存在关系型数据库中的验证信息。
> - DataSourceRealm——访问保存在关系型数据库中的验证信息。
> - JNDIRealm——访问保存在 LDAP 目录服务器中的验证信息。
> - UserDatabaseRealm——访问存储在一个 UserDatabase JNDI 数据源中的认证信息，通常依赖一个 XML 文档（conf/tomcat-users.xml）。
> - MemoryRealm——访问存储在一个内存中对象集合中的认证信息，通过 XML 文档初始化（conf/tomcat-users.xml）。
> - JAASRealm——通过 Java 认证与授权服务（JAAS）架构来获取认证信息。

另外，还可以编写自定义 Realm 实现，将其整合到 Tomcat 中，只需这样做：
> - 实现 org.apache.catalina.Realm 接口。
> - 将编译好的 realm 放到 $CATALINA_HOME/lib 中。
> - 声明自定义 realm，具体方法详见“配置 Realm”一节。
> - 在 MBeans 描述符文件中声明自定义realm。


所以这个的getRealm调用就是在创建用户登录管理的内容，具体的细节后续再进行补充。

参考：
[Tomcat8权威指南 - Realm 配置](http://wiki.jikexueyuan.com/project/tomcat/realms-aaa.html)




###### ContainerBase调用reconfigureStartStopExecutor：
现在进入ContainerBase中的initInternal中，首先调用了reconfigureStartStopExecutor，然后再向上进行调用，看看这个函数：
```java
private void reconfigureStartStopExecutor(int threads) {
    if (threads == 1) {
        if (!(startStopExecutor instanceof InlineExecutorService)) {
            startStopExecutor = new InlineExecutorService();
        }
    } else {
        if (startStopExecutor instanceof ThreadPoolExecutor) {
            ((ThreadPoolExecutor) startStopExecutor).setMaximumPoolSize(threads);
            ((ThreadPoolExecutor) startStopExecutor).setCorePoolSize(threads);
        } else {
            BlockingQueue<Runnable> startStopQueue = new LinkedBlockingQueue<>();
            ThreadPoolExecutor tpe = new ThreadPoolExecutor(threads, threads, 10,
                    TimeUnit.SECONDS, startStopQueue,
                    new StartStopThreadFactory(getName() + "-startStop-"));
            tpe.allowCoreThreadTimeOut(true);
            startStopExecutor = tpe;
        }
    }
}
```
只是创建了一个ThreadPoolExecutor。

参考：
[tomcat的启动过程（Tomcat源码解析（三））](http://blog.csdn.net/jiaomingliang/article/details/47401275)

###### LifecycleMBeanBase执行initInternal：
这块的执行，之前就已经见过了，就是为了将当前的容器统一注册到JMX。

###### StandardEngine初始化小结：
总体上，就是完成了对容器的注册，并且生成了startStopExecutor的初始化。
StandardEngine#init竟然没有调用StandardHost#init方法，那么StandardHost的init方法是什么时候被调用的呢？
我们后续就能看到了。


##### Executor的初始化：
完成了Engine的初始化之后，Service继续开始Executor的初始化操作。
第一个问题：Executer从哪里来？
第二个问题：初始化完成了什么流程？

###### Executer生成：
代码中获取executer对象的方式是从函数findExecutors中来的，我们看看这个函数：
```java
@Override
public Executor[] findExecutors() {
    synchronized (executors) {
        Executor[] arr = new Executor[executors.size()];
        executors.toArray(arr);
        return arr;
    }
}
```
这个函数中，只是将一个容器转换为一个数组而已，说明Executor已经存在了。
还是在Catalina中的createStartDigester找吧：
```java
//Executor
digester.addObjectCreate("Server/Service/Executor",
                    "org.apache.catalina.core.StandardThreadExecutor",
                    "className");
digester.addSetProperties("Server/Service/Executor");

digester.addSetNext("Server/Service/Executor",
                    "addExecutor",
                    "org.apache.catalina.Executor");
```
是在这儿，将StandardThreadExecutor用addExecutor添加到StandardService中去的。
看看这个addExecutor方法：
```java
@Override
public void addExecutor(Executor ex) {
    synchronized (executors) {
        if (!executors.contains(ex)) {
            executors.add(ex);
            if (getState().isAvailable()) {
                try {
                    ex.start();
                } catch (LifecycleException x) {
                    log.error("Executor.start", x);
                }
            }
        }
    }
}
```
功能很简单，就是将当前的Executor实例添加到Service的Executor列表中。
同样的，会使用getState().isAvailable()检查当前Service的状态，确定是否对Executor启动。

###### Executer初始化：
完成了生成，在initInternal函数中，接着调用Executor的init函数了，进入StandardThreadExecutor，没有init函数，一直向上找，还是在LifecycleBase中的init函数。
非常熟悉的过程，就是设置当前的容器生命周期，然后调用子类的initInternal方法，查看StandardThreadExecutor的initInternal方法：
```java
@Override
protected void initInternal() throws LifecycleException {
    super.initInternal();
}
```
就是直接调用父类的initInternal函数，也就是LifecycleMBeanBase的initInternal方法，也是为了注册给JMX。

###### StandardThreadExecutor的作用
StandardThreadExecutor是Tomcat的标准的线程池实现，该类其实就是个配置服务类，其主要还是将配置传到到org.apache.tomcat.util.threads包中的核心类中进行。
标准执行器内在地使用了一个 java.util.concurrent.ThreadPoolExecutor。它通过具有一个变量大小的工作线程的线程池进行工作，一旦这些线程完成了一个任务，将会等待一个阻塞队列，直到一个新的任务进来。或者直到它等待了一个设定的时间，这时将会 "超时"，该线程将被关闭。这里边的关键点是第一个完成了一个任务的线程会首先被分配新的任务，线程池遵守一个先进先出(FIFO)的模式。在我们检查它将如何影响 Tomcat 执行器的时候我们需要时刻注意这一点。
我们再看看对这个类的注释：
```java
public class StandardThreadExecutor extends LifecycleMBeanBase
        implements Executor, ResizableExecutor {
    //默认线程的优先级
    protected int threadPriority = Thread.NORM_PRIORITY;
    //守护线程
    protected boolean daemon = true;
    //线程名称的前缀
    protected String namePrefix = "tomcat-exec-";
    //最大线程数默认200个
    protected int maxThreads = 200;
    //最小空闲线程25个
    protected int minSpareThreads = 25;
    //超时时间为6000
    protected int maxIdleTime = 60000;
    //线程池容器
    protected ThreadPoolExecutor executor = null;
    //线程池的名称
    protected String name;
     //是否提前启动线程
    protected boolean prestartminSpareThreads = false;
    //队列最大大小
    protected int maxQueueSize = Integer.MAX_VALUE;
    //为了避免在上下文停止之后，所有的线程在同一时间段被更新，所以进行线程的延迟操作
    protected long threadRenewalDelay = 1000L;
    //任务队列
    private TaskQueue taskqueue = null;

    //容器启动时进行,具体可参考org.apache.catalina.util.LifecycleBase#startInternal()
    @Override
    protected void startInternal() throws LifecycleException {
        //实例化任务队列
        taskqueue = new TaskQueue(maxQueueSize);
        //自定义的线程工厂类,实现了JDK的ThreadFactory接口
        TaskThreadFactory tf = new TaskThreadFactory(namePrefix,daemon,getThreadPriority());
        //这里的ThreadPoolExecutor是tomcat自定义的,不是JDK的ThreadPoolExecutor
        executor = new ThreadPoolExecutor(getMinSpareThreads(), getMaxThreads(), maxIdleTime, TimeUnit.MILLISECONDS,taskqueue, tf);
        executor.setThreadRenewalDelay(threadRenewalDelay);
        //是否提前启动线程，如果为true，则提前初始化minSpareThreads个的线程，放入线程池内
        if (prestartminSpareThreads) {
            executor.prestartAllCoreThreads();
        }
        //设置任务容器的父级线程池对象
        taskqueue.setParent(executor);
        //设置容器启动状态
        setState(LifecycleState.STARTING);
    }

  //容器停止时的生命周期方法,进行关闭线程池和资源清理
    @Override
    protected void stopInternal() throws LifecycleException {

        setState(LifecycleState.STOPPING);
        if ( executor != null ) executor.shutdownNow();
        executor = null;
        taskqueue = null;
    }

    //这个执行线程方法有超时的操作，参考org.apache.catalina.Executor接口
    @Override
    public void execute(Runnable command, long timeout, TimeUnit unit) {
        if ( executor != null ) {
            executor.execute(command,timeout,unit);
        } else { 
            throw new IllegalStateException("StandardThreadExecutor not started.");
        }
    }

    //JDK默认操作线程的方法,参考java.util.concurrent.Executor接口
    @Override
    public void execute(Runnable command) {
        if ( executor != null ) {
            try {
                executor.execute(command);
            } catch (RejectedExecutionException rx) {
                //there could have been contention around the queue
                if ( !( (TaskQueue) executor.getQueue()).force(command) ) throw new RejectedExecutionException("Work queue full.");
            }
        } else throw new IllegalStateException("StandardThreadPool not started.");
    }

    //由于继承了org.apache.tomcat.util.threads.ResizableExecutor接口，所以可以重新定义线程池的大小
    @Override
    public boolean resizePool(int corePoolSize, int maximumPoolSize) {
        if (executor == null)
            return false;

        executor.setCorePoolSize(corePoolSize);
        executor.setMaximumPoolSize(maximumPoolSize);
        return true;
    }
}
```
整个类的继承关系为：
![StandardThreadExecutor类继承关系](StandardThreadExecutor类继承关系.png)

为了分析功能作用，我们看看StandardThreadExecutor这个类的继承关系细节：
StandardThreadExecutor继承了LifecycleMBeanBase，并且实现了Executor和ResizableExecutor接口。

（1）ResizableExecutor接口：
```java
import java.util.concurrent.Executor;

public interface ResizableExecutor extends Executor {

    /**
     * Returns the current number of threads in the pool.
     *
     * @return the number of threads
     */
    public int getPoolSize();

    public int getMaxThreads();

    /**
     * Returns the approximate number of threads that are actively executing
     * tasks.
     *
     * @return the number of threads
     */
    public int getActiveCount();

    public boolean resizePool(int corePoolSize, int maximumPoolSize);

    public boolean resizeQueue(int capacity);
}
```
这个接口定义了多线程连接池的容量操作函数接口定义。需要注意的是这儿的Executor是JDK中的。
实现这个接口之后，就能动态改变线程池的大小和任务队列的大小了

（2）Ececutor接口：
```java
public interface Executor extends java.util.concurrent.Executor, Lifecycle {
    public String getName();
    void execute(Runnable command, long timeout, TimeUnit unit);
}
```
这个接口非常简单，就是定义了线程的名称和执行线程的入口函数。

（3）核心类：ThreadPoolExecutor
这个是和JDK中同名的自定义类，Tomcat的线程池文档中说：
org.apache.tomcat.util.threads.ThreadPoolExecutor.java和java.util.concurrent一样,但是它实现了一个高效的方法getSubmittedCount()方法用来处理工作队列。
我们看看这个自定义的重名线程池的实现：
```java
public class ThreadPoolExecutor extends java.util.concurrent.ThreadPoolExecutor {
    protected static final StringManager sm = StringManager
            .getManager("org.apache.tomcat.util.threads.res");
    /**
     *  已经提交但尚未完成的任务数量。
     *  这包括已经在队列中的任务和已经交给工作线程的任务但还未开始执行的任务
     *  这个数字总是大于getActiveCount()的
     **/
    private final AtomicInteger submittedCount = new AtomicInteger(0);
    private final AtomicLong lastContextStoppedTime = new AtomicLong(0L);
    /**
    *  最近的时间在ms时，一个线程决定杀死自己来避免
    *  潜在的内存泄漏。 用于调节线程的更新速率。
    */
    private final AtomicLong lastTimeThreadKilledItself = new AtomicLong(0L);
    //延迟2个线程之间的延迟。 如果为负，不更新线程。
    private long threadRenewalDelay = 1000L;
    //4个构造方法  ... 省略

    public long getThreadRenewalDelay() {
        return threadRenewalDelay;
    }

    public void setThreadRenewalDelay(long threadRenewalDelay) {
        this.threadRenewalDelay = threadRenewalDelay;
    }

    /**
    *  方法在完成给定Runnable的执行时调用。
    *  此方法由执行任务的线程调用。 如果
    *  非null，Throwable是未捕获的{@code RuntimeException}
    *  或{@code Error}，导致执行突然终止。...
    *  @param r 已完成的任务
    *  @param t 引起终止的异常，如果执行正常完成则为null
    **/
    @Override
    protected void afterExecute(Runnable r, Throwable t) {
        submittedCount.decrementAndGet();

        if (t == null) {
            stopCurrentThreadIfNeeded();
        }
    }

    //如果当前线程在上一次上下文停止之前启动，则抛出异常，以便停止当前线程。
    protected void stopCurrentThreadIfNeeded() {
        if (currentThreadShouldBeStopped()) {
            long lastTime = lastTimeThreadKilledItself.longValue();
            if (lastTime + threadRenewalDelay < System.currentTimeMillis()) {
                if (lastTimeThreadKilledItself.compareAndSet(lastTime,
                        System.currentTimeMillis() + 1)) {
                    // OK, it's really time to dispose of this thread

                    final String msg = sm.getString(
                                    "threadPoolExecutor.threadStoppedToAvoidPotentialLeak",
                                    Thread.currentThread().getName());

                    throw new StopPooledThreadException(msg);
                }
            }
        }
    }

    //当前线程是否需要被终止
    protected boolean currentThreadShouldBeStopped() {
        if (threadRenewalDelay >= 0
            && Thread.currentThread() instanceof TaskThread) {
            TaskThread currentTaskThread = (TaskThread) Thread.currentThread();
            //线程创建的时间<上下文停止的时间,则可以停止该线程
            if (currentTaskThread.getCreationTime() <
                    this.lastContextStoppedTime.longValue()) {
                return true;
            }
        }
        return false;
    }

    public int getSubmittedCount() {
        return submittedCount.get();
    }

    @Override
    public void execute(Runnable command) {
        execute(command,0,TimeUnit.MILLISECONDS);
    }

    public void execute(Runnable command, long timeout, TimeUnit unit) {
        submittedCount.incrementAndGet();
        try {
            super.execute(command);
        } catch (RejectedExecutionException rx) {
            if (super.getQueue() instanceof TaskQueue) {
                final TaskQueue queue = (TaskQueue)super.getQueue();
                try {
                    if (!queue.force(command, timeout, unit)) {
                        submittedCount.decrementAndGet();
                        throw new RejectedExecutionException("Queue capacity is full.");
                    }
                } catch (InterruptedException x) {
                    submittedCount.decrementAndGet();
                    Thread.interrupted();
                    throw new RejectedExecutionException(x);
                }
            } else {
                submittedCount.decrementAndGet();
                throw rx;
            }

        }
    }

    public void contextStopping() {
        this.lastContextStoppedTime.set(System.currentTimeMillis());
        int savedCorePoolSize = this.getCorePoolSize();
        TaskQueue taskQueue =
                getQueue() instanceof TaskQueue ? (TaskQueue) getQueue() : null;
        if (taskQueue != null) {
            taskQueue.setForcedRemainingCapacity(Integer.valueOf(0));
        }

        // setCorePoolSize(0) wakes idle threads
        this.setCorePoolSize(0);

        if (taskQueue != null) {
            // ok, restore the state of the queue and pool
            taskQueue.setForcedRemainingCapacity(null);
        }
        this.setCorePoolSize(savedCorePoolSize);
    }
}
```
需要注意到的是，这个线程池还是建立在JDK的ThreadPoolExecutor基础上的。
其中的getSubmittedCount方法，是通过AtomicInteger这个类来实现的。


参考文档：
[Tomcat线程池详解](http://www.jianshu.com/p/0b6eed03eb10)

（4）StandardThreadExecutor的作用小结：
通过上述分析，可以看到StandardThreadExecutor这个类给tomcat的所有组件提供了统一的线程池管理接口，最终线程池的实现是通过自定义的ThreadPoolExecutor来完成的。
这个标准线程池操作类在后续调优的过程中都是非常重要的。
后续再遇到线程池的时候，还会再对这块的内容进行补充的。


##### MapperListener的初始化：
我们先看看这个MapperListener的继承体系：
```java
public class MapperListener extends LifecycleMBeanBase implements ContainerListener, LifecycleListener
```
对LifecycleMBeanBase的继承，说明对象的启动会被注册到jmx上面去；
实现了ContainerListener和LifecycleListener，说明既可以响应container的事件，例如添加child，添加valve啥的，同时还可以响应组件的状态事件。

首先看看属性定义：
```java
private final Mapper mapper;  //service的mapper，用于对请求进行路由  
  
private final Service service;  //所属的service对象  
  
private static final StringManager sm =  
    StringManager.getManager(Constants.Package);  
  
private final String domain = null;  //注册在jmx哪个域名下面  
  
public MapperListener(Mapper mapper, Service service) {  
    this.mapper = mapper;  
    this.service = service;  
}
```





























































































































### Boostrap调用start方法启动服务器：
完成上述load之后，接下来在Boostrap中调用的就是start方法来启动整个服务了。


#### Engine的启动：
上述StandardEngine和StandardService分析中，会使用setContainer进行调用，如果getState().isAvailable()的时候，就会执行start操作，我们来详细看看LifecycleBase中的start方法的内容：
```java
@Override
public final synchronized void start() throws LifecycleException {

    if (LifecycleState.STARTING_PREP.equals(state) || LifecycleState.STARTING.equals(state) ||
            LifecycleState.STARTED.equals(state)) {

        if (log.isDebugEnabled()) {
            Exception e = new LifecycleException();
            log.debug(sm.getString("lifecycleBase.alreadyStarted", toString()), e);
        } else if (log.isInfoEnabled()) {
            log.info(sm.getString("lifecycleBase.alreadyStarted", toString()));
        }

        return;
    }

    if (state.equals(LifecycleState.NEW)) {
        init();
    } else if (state.equals(LifecycleState.FAILED)) {
        stop();
    } else if (!state.equals(LifecycleState.INITIALIZED) &&
            !state.equals(LifecycleState.STOPPED)) {
        invalidTransition(Lifecycle.BEFORE_START_EVENT);
    }

    try {
        setStateInternal(LifecycleState.STARTING_PREP, null, false);
        startInternal();
        if (state.equals(LifecycleState.FAILED)) {
            // This is a 'controlled' failure. The component put itself into the
            // FAILED state so call stop() to complete the clean-up.
            stop();
        } else if (!state.equals(LifecycleState.STARTING)) {
            // Shouldn't be necessary but acts as a check that sub-classes are
            // doing what they are supposed to.
            invalidTransition(Lifecycle.AFTER_START_EVENT);
        } else {
            setStateInternal(LifecycleState.STARTED, null, false);
        }
    } catch (Throwable t) {
        // This is an 'uncontrolled' failure so put the component into the
        // FAILED state and throw an exception.
        handleSubClassException(t, "lifecycleBase.startFail", toString());
    }
}
```
可以看到，这个过程中，会根据当前的LifecycleState做一些初识操作，然后调用StandardEngine的startInternal方法：
```java
@Override
protected synchronized void startInternal() throws LifecycleException {

    // Log our server identification information
    if(log.isInfoEnabled())
        log.info( "Starting Servlet Engine: " + ServerInfo.getServerInfo());

    // Standard container startup
    super.startInternal();
}
```
很简单，只是做一个日志，然后调用父类ContainerBase的startInternal方法：
```java
@Override
protected synchronized void startInternal() throws LifecycleException {

    // Start our subordinate components, if any
    logger = null;
    getLogger();
    Cluster cluster = getClusterInternal();
    if (cluster instanceof Lifecycle) {
        ((Lifecycle) cluster).start();
    }
    Realm realm = getRealmInternal();
    if (realm instanceof Lifecycle) {
        ((Lifecycle) realm).start();
    }

    // Start our child containers, if any
    Container children[] = findChildren();
    List<Future<Void>> results = new ArrayList<>();
    for (int i = 0; i < children.length; i++) {
        results.add(startStopExecutor.submit(new StartChild(children[i])));
    }

    boolean fail = false;
    for (Future<Void> result : results) {
        try {
            result.get();
        } catch (Exception e) {
            log.error(sm.getString("containerBase.threadedStartFailed"), e);
            fail = true;
        }

    }
    if (fail) {
        throw new LifecycleException(
                sm.getString("containerBase.threadedStartFailed"));
    }

    // Start the Valves in our pipeline (including the basic), if any
    if (pipeline instanceof Lifecycle)
        ((Lifecycle) pipeline).start();


    setState(LifecycleState.STARTING);

    // Start our thread
    threadStart();

}
```
所有的默认容器组件（StandardEngine、StandardHost、StandardContext、StandardWrapper）都会继承父类org.apache.catalina.core.ContainerBase，在这些容器组件启动时将会调用自己内部的startInternal方法，在该方法内部一般会调用父类的startInternal方法（StandardContext类的实现除外）。
我们来看看这个启动流程主要的执行内容：
（1）首先，是一个基本组件的检查和启动，包含：日志log，Cluster和Realm，这部分的组件具体内容，后续分析；
（2）然后，找到当前Engine的子容器，然后将这些子容器用StartChild类包装成Callable的类，接着使用线程池启动，主要代码就是：
```java
Container children[] = findChildren();
List<Future<Void>> results = new ArrayList<>();
for (int i = 0; i < children.length; i++) {
    results.add(startStopExecutor.submit(new StartChild(children[i])));
}

boolean fail = false;
for (Future<Void> result : results) {
    try {
        result.get();
    } catch (Exception e) {
        log.error(sm.getString("containerBase.threadedStartFailed"), e);
        fail = true;
    }

}
if (fail) {
    throw new LifecycleException(
            sm.getString("containerBase.threadedStartFailed"));
}
```
（3）调用管道的中 Valve 的 start 方法来启动管道；
（4）启动完成后将生命周期状态设置为 LifecycleState.STRATING 状态；
（5）启动后台线程定时处理一些事情。

其中第一步的组件启动中：
Cluster 用于配置集群，作用是同步session。
Realm 是 Tomcat 的安全域，可以用来管理资源的访问权限。

子容器使用 startStopExecutor 调用新线程来启动，可以提高效率。遍历future有两个作用：
> - 其get方法是阻塞的，保证管道Pipeline 启动前容器就已经启动完成。
> - 可以处理启动过程中遇到的异常。

启动子容器的线程类型是 StartChild 是一个实现了Callable 的的内部类在 call 方法中调用注入子类的 start 方法。
因为这里的 startInternal 是 ContainerBase 的方法，而所有的容器类都继承了 ContainerBase ，所以所有容器都会在启动过程中调用子类的 start 方法启动子容器。

参考文档：
[Tomcat源代码分析](https://my.oschina.net/liting/blog/422839)
[7.3 Container 分析](http://www.jianshu.com/p/bc38110622d3)
[ExecutorService的execute和submit方法](http://blog.csdn.net/peachpi/article/details/6771946)





并且，这段代码提示：engine对象在创建的时候会将parentClassLoader设置为sharLoader
```java
digester.addRule("Server/Service/Engine",  
                new SetParentClassLoaderRule(parentClassLoader));
```




还是要从Catalina中createStartDigester函数将Service和Server绑定的方式说起。
他调用了StandardServer的addService方法来绑定StandardService，我们看看这个方法：
```java
 @Override
public void addService(Service service) {

    service.setServer(this);

    synchronized (servicesLock) {
        Service results[] = new Service[services.length + 1];
        System.arraycopy(services, 0, results, 0, services.length);
        results[services.length] = service;
        services = results;

        if (getState().isAvailable()) {
            try {
                service.start();
            } catch (LifecycleException e) {
                // Ignore
            }
        }

        // Report this property change to interested listeners
        support.firePropertyChange("service", null, service);
    }
}
```
其中的 service.start() 完成了service的初始化设置。
但是StandardService中没有start方法，找其父类LifecycleMBeanBase也没有这个方法，继续向上查找，在LifecycleBase中有start方法，具体看看代码：
```java
@Override
public final synchronized void start() throws LifecycleException {

    if (LifecycleState.STARTING_PREP.equals(state) || LifecycleState.STARTING.equals(state) ||
            LifecycleState.STARTED.equals(state)) {

        if (log.isDebugEnabled()) {
            Exception e = new LifecycleException();
            log.debug(sm.getString("lifecycleBase.alreadyStarted", toString()), e);
        } else if (log.isInfoEnabled()) {
            log.info(sm.getString("lifecycleBase.alreadyStarted", toString()));
        }

        return;
    }

    if (state.equals(LifecycleState.NEW)) {
        init();
    } else if (state.equals(LifecycleState.FAILED)) {
        stop();
    } else if (!state.equals(LifecycleState.INITIALIZED) &&
            !state.equals(LifecycleState.STOPPED)) {
        invalidTransition(Lifecycle.BEFORE_START_EVENT);
    }

    try {
        setStateInternal(LifecycleState.STARTING_PREP, null, false);
        startInternal();
        if (state.equals(LifecycleState.FAILED)) {
            // This is a 'controlled' failure. The component put itself into the
            // FAILED state so call stop() to complete the clean-up.
            stop();
        } else if (!state.equals(LifecycleState.STARTING)) {
            // Shouldn't be necessary but acts as a check that sub-classes are
            // doing what they are supposed to.
            invalidTransition(Lifecycle.AFTER_START_EVENT);
        } else {
            setStateInternal(LifecycleState.STARTED, null, false);
        }
    } catch (Throwable t) {
        // This is an 'uncontrolled' failure so put the component into the
        // FAILED state and throw an exception.
        handleSubClassException(t, "lifecycleBase.startFail", toString());
    }
}
```
又调用到了startInternal这个方法，
















































































## Tomcat的总体设计框架：
上述章节中我们梳理了调用的具体代码，也忽略了不少的实现细节。

通过上述比较粗略的代码浏览，我们可以把握整个tomcat的组织和设计框架了，能够从总体上了解整个tomcat的主要实现功能是如何在这个启动流程中体现的。

### 总体结构：
在 《Tomcat的功能描述》 这一节中，我们从一般的思路出发去猜测一个web服务的提供者需要给出那些功能才能完成这个服务。
现在我们看看tomcat的总体结构来帮助我们结合代码来理解一下。

![Tomcat的总体结构](Tomcat的总体结构.gif)

从上图中可以看出 Tomcat 的心脏是两个组件：Connector 和 Container。
Connector 组件是可以被替换，这样可以提供给服务器设计者更多的选择，因为这个组件是如此重要，不仅跟服务器的设计的本身，而且和不同的应用场景也十分相关，所以一个 Container 可以选择对应多个 Connector。多个 Connector 和一个 Container 就形成了一个 Service，Service 的概念大家都很熟悉了，有了 Service 就可以对外提供服务了，但是 Service 还要一个生存的环境，必须要有人能够给她生命、掌握其生死大权，那就非 Server 莫属了。所以整个 Tomcat 的生命周期由 Server 控制。

#### 类和对应的概念：
Server：代表整个Catalina Servlet容器，可以包含一个或多个Service
Service：包含Connector和Container的集合，Service用适当的Connector接收用户的请求，再发给相应的Container来处理
Connector：实现某一协议的连接器，用来处理客户端发送来的协议，如默认的实现协议有HTTP、HTTPS、AJP
Engine：接到来自不同Connector请求，处理后将结果返回给Connector。Engine是一个逻辑容器
Host：虚拟主机，即域名或网络名，一个Engine可以包含多个Host。
Context：部署的具体Web应用的上下文，每个请求都在是相应的上下文里处理，如一个war包
Wrapper：对应定义的Servlet

由上可知，Catalina中有两个主要的模块：连接器（Connector）和容器（Container）

#### tomcat启动时序关系：
![tomcat启动时序图](tomcat启动时序图.jpg)


参考：
> - [Tomcat 系统架构与设计模式，第 1 部分 工作原理](https://www.ibm.com/developerworks/cn/java/j-lo-tomcat1/index.html)
> - [Tomcat 系统架构与设计模式，第 2 部分 设计模式分析](https://www.ibm.com/developerworks/cn/java/j-lo-tomcat2/index.html)
> - [Tomcat的启动分析](http://www.fanyilun.me/2016/10/10/Tomcat%E7%9A%84%E5%90%AF%E5%8A%A8%E5%88%86%E6%9E%90/)
























































## Tomcat的功能和实现细节：
现在我们在深入到之前忽略的细节中去，看看一些重要的功能的实现和组织，让整个图景更加的丰满，让自己的知识点更加的扎实。

### ContainerBase中的ContainerBackgroundProcessor：
Tomcat在启动完成后会有一个后台线程ContainerBackgroundProcessor，这个线程将会定时（默认为10秒）执行Engine、Host、Context、Wrapper各容器组件及与它们相关的其它组件的backgroundProcess方法，这段代码在所有容器组件的父类org.apache.catalina.core.ContainerBase类的backgroundProcess方法中，与自动加载类相关的代码在loader的backgroundProcess方法的调用时。
参考：
[Tomcat7自动加载类及检测文件变动原理](http://tyrion.iteye.com/blog/1947838)


### Tomcat容器的生命周期管理：
所有的对象都会经历从创建到消亡的过程。
Tomcat也有生命，在执行java Bootstrap命令的时候，tomcat类实例被创建，初始化后提供服务；当关闭tomcat服务窗口或者kill掉tomcat进程的时候，tomcat被消亡。
在这个过程中，被tomcat所管理的java web应用也会有新生和终结，这些对象的生命周期管理也交给tomcat进行管理了。

[tomcat学习之生命周期模型](https://my.oschina.net/psuyun/blog/298500)

#### 生命周期管理：LifeCycle接口
通过上述章节的代码可以看到，Server，Service都是继承了LifecycleMBeanBase，并且实现了Lifecycle和Service接口，Service接口又继承自Lifecycle接口。
也就是说，Tomcat的所有容器类都实现了统一的Lifecycle接口，由基类LifecycleBase提供统一的init方法来负责处理容器的状态，调用模板方法initInternal来处理各个容器自身所负责的内容。
我们先看看所有的容器都要实现的接口定义：
```java
package org.apache.catalina;
public interface Lifecycle {
    // ----------------------------------------------------- Manifest Constants
    public static final String BEFORE_INIT_EVENT = "before_init";
    public static final String AFTER_INIT_EVENT = "after_init";
    public static final String START_EVENT = "start";
    public static final String BEFORE_START_EVENT = "before_start";
    public static final String AFTER_START_EVENT = "after_start";
    public static final String STOP_EVENT = "stop";
    public static final String BEFORE_STOP_EVENT = "before_stop";
    public static final String AFTER_STOP_EVENT = "after_stop";
    public static final String AFTER_DESTROY_EVENT = "after_destroy";
    public static final String BEFORE_DESTROY_EVENT = "before_destroy";
    public static final String PERIODIC_EVENT = "periodic";
    public static final String CONFIGURE_START_EVENT = "configure_start";
    public static final String CONFIGURE_STOP_EVENT = "configure_stop";
    // --------------------------------------------------------- Public Methods
    public void addLifecycleListener(LifecycleListener listener);
    public LifecycleListener[] findLifecycleListeners();
    public void removeLifecycleListener(LifecycleListener listener);
    public void init() throws LifecycleException;
    public void start() throws LifecycleException;
    public void stop() throws LifecycleException;
    public void destroy() throws LifecycleException;
    public LifecycleState getState();
    public String getStateName();
    public interface SingleUse {
    }
}
```
上述代码使用正则对源代码去除多余内容：
```shell
行注释：\/\/[^\n]*
块注释：\/\*([^\*^\/]*|[\*^\/*]*|[^\**\/]*)*\*\/
去除空行：^[ ]*\r\n+
```

Lifecycle实现了一个观察者模式，而上面这些事件就代表了tomcat中各个组件生命周期中的各个事件，当这些事件发生的时候，会通知感兴趣的观察者（LifecycleListener）


参考：
> - [java匹配注释的正则表达式](http://www.cnblogs.com/xiziyin/archive/2012/01/25/2329350.html)


#### LifecycleMBeanBase的实现：
因为LifecycleMBeanBase实现了JmxEnabled这个接口，进入JmxEnabled查看：
```java
package org.apache.catalina;
import javax.management.MBeanRegistration;
import javax.management.ObjectName;
public interface JmxEnabled extends MBeanRegistration {
    String getDomain();
    void setDomain(String domain);
    ObjectName getObjectName();
}
```
继续找他的父类的定义为：
```java
package javax.management;
public interface MBeanRegistration   {
    public ObjectName preRegister(MBeanServer server,
                                  ObjectName name) throws java.lang.Exception;
    public void postRegister(Boolean registrationDone);
    public void preDeregister() throws java.lang.Exception ;
    public void postDeregister();
 }
```

JmxEnable是Java Managed Bean相关的功能。它也是个抽象类，实现了父类的initInternal()和destroyInternal()，这两个方法也很简单，就是对ManabedBean的注册和注销。

参考：
[Tomcat源码分析-LifecycleMBeanBase](http://blog.csdn.net/wojiushiwo945you/article/details/73331057)
[TOMCAT源码分析——生命周期管理](http://www.cnblogs.com/jiaan-geng/p/4864501.html)

#### Registry的作用：
Registry实现了单例模式。





## Tomcat编译脚本和整体呈现形式分析：
我们拿到tomcat的源代码，然后进行编译，最终得到的是一个目录结构，查看out/build这个目录下的结果为：
```shell
|-- bin
|   |-- bootstrap.jar
|   |-- catalina-tasks.xml
|   |-- catalina.bat
|   |-- catalina.sh
|   |-- commons-daemon-native.tar.gz
|   |-- commons-daemon.jar
|   |-- configtest.bat
|   |-- configtest.sh
|   |-- daemon.sh
|   |-- digest.bat
|   |-- digest.sh
|   |-- service.bat
|   |-- setclasspath.bat
|   |-- setclasspath.sh
|   |-- shutdown.bat
|   |-- shutdown.sh
|   |-- startup.bat
|   |-- startup.sh
|   |-- tomcat-juli.jar
|   |-- tomcat-native.tar.gz
|   |-- tool-wrapper.bat
|   |-- tool-wrapper.sh
|   |-- version.bat
|   `-- version.sh
|-- conf
|   |-- Catalina
|   |-- catalina.policy
|   |-- catalina.properties
|   |-- context.xml
|   |-- jaspic-providers.xml
|   |-- jaspic-providers.xsd
|   |-- logging.properties
|   |-- server.xml
|   |-- tomcat-users.xml
|   |-- tomcat-users.xsd
|   `-- web.xml
|-- lib
|   |-- annotations-api.jar
|   |-- catalina-ant.jar
|   |-- catalina-ha.jar
|   |-- catalina-storeconfig.jar
|   |-- catalina-tribes.jar
|   |-- catalina.jar
|   |-- ecj-4.6.3.jar
|   |-- el-api.jar
|   |-- jasper-el.jar
|   |-- jasper.jar
|   |-- jaspic-api.jar
|   |-- jsp-api.jar
|   |-- servlet-api.jar
|   |-- tomcat-api.jar
|   |-- tomcat-coyote.jar
|   |-- tomcat-dbcp.jar
|   |-- tomcat-i18n-es.jar
|   |-- tomcat-i18n-fr.jar
|   |-- tomcat-i18n-ja.jar
|   |-- tomcat-jdbc.jar
|   |-- tomcat-jni.jar
|   |-- tomcat-util-scan.jar
|   |-- tomcat-util.jar
|   |-- tomcat-websocket.jar
|   `-- websocket-api.jar
|-- logs
|   |-- catalina.2017-06-29.log
|   |-- host-manager.2017-06-29.log
|   |-- localhost.2017-06-29.log
|   |-- localhost_access_log.2017-06-29.txt
|   `-- manager.2017-06-29.log
|-- temp
`-- webapps
    |-- ROOT
    |-- docs
    |-- examples
    |-- host-manager
    `-- manager

12 directories, 64 files
```
可以看到生成了很多的jar包，但是不同的jar包位于不同的目录下，那么这个分配是根据什么进行安排的？
我们使用发编译打开这些jar包，能看到是源代码的不同package下的内容的打包，那么这些jar如何编译出来的？

带着上述问题，我们进入tomcat的编译构建环节看看。

### Ant构建脚本分析：




















