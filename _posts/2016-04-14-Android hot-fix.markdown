---
layout:     post
title:      "Android hot-fix热修复"
subtitle:   "内部交流分享"
date:       2016-04-14
author:     "Chenfeiyue"
header-img: "img/post-bg-android.jpg"
tags:
    - Android
    - 分享
    - Blog
---


# AndFix

---

项目地址：[https://github.com/alibaba/AndFix](https://github.com/alibaba/AndFix)


AndFix is a solution to fix the bugs online instead of redistributing Android App.
Andfix is an acronym for "Android hot-fix".
AndFix supports Android version from 2.3 to 6.0, both ARM and X86 architecture, both Dalvik and ART runtime.

The compressed file format of AndFix's patch is .apatch. It is dispatched from your own server to client to fix your App's bugs.

优点：实时修复，仅支持java层修改

缺点：不支持添加文件到assets文件夹
不支持layout文件添加组件
不支持add filed(R.xx.xx)， new class， 内部类， 匿名内部类

## Principle

The implementation principle of AndFix is method body's replacing,

具体的实现原理就是方法替换

![image](http://nulibj.github.io/src/andfix/images/principle.png)

### Method replacing

AndFix judges the methods should be replaced by java custom annotation and replaces it by hooking it. AndFix has a native method `art_replaceMethod` in ART or `dalvik_replaceMethod` in Dalvik. 

For more details, [here](https://github.com/alibaba/AndFix/tree/master/jni).

## Fix Process

![image](http://nulibj.github.io/src/andfix/images/process.png)

## Integration

### How to get?

Directly add AndFix aar to your project as compile libraries.

For your maven dependency,

```
<dependency>
	<groupId>com.alipay.euler</groupId>
	<artifactId>andfix</artifactId>
	<version>0.4.0</version>
	<type>aar</type>
</dependency>
```
For your gradle dependency,

```
dependencies {
	compile 'com.alipay.euler:andfix:0.4.0@aar'
}
```

### How to use?

1. Initialize PatchManager,

```
patchManager = new PatchManager(context);
patchManager.init(appversion);//current version
```

2. Load patch,

```
patchManager.loadPatch();
```

You should load patch as early as possible, generally, in the initialization phase of your application(such as `Application.onCreate()`).

3. Add patch,

```
patchManager.addPatch(path);//path of the patch file that was downloaded
```

When a new patch file has been downloaded, it will become effective immediately by `addPatch`.

## Developer Tool

AndFix provides a patch-making tool called **apkpatch**.

### How to get?

The `apkpatch` tool can be found [here](https://github.com/alibaba/AndFix/raw/master/tools/apkpatch-1.0.3.zip).

### How to use?

* Prepare two android packages, one is the online package, the other one is the package after you fix bugs by coding.

* Generate `.apatch` file by providing the two package,

```
usage: apkpatch -f <new> -t <old> -o <output> -k <keystore> -p <***> -a <alias> -e <***>
 -a,--alias <alias>     keystore entry alias.
 -e,--epassword <***>   keystore entry password.
 -f,--from <loc>        new Apk file path.
 -k,--keystore <loc>    keystore path.
 -n,--name <name>       patch name.
 -o,--out <dir>         output dir.
 -p,--kpassword <***>   keystore password.
 -t,--to <loc>          old Apk file path.
```

Now you get the application savior, the patch file. Then you need to dispatch it to your client in some way, push or pull.

Sometimes, your team members may fix each other's bugs, and generate not only one `.apatch`. For this situation, you can
merge `.apatch` files using this tool,

```
usage: apkpatch -m <apatch_path...> -o <output> -k <keystore> -p <***> -a <alias> -e <***>
 -a,--alias <alias>     keystore entry alias.
 -e,--epassword <***>   keystore entry password.
 -k,--keystore <loc>    keystore path.
 -m,--merge <loc...>    path of .apatch files.
 -n,--name <name>       patch name.
 -o,--out <dir>         output dir.
 -p,--kpassword <***>   keystore password.
```

## Running sample

1. Import samplesI/AndFixDemo to your IDE, append AndFixDemo dependencies with AndFix(library project or aar).
2. Build project, save the package as 1.apk, and then install on device/emulator.
3. Modify com.euler.test.A, references com.euler.test.Fix.
4. Build project, save the package as 2.apk.
5. Use apkpatch tool to make a patch.
6. Rename the patch file to out.apatch, and then copy it to sdcard.
7. Run 1.apk and view log.

## Notice

### ProGuard

If you enable ProGuard, you must save the mapping.txt, so your new version's build can use it with ["-applymapping"](http://proguard.sourceforge.net/manual/usage.html#applymapping).

And it is necessary to keep classes as follow,

* Native method

	com.alipay.euler.andfix.AndFix

* Annotation

	com.alipay.euler.andfix.annotation.MethodReplace

To ensure that these classes can be found after running an obfuscation and static analysis tool like ProGuard, add the configuration below to your ProGuard configuration file.


```
-keep class * extends java.lang.annotation.Annotation
-keepclasseswithmembernames class * {
    native <methods>;
}
```

### Self-Modifying Code

If you use it, such as *Bangcle*. To generate patch file, you'd better to use raw apk.

### Security

The following is important but out of AndFix's range.

-  verify the signature of patch file
-  verify the fingerprint of optimize file

## API Documentation

The libraries javadoc can be found [here](https://rawgit.com/alibaba/AndFix/master/docs/index.html).

## License

[Apache License, Version 2.0](http://www.apache.org/licenses/LICENSE-2.0.html)

Copyright (c) 2015, alipay.com


### 使用方法

### Server

Server端使用apkpatch生成差分补丁，后缀.apatch

./apkpatch.sh -f demo-debug2.apk -t demo-debug1.apk -o out -k demo.jks -p 123456 -a key -e 123456

### Client 

Application初始化AndFix组件，下载补丁，加载补丁，删除补丁

### 源码解析

Application.onCreate()初始化AndFix组件

```
private void initAndFix() {
    mPatchManager = new PatchManager(this); // 
    mPatchManager.init(ApkUtil.getVersionName(this)); // 根据版本号处理补丁文件的加载、删除等（暂时把files/apatch下的补丁添加到												mPatchManager.mPatchs集合里，没有加载）
    // load patch
    mPatchManager.loadPatch(); //加载补丁
    // add patch at runtime
    try {
        // .apatch file path
        String patchFileString = Environment.getExternalStorageDirectory()
                .getAbsolutePath() + APATCH_PATH;
        mPatchManager.addPatch(patchFileString);
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

### PatchManager.java

补丁文件的管理类（加载、删除）

```
private static final String SUFFIX = ".apatch"; // 补丁后缀
private static final String DIR = "apatch";   // 补丁路径 files/apatch
private static final String SP_NAME = "_andfix_"; // sharedpreferences文件名
private static final String SP_VERSION = "version";    
```

```
public PatchManager(Context context) {
    mContext = context;
    mAndFixManager = new AndFixManager(mContext); // AndFixManager
    mPatchDir = new File(mContext.getFilesDir(), DIR); // 补丁路径 files/apatch
    mPatchs = new ConcurrentSkipListSet<Patch>(); // 存放补丁信息的集合（同步高并发）
    mLoaders = new ConcurrentHashMap<String, ClassLoader>(); // classloaders
}
```

```
/**
 * initialize
 *
 * @param appVersion App version
 */
public void init(String appVersion) {
    if (!mPatchDir.exists() && !mPatchDir.mkdirs()) {// make directory fail
        Log.e(TAG, "patch dir create error.");
        return;
    } else if (!mPatchDir.isDirectory()) {// not directory
        mPatchDir.delete();
        return;
    }
    SharedPreferences sp = mContext.getSharedPreferences(SP_NAME,
            Context.MODE_PRIVATE);
    String ver = sp.getString(SP_VERSION, null);
    if (ver == null || !ver.equalsIgnoreCase(appVersion)) {
        cleanPatch();  // app版本升级后清空之前的历史补丁（新版已修复bug）
        sp.edit().putString(SP_VERSION, appVersion).commit();
    } else {
        initPatchs();
    }
}
```
```
private void initPatchs() {
    File[] files = mPatchDir.listFiles();
    for (File file : files) {
        addPatch(file);  // 加载mPatchDir下的appath补丁到mPatchs集合
    }
}
```
```
// 删除源文件和输出dex文件
private void cleanPatch() {
    File[] files = mPatchDir.listFiles();
    for (File file : files) {
        mAndFixManager.removeOptFile(file);
        if (!FileUtil.deleteFile(file)) {
            Log.e(TAG, file.getName() + " delete error.");
        }
    }
}
```
```
/**
 * add patch at runtime
 * 
 *
 * @param path patch path
 * @throws IOException
 */
public void addPatch(String path) throws IOException {
    File src = new File(path);
    File dest = new File(mPatchDir, src.getName());
    if (!src.exists()) {
        throw new FileNotFoundException(path);
    }
    if (dest.exists()) {
        Log.d(TAG, "patch [" + path + "] has be loaded.");
        return;
    }
    FileUtil.copyFile(src, dest);// copy to patch's directory
    Patch patch = addPatch(dest);
    if (patch != null) {
        loadPatch(patch);
    }
}
```
```
/**
 * load specific patch
 *
 * @param patch patch
 */
private void loadPatch(Patch patch) {
    Set<String> patchNames = patch.getPatchNames();
    ClassLoader cl;
    List<String> classes;
    for (String patchName : patchNames) {
        if (mLoaders.containsKey("*")) {
            cl = mContext.getClassLoader();
        } else {
            cl = mLoaders.get(patchName);
        }
        if (cl != null) {
            classes = patch.getClasses(patchName);
            mAndFixManager.fix(patch.getFile(), cl, classes);
        }
    }
}
```

### AndFixManager.java

fix dex files

```
/**
 * fix
 *
 * @param file        patch file
 * @param classLoader classloader of class that will be fixed
 * @param classes     classes will be fixed
 */
public synchronized void fix(File file, ClassLoader classLoader,
                             List<String> classes) {
    // 系统是否支持                         
    if (!mSupport) {
        return;
    }
    if (!mSecurityChecker.verifyApk(file)) {// security check fail 签名校验
        return;
    }
    try {
        // loadClass 输出目录
        File optfile = new File(mOptDir, file.getName());
        // 保存指纹签名
        boolean saveFingerprint = true;
        if (optfile.exists()) {
            // need to verify fingerprint when the optimize file exist,
            // prevent someone attack on jailbreak device with
            // Vulnerability-Parasyte.
            // btw:exaggerated android Vulnerability-Parasyte
            // http://secauo.com/Exaggerated-Android-Vulnerability-Parasyte.html
            // 校验MD5            
            if (mSecurityChecker.verifyOpt(optfile)) {  
                saveFingerprint = false;
            } else if (!optfile.delete()) {
                return;
            }
        }
        final DexFile dexFile = DexFile.loadDex(file.getAbsolutePath(),
                optfile.getAbsolutePath(), Context.MODE_PRIVATE);
        if (saveFingerprint) {        
        	// 保存MD5
            mSecurityChecker.saveOptSig(optfile); 
        }
        ClassLoader patchClassLoader = new ClassLoader(classLoader) {
            @Override
            protected Class<?> findClass(String className)
                    throws ClassNotFoundException {
                Class<?> clazz = dexFile.loadClass(className, this);
                if (clazz == null
                         && className.startsWith("com.alipay.euler.andfix")) {
                    return Class.forName(className);// annotation’s class
                    // not found
                }
                if (clazz == null) {
                    throw new ClassNotFoundException(className);
                }
                return clazz;
            }
        };
        Enumeration<String> entrys = dexFile.entries();
        Class<?> clazz = null;
        while (entrys.hasMoreElements()) {
            String entry = entrys.nextElement();
            if (classes != null && !classes.contains(entry)) {
                continue;// skip, not need fix
            }
            // loadClass
            clazz = dexFile.loadClass(entry, patchClassLoader);  
            if (clazz != null) {
                // fixClass
                fixClass(clazz, classLoader);          
            }
        }
    } catch (IOException e) {
        Log.e(TAG, "pacth", e);
    }
}
```

```
/**
 * fix class
 *
 * @param clazz class
 */
private void fixClass(Class<?> clazz, ClassLoader classLoader) {
    Method[] methods = clazz.getDeclaredMethods();
    MethodReplace methodReplace;
    String clz;
    String meth;
    for (Method method : methods) {
        //  反射提取带有MethodReplace注解的方法
        methodReplace = method.getAnnotation(MethodReplace.class);  
        if (methodReplace == null)
            continue;
        clz = methodReplace.clazz();
        meth = methodReplace.method();
        if (!isEmpty(clz) && !isEmpty(meth)) {
            // jni层替换方法
            replaceMethod(classLoader, clz, meth, method);
        }
    }
}
```


### SecurityChecker.java

用于验证apk和dex的签名

```
init()获取apk的签名mPublicKey
```
```
verifyApk(File path)  //loadPatch()时检查apk和补丁的签名
```
```
签名校验失败时抛出异常
E/SecurityChecker: /data/data/com.euler.andfix/files/apatch/out.apatch java.security.SignatureException: Signature was not verified 
at org.apache.harmony.security.provider.cert.X509CertImpl.verify(X509CertImpl.java:384) 
at com.alipay.euler.andfix.security.SecurityChecker.check(SecurityChecker.java:158)                                                                                                                                              at com.alipay.euler.andfix.security.SecurityChecker.verifyApk(SecurityChecker.java:124) 
at com.alipay.euler.andfix.AndFixManager.fix(AndFixManager.java:121) 
at com.alipay.euler.andfix.patch.PatchManager.loadPatch(PatchManager.java:230) 
at com.alipay.euler.andfix.patch.PatchManager.addPatch(PatchManager.java:161) 
at com.euler.andfix.MainApplication.onCreate(MainApplication.java:63)
```

### Compat.java

检查当前系统是否支持andfix
AndFix supports Android version from 2.3 to 6.0, both ARM and X86 architecture, both Dalvik and ART runtime.
not support alibaba's YunOs

### MethodReplace.java

```
/**
 * Annotation for method
 *
 * @author sanping.li@alipay.com
 */
@Target(ElementType.METHOD) // ElementType是用来指定Annotation类型可以用在哪一些元素上的
@Retention(RetentionPolicy.RUNTIME)  // 注解的保存策略
/**
 * SOURCE：只会保留在程序源码里，源码如果经过了编译之后，Annotation的数据就会消失,并不会保留在编译好的.class文件里面
 * CLASS：Annotation类型的信息保留在程序源码里,同时也会保留在编译好的.class文件里面,在执行的时候，并不会把这一些信息加载到虚拟机
 * (JVM)中去.注意一下，当你没有设定一个Annotation类型的Retention值时，系统默认值是CLASS.
 * RUNTIME：表示在源码、编译好的.class文件中保留信息，在执行的时候会把这一些信息加载到JVM中去的．
 */
public @interface MethodReplace {
    String clazz();
    String method();
}
```

### AndFix.java


```
// initialize art or dalvik 
private static native boolean setup(boolean isArt, int apilevel);
// 
private static native void replaceMethod(Method dest, Method src);
/**
 * modify access flag of class’ fields to public
 *
 * @param field field
 */
private static native void setFieldFlag(Field field);
```

### 其他

生成的补丁out.apatch是带有签名信息的压缩包，

![image](http://nulibj.github.io/src/andfix/images/out.apatch.zip.png)

META_INFO文件夹包含MANIFEST.MF、CERT.SF和CERT.RSA、PATCH.MF。这三个文件分别表征以下含义：

（1）MANIFEST.MF：这是摘要文件。程序遍历Apk包中的所有文件(entry)，对非文件夹非签名文件的文件，逐个用SHA1生成摘要信息(用SHA1算法摘要的消息最终有160比特位的输出)，再用Base64进行编码。如果你改变了apk包中的文件，那么在apk安装校验时，改变后的文件摘要信息与MANIFEST.MF的检验信息不同，于是程序就不能成功安装。
说明：如果攻击者修改了程序的内容，有重新生成了新的摘要，那么就可以通过验证，所以这是一个非常简单的验证。

（2）CERT.SF：这是对摘要的签名文件。对前一步生成的MANIFEST.MF，使用SHA1-RSA算法，用开发者的私钥进行签名。在安装时只能使用公钥才能解密它。解密之后，将它与未加密的摘要信息（即，MANIFEST.MF文件）进行对比，如果相符，则表明内容没有被异常修改。
说明：在这一步，即使开发者修改了程序内容，并生成了新的摘要文件，但是攻击者没有开发者的私钥，所以不能生成正确的签名文件（CERT.SF）。系统在对程序进行验证的时候，用开发者公钥对不正确的签名文件进行解密，得到的结果和摘要文件（MANIFEST.MF）对应不起来，所以不能通过检验，不能成功安装文件。

（3）CERT.RSA文件中保存了公钥、所采用的加密算法等信息。
说明：系统对签名文件进行解密，所需要的公钥就是从这个文件里取出来的。
结论：从上面的总结可以看出，META-INFO里面的说那个文件环环相扣，从而保证Android程序的安全性。（只是防止开发者的程序不被攻击者修改，如果开发者的公私钥对对攻击者得到或者开发者开发出攻击程序，Android系统都无法检测出来。）

![image](http://nulibj.github.io/src/andfix/images/CERT.RSA.png)

（4）PATCH.MF 由`apkpatch` tool 生成，主要内容：

Manifest-Version: 1.0

Patch-Name: demo-debug2

Created-Time: 15 Apr 2016 10:10:12 GMT

From-File: demo-debug2.apk

To-File: demo-debug1.apk

Patch-Classes: com.euler.andfix.SecondAvtivity_CF,com.euler.andfix.MainApplication_CF

Created-By: 1.0 (ApkPatch)

// 获取CERT.RSA公钥信息
openssl pkcs7 -inform DER -in CERT.RSA -noout -print_certs -text 


### 加载apatch过程

1、isSupport 判断设备是否支持andfix

2、copy /data/data/packageName/files/apatch/ 下

3、verify 校验apatch签名，对比apk的publickey和apatch的publickey

4、loaddex /data/data/packageName/files/apatch_opt/

5、repleaseMethod  根据PATCH.MF中Patch-Classes找到需要替换的class，再由class反射提取带有MethodReplace注解的方法，jni层替换，立即修复

![image](http://nulibj.github.io/src/andfix/images/method_replace.png)

### Apk重签名

1、解压apk，删除META_INFO文件夹，再压缩改后缀.apk

2、生成keystore签名文件

keytool -genkey -alias demo -keyalg RSA -validity 20000 -keystore demo.keystore

-genkey    产生证书文件 

-keystore  指定密钥库的.keystore文件中 

-keyalg    指定密钥的算法

-validity  为证书有效天数，这里我们写的是20000天

-alias     产生别名 

3、apk签名

jarsigner -verbose -keystore demo.keystore -signedjar demo.apk demo_old.apk demo -digestalg SHA1 -sigalg MD5withRSA

## 参考
[https://github.com/alibaba/AndFix](https://github.com/alibaba/AndFix)

[Alibaba-AndFix Bug热修复框架原理及源码解析](http://blog.csdn.net/qxs965266509/article/details/49816007)

[Android签名与认证详细分析之一（CERT.RSA剖析）](http://myeyeofjava.iteye.com/blog/2125348)

[[Android Pro] Android签名与认证详细分析之二（CERT.RSA剖析）](http://www.cnblogs.com/0616--ataozhijia/p/4482667.html)


