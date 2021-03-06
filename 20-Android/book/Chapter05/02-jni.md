JNI
===

Android NDK开发指Java层通过JNI（Java Native Interface）与原生代码如（C，C++等）进行交互。

### JNI基本数据类型

|  Java类型  |    原生类型    |    描述  |
|:---------:|:-------------:|:--------:|
| boolean   | jboolean   | unsigned 8 bits
| byte   | jbyte   | unsigned 8 bits
| char   | jchar   | unsigned 16 bits
| short   | jshort   | signed 16 bits
| int   | jint   | signed 32 bits
| long   | jlong   | signed 64 bits
| float   | jfloat   | 32 bits
| double   | jdouble   | 64 bits

### Java调用原生方法

```
1. java层定义native函数
public native int add(int x, int y);

2. 在C中实现native函数
jint Java_com_lex_MainActivity_add(JNIEnv *env, jobject thiz, jint x, jint y) {
    __android_log_print(ANDROID_LOG_INFO, "JNIMsg", "Get Param:x=%d y=%d", x, y);
    return x + y;
}

3. 编译C函数为so文件，通过loadLibarary载入Java工程。
```
### C调用Java静态方法

```
1. 在Java层定义将被调用的静态函数
public static int showMessage(String title, String str)

2. 在C中查找类对象jclass
jclass java_class = (*env)->FindClass(env, "com/lex/jni/JniActivity");

3. 在C中得到静态方法的ID对象jmethodID
jmethodID java_method = (*env)->GetStaticMethodID(env, java_class, "showMessage", "(Ljava/lang/String;Ljava/lang/String;)I");

4. C中调用静态方法ID对象jmethodID
(*env)->CallStaticIntMethod(env, java_class, java_method, title, val);

上述代码返回值是jint，根据返回类型不同，调用的静态方法也不同具体如下：
CallStaticBooleanMethod
CallStaticByteMethod
CallStaticCharMethod
CallStaticDoubleMethod
CallStaticFloatMethod
CallStaticIntMethod
CallStaticLongMethod
CallStaticObjectMethod
CallStaticShortMethod
CallStaticStringMethod
CallStaticVoidMethod
```


### C调用Java非静态方法，即成员方法

```
1. 在Java层定义将被调用的静态函数
public static int showMessage2(String title, String str)

2. 在C中查找类对象jclass
jclass java_class = (*env)->FindClass(env, "com/lex/jni/JniActivity");

3. 获取构造方法ID并构建类对象jobject

jmethodID construction_id = (*env)->GetMethodID(env, java_class, "<int>", "()V");
jobject obj = (*env)->NewObject(env, java_class, construction_id);

4. 在C中查找要调用的非静态方法ID对象jmethodID
jmethodID java_method = (*env)->GetMethodID(env, java_class, "showMessage2", "(Ljava/lang/String;Ljava/lang/String;)I");

4. C中调用非静态方法
(*env)->CallIntMethod(env, java_obj, java_method, title, val);

上述代码返回值是jint，根据返回类型不同，调用的非静态方法也不同具体如下：
CallBooleanMethod
CallByteMethod
CallCharMethod
CallDoubleMethod
CallFloatMethod
CallIntMethod
CallLongMethod
CallObjectMethod
CallShortMethod
CallStringMethod
CallVoidMethod
```

### Java方法映射到C中的签名

|  Java类型  |    符号    |
|:---------:|:----------:|
| Boolean | Z
| Byte | B
| Char | C
| Short | S
| Int | I
| Long | J
| Float | F
| Double | D
| Void | V
| Objects | 以"L"开头，以";"结尾，中间是用"/" 隔开的包及类名。比如：Ljava/lang/String;如果是嵌套类，则用$来表示嵌套。例如 "(Ljava/lang/String;Landroid/os/FileUtils$FileStatus;)Z"


### Android应用层调用C函数示例

1. Java定义native接口。
2. 使用C或C++实现本地方法。
3. 生成动态链接库so文件，将so文件复制到Java工程，通过loadLibrary()载入库文件。


##### 1. 定义native接口
创建Android工程，新建一个java文件如下：

```java
public class HelloJni extends Activity {
    /** Called when the activity is first created. */
    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        /* Create a TextView and set its content.
         * the text is retrieved by calling a native
         * function.
         */
        TextView  tv = new TextView(this);
        tv.setText( stringFromJNI() );
        setContentView(tv);
    }

    /* A native method that is implemented by the
     * 'hello-jni' native library, which is packaged
     * with this application.
     */
    public native String  stringFromJNI();

    /* This is another native method declaration that is *not*
     * implemented by 'hello-jni'. This is simply to show that
     * you can declare as many native methods in your Java code
     * as you want, their implementation is searched in the
     * currently loaded native libraries only the first time
     * you call them.
     *
     * Trying to call this function will result in a
     * java.lang.UnsatisfiedLinkError exception !
     */
    public native String  unimplementedStringFromJNI();

    /* this is used to load the 'hello-jni' library on application
     * startup. The library has already been unpacked into
     * /data/data/com.example.hellojni/lib/libhello-jni.so at
     * installation time by the package manager.
     */
    static {
        System.loadLibrary("hello-jni");
    }
}
```

##### 2. 使用C或C++实现原生方法
在工程目录下，新建一个jni文件夹。创建hello-jni.c文件。

```c
#include <string.h>
#include <jni.h> // Jni所有接口都定义在Jni.h中，通常在NDK程序中都要包含该头文件

/* This is a trivial JNI example where we use a native method
 * to return a new VM String. See the corresponding Java source
 * file located at:
 *
 *   src/com/example/hellojni/HelloJni.java
 */
jstring Java_com_example_hellojni_HelloJni_stringFromJNI( JNIEnv* env, jobject thiz )
{
    return (*env)->NewStringUTF(env, "Hello from JNI");
}
```

##### 3. 生成动态链接库so文件

在原生代码目录下建立一个Android.mk文件：

```makefile
LOCAL_PATH := $(call my-dir)    # 宏函数’my-dir’由编译系统提供，用于返回当前路径，以便查找源文件
include $(CLEAR_VARS)           # 清除许多LOCAL_XXX变量，如LOCAL_MODULE,LOCAL_SRC_FILES等
LOCAL_MODULE:= hello-jni        # so文件名，系统会自动增加lib前缀和.so后缀
LOCAL_SRC_FILES := hello-jni.c  # 要编译的源码文件，不需要列出头文件和包含文件，系统自动加入依赖文件
include $(BUILD_SHARED_LIBRARY) # 编译为动态库，要编译.a动态库使用BUILD_STATIC_LIBRARY
```

进入原生代码目录执行：$NDK/ndk-build，此时会在工程目录下生成libs目录，在libs/armeabi/目录中会生成libhello-jni.so文件。将so文件载入到Android工程的libs/armeabi/目录中，在代码中调用即可。

```
static {
    System.loadLibrary("hello-jni");
}
```
