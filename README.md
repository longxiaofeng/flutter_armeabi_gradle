Flutter 适配Android armeabi
由于 Flutter 在设计之初就不支持 armeabi 架构的 cpu，若以如果原生工程无法修改的情况下就要介入 Flutter 的编译过程来适配 armeabi。 不同版本的 Flutter 适配方案也不太一样

FlutterSdk 1.9.6 老版本在so在sdk bin/cache/artifacts/engine 中

方案1 ：https://github.com/lizhangqu/plugin-flutter-armeabi

方案2：美团方案
```javascript
for arch in android-arm android-arm-profile android-arm-release ; do
    pushd $arch
    cp flutter.jar flutter-armeabi-v7a.jar #备份
    unzip flutter.jar lib/armeabi-v7a/libflutter.so
    mv lib/armeabi-v7a lib/armeabi
    zip -d flutter.jar lib/armeabi-v7a/libflutter.so
    zip flutter.jar lib/armeabi/libflutter.so
    popd
done
```
复制代码
FlutterSdk 2.0.3 新版本为远程依赖 mirrors.sjtug.sjtu.edu.cn/download.fl… 上面的方案1与方案2已经不太适用

新方案修改打包流程，在打包流程中完成so的拷贝逻辑，可以减少FlutterSdk重大修改造成脚本不适用

新建flutter_arm.gradle文件
```javascript
project.setProperty('target-platform', 'android-arm')
project.afterEvaluate {
    android.applicationVariants.all { variant ->

        def variantName = variant.name.capitalize()

        def multidexTask = project.tasks.findByName("transformNativeLibsWithMergeJniLibsFor${variantName}")
        if (multidexTask != null) {
            def copyFlutterSoTask = createcopyFlutterSoTask(variant)
            multidexTask.finalizedBy copyFlutterSoTask
        }
    }
}

def copyFile(String src, String dest) {
    println "libflutter src: " + src + " dest: " + dest
    def srcFile = new File(src)
    def targetFile = new File(dest)
    targetFile.withOutputStream { os ->
        srcFile.withInputStream { ins ->
            os << ins
        }
    }
}

def createcopyFlutterSoTask(variant) {
    def variantName = variant.name.capitalize()

    return task("replace${variantName}FlutterArm").doLast {

        def root = rootDir.getParentFile().getAbsolutePath() + File.separator + "build" + File.separator + "app" + File.separator + "intermediates" + File.separator +
                "transforms" + File.separator + "mergeJniLibs" + File.separator + variantName + File.separator + "0" + File.separator + "lib";

        String fromFilePath = root + File.separator + "armeabi-v7a" + File.separator + "libflutter.so"
        String foFilePath = root + File.separator + "armeabi" + File.separator + "libflutter.so"
        copyFile(fromFilePath, foFilePath)
        File foFile = new File(foFilePath)
        if (!foFile.exists() || foFile.length() < 100) {
            throw new GradleException("copyFlutter libflutter.so error please check")
        }
    }
}
```


复制代码
在MergeJniLibs后，将armeabi-v7a的libflutter.so复制到armeabi中
```javascript
apply from: "flutter_arm.gradle"  // 必须放在flutter.gradle引用之前 要不然读取不到target-platform的值
apply from: "$flutterRoot/packages/flutter_tools/gradle/flutter.gradle"
```
