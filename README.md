# Gradle语法学习

[TOC]

## 一、概念

* `Gradle`核心是基于`Groovy`脚本语言，`Groovy`脚本基于`Java`且扩展了`Java`，因此`Gradle`需要依赖`JDK`和`Groovy`库；
* 和`ant`、`maven`构建有区别，`gradle`是一种编程思想；
* `Gradle`构建工具的出现让工程有无限可能。

## 二、配置

### 2.1 声明配置文件

在项目根目录下新建配置文件`config.gradle`:

```groovy
//添加多个自定义属性，可以通过ext代码块
ext {
    username = "Tian"

    //生产/开发环境（正式/测试）
    isRelease = true

    //建立Map存储，对象名、key都可以自定义，groovy糖果语法，非常灵活
    androidId = [
            compileSdkVersion: 30,
            buildToolsVersion: "30.0.0",
            minSdkVersion    : 21,
            targetSdkVersion : 30,
            versionCode      : 1,
            versionName      : "1.0"
    ]

    appId = [
            applicationId: "com.sty.ne.gradlelearning",
            library      : "com.sty.ne.mylibrary"
    ]

    //生产/开发环境URL
    url = [
            "debug"  : "https://11.22.33.44/debug",
            "release": "https://11.22.33.44/release"
    ]

    supportLibrary = "1.1.0"
    //第三方库
    dependencies = [
            "appcompat"   : "androidx.appcompat:appcompat:1.2.0",
            "recyclerview": "androidx.recyclerview:recyclerview:${supportLibrary}",
            "constraint"  : "androidx.constraintlayout:constraintlayout:2.0.2"
    ]
}
```

### 2.2 引用配置文件

在项目的`build.gradle`文件中引用配置文件：

```groovy
//根目录下的build.gradle头部加入自定义config.gradle，相当于layout布局中加入include
apply from: "config.gradle"

buildscript {
    repositories {
        google()
        jcenter()
    }
    dependencies {
        classpath "com.android.tools.build:gradle:4.0.1"
    }
}
//...
```

### 2.3 使用配置项

`app`的`build.gradle`文件：

```groovy
apply plugin: 'com.android.application'

println("hello gradle")
println "hello gradle"

println "${username}"
//正确的语法：${rootProject.ext.username}
//rootProject.ext.username = 1234 //弱类型语言Groovy
println "${rootProject.ext.username}"

//赋值与引用
def androidId = rootProject.ext.androidId
def appId = rootProject.ext.appId
def support = rootProject.ext.dependencies
def url = rootProject.ext.url
def isRelease = rootProject.ext.isRelease

android {
    compileSdkVersion androidId.compileSdkVersion
    buildToolsVersion androidId.buildToolsVersion

    defaultConfig {
        applicationId appId.applicationId
        minSdkVersion androidId.minSdkVersion
        targetSdkVersion androidId.targetSdkVersion
        versionCode androidId.versionCode
        versionName androidId.versionName

        testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"

        //开启分包
        multiDexEnabled true
        //设置分包配置
        //multiDexKeepFile file('multidex-config.txt')

        //将svg图片生成指定维度的png图片
        //vectorDrawables.generatedDensities('xhdpi', "xxhdpi")
        //使用support-v7兼容（5.0版本以上）
        vectorDrawables.useSupportLibrary = true
        //只保留指定和默认资源
        resConfigs('zh-rCN')

        //配置so库CPU架构（真机：arm，模拟器：x86）
        //x86 x86_64 mips mips64
        ndk {
            abiFilters('armeabi', 'armeabi-v7a')
            //为了模拟器启动
            //abiFilters('x86', "x86_64")
        }

        //源集 - 设置源集的属性，更改源集的Java目录或者自由目录等
        sourceSets {
            main {
                if(!isRelease) {
                    //如果是组件化模式，需要单独运行时
                    manifest.srcFile 'src/main/AndroidManifest.xml'
                    java.srcDirs = ['src/main/java']
                    res.srcDirs = ['src/main/res']
                    resources.srcDirs = ['src/main/resources']
                    aidl.srcDirs = ['src/main/aidl']
                    assets.srcDirs = ['src/main/assets']
                }else {
                    //集成化模式，整个项目打包
                    manifest.srcFile 'src/main/AndroidManifest.xml'
                }
            }
        }
    }

    //这个配置必须写在buildTypes的节点之前
    signingConfigs {
        debug {
            //天坑：填错了，编译不通过还找不到问题
            storeFile file('/Users/tian/.android/debug.keystore')
            storePassword "android"
            keyAlias "androiddebugkey"
            keyPassword "android"
        }
        release {
            //签名证书文件
            storeFile file('/Users/tian/NeCloud/ArchitectWorkspace/NeGradleLearning/keystore.jks')
            //签名证书的类型
            storeType ""
            //清明证书文件的密码
            storePassword "123456"
            //签名证书中秘钥别名
            keyAlias "key0"
            //签名证书中该秘钥的密码
            keyPassword "123456"
            //是否开启V2签名
            v2SigningEnabled true
        }
    }

    buildTypes {
        debug {
            buildConfigField("String", "url", "\"${url.debug}\"")
            //对构建类型设置签名信息
            signingConfig signingConfigs.debug
        }

        release {
            minifyEnabled false
            buildConfigField("String", "url", "\"${url.release}\"")
            //对构建类型设置签名信息
            signingConfig signingConfigs.release
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }

    //AdbOptions可以对adb操作选项添加配置
    adbOptions {
        //配置操作超时时间，单位毫秒
        timeOutInMs = 5 * 1000_0
        //adb install 命令的选项配置
        installOptions '-r', '-s'
    }

    //对dx操作的配置，接受一个DexOptions 类型的闭包，配置由DexOptions提供
    dexOptions {
        //配置执行dx命令是为其分配的最大堆内存
        javaMaxHeapSize "4g"
        //配置是否预执行 dex Libraries 工程，开启后会提高增量构建速度，不过会影响clean构建的速度，默认为true
        preDexLibraries = false
        //配置是否开启jumbo模式，代码方法数超过65535 需要强制开启才能构建成功
        jumboMode true
        //配置Gradle运行dx命令时使用的线程数量
        threadCount 8
        //配置multiDex参数
        additionalParameters = [
                '--multi-dex', //多dex分包
                '--set-max-idx-number=50000', //每个包内方法数上限
                //'--main-dex-list=' + '/multidex-config.txt', //打包到classes.dex的文件列表
                '--minimal-main-dex'
        ]
    }

    //执行 gradle lint命令即可运行lint检查，默认生成的报告在outputs/lint-results.html中
    lintOptions {
        //遇到lint检查错误会终止构建，一般设置为false
        abortOnError false
        //将警告当做错误来处理（老版本：warningAsErros）
        warningsAsErrors false
        //检查新API
        check "NewApi"
    }

}

dependencies {
    implementation fileTree(dir: "libs", include: ["*.jar"])
    //标准写法：
    //implementation group: 'androidx.appcompat', name: 'appcompat', version: '1.2.0'
    //implementation 'androidx.appcompat:appcompat:1.2.0'

//    implementation support.appcompat
//    implementation support.recyclerview
//    implementation support.constraint

    //获取第三方依赖库最简洁的方式：
    support.each { k, v -> implementation v}

    //依赖library
    implementation project(":mylibrary")
    //HashMap<String, String> map = new HashMap<>();
    //implementation map.get("appcompat")
}
```

