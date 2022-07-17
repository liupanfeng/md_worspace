### Android ASM 字节码插桩

字节码插桩：就是由class到dex之前修改class文件，达到增强现有类的功能。

**1.Android工程的构建过程**

* 1.Android Resources-->**通过aapt**-->R.java

* 2.aidl Files-->**通过aidl**-->java interface

* 3.(R.java、Android Resouce code、java interface）-->**java compile**-->.class Files
* 4.(.class Files、3rd Party Libraries and class Files)-->**dex 编译器**-->.dex Files
* 5.(dex Files、Other Resources）-->**Apk Builder**-->Android Package(.apk)-->**jar signer**-->Signed Apk

以上就是构建的整个过程，其中字节码插桩就是.class Files 经过dex编译器之前的操作，对class的增强。



**2.Transform**

Transform是Android 官方插件提供给开发者在项目构建阶段由Class到Dex转换之前修改Class文件的一套API。目前典型的应用场景就是字节码插桩。

通过Transform可以得到所有的class字节码，我们自定义的Transfrom会先执行，执行的结果做为参数进行传递。



**3.新建一个插件类**

```java
public class APMPlugin implements Plugin<Project> {

    @Override
    public void apply(Project project) {
        BaseExtension android = project.getExtensions().getByType(BaseExtension.class);
        /**
         * 注册一个Transform
         */
        android.registerTransform(new ASMTransfrom());
    }
}
```

android 插件能够获得所有的class，并通过接口的形式暴露出来。



**4.创建一个ASM**

```java
public class ASMTransform extends Transform {

    @Override
    public String getName() {
        return "ms_asm";
    }

    /**
     * 处理所有class
     *
     * @return
     */
    @Override
    public Set<QualifiedContent.ContentType> getInputTypes() {
        return TransformManager.CONTENT_CLASS;
    }

    /**
     * 范围仅仅是主项目所有的类
     *
     * @return
     */
    @Override
    public Set<? super QualifiedContent.Scope> getScopes() {
        return TransformManager.PROJECT_ONLY;
    }

    /**
     * 不使用增量
     * @return
     */
    @Override
    public boolean isIncremental() {
        return false;
    }

    /**
     * android插件将所有的class通过这个方法告诉给我们
     *
     * @param transformInvocation
     * @throws TransformException
     * @throws InterruptedException
     * @throws IOException
     */
    @Override
    public void transform(TransformInvocation transformInvocation) throws TransformException, InterruptedException, IOException {
        super.transform(transformInvocation);

        TransformOutputProvider outputProvider = transformInvocation.getOutputProvider();
        //清理上次缓存的文件信息
        outputProvider.deleteAll();


        //得到所有的输入
        Collection<TransformInput> inputs = transformInvocation.getInputs();
        for (TransformInput input : inputs) {
            // 处理class目录
            for (DirectoryInput directoryInput : input.getDirectoryInputs()) {
                // 直接复制输出到对应的目录
                String dirName = directoryInput.getName();
                File src = directoryInput.getFile();
                System.out.println("输出class文件：" + src);
                String md5Name = DigestUtils.md5Hex(src.getAbsolutePath());
                //得到输出class文件的目录
                File dest = outputProvider.getContentLocation(dirName + md5Name,
                        directoryInput.getContentTypes(), directoryInput.getScopes(),
                        Format.DIRECTORY);
                //执行插桩操作
                processInject(src, dest);
            }
            // 处理jar（依赖）的class
            for (JarInput jarInput : input.getJarInputs()) {
                String jarName = jarInput.getName();
                File src = jarInput.getFile();
                System.out.println("输出jar包：" + src);
                String md5Name = DigestUtils.md5Hex(src.getAbsolutePath());
                if (jarName.endsWith(".jar")) {
                    jarName = jarName.substring(0, jarName.length() - 4);
                }
                File dest = outputProvider.getContentLocation(jarName + md5Name,
                        jarInput.getContentTypes(), jarInput.getScopes(), Format.JAR);
                FileUtils.copyFile(src, dest);
            }
        }
    }

    private void processInject(File src, File dest) throws IOException {
        String dir = src.getAbsolutePath();
        FluentIterable<File> allFiles = FileUtils.getAllFiles(src);
        for (File file : allFiles) {
            //得到文件输入流
            FileInputStream fis = new FileInputStream(file);
            //得到字节码Reader
            ClassReader cr = new ClassReader(fis);
            //得到写出器
            ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);
            //将注入的时间信息，写入
            cr.accept(new ClassInjectTimeVisitor(cw,file.getName()),
                    ClassReader.EXPAND_FRAMES);

            byte[] newClassBytes = cw.toByteArray();
            String absolutePath = file.getAbsolutePath();
            String fullClassPath = absolutePath.replace(dir, "");
            //将得到的字节码信息 写如输出目录
            File outFile = new File(dest, fullClassPath);
            FileUtils.mkdirs(outFile.getParentFile());
            FileOutputStream fos = new FileOutputStream(outFile);
            fos.write(newClassBytes);
            fos.close();
        }

    }
}

```

继承Transform 重写父类的方法（com.android.build.api.transform.Transform）是这个别弄错了。

* getName() 返回transfrom的方法名，这个随便定义
* getInputTypes() 得到需要处理的内容类型，TransformManager.CONTENT_CLASS 这个表示字节码。
* getScopes() 返回的是处理范围，比如是整个项目还是仅仅主app等
* isIncremental() 是否增量
* transform() 这个方法会回调我们需要的所有类信息



**5.创建字节码注入时间方法器**

```java
public class ClassInjectTimeVisitor extends ClassVisitor {

    /**
     * 得到类名
     */
    private String mClassName;

    public ClassInjectTimeVisitor(ClassVisitor cv, String fileName) {
        super(Opcodes.ASM5, cv);
        mClassName = fileName.substring(0, fileName.lastIndexOf("."));
    }


    /**
     * 访问方法
     * @param access 方法的访问flag
     * @param name 方法名
     * @param desc 描述信息
     * @param signature 签名信息
     * @param exceptions
     * @return
     */
    @Override
    public MethodVisitor visitMethod(int access, String name, String desc, String signature,
                                     String[] exceptions) {

        MethodVisitor mv = super.visitMethod(access, name, desc, signature,
                exceptions);
        return new MethodAdapterVisitor(mv, access, name, desc, mClassName);
    }

}
```

visitMethod这个方法负责拦截所有的方法，并初始化一个方法适配器MethodAdapterVisitor

MethodAdapterVisitor，负责具体的插桩代码逻辑。



**5.准备需要插入的class信息**

```java
public static void main(String[] args) throws InterruptedException {
    long start = System.currentTimeMillis();

    Thread.sleep(1000);

    long end = System.currentTimeMillis();
    System.out.println("execute:"+(end-start)+" ms.");
}
```

插入方法耗时统计：

* 需要在方法开始插入这行代码：long start = System.currentTimeMillis();

* 结尾处插入：

  ​    long end = System.currentTimeMillis();
  ​    System.out.println("execute:"+(end-start)+" ms.");

将上面的java信息转成ASM Bytecode，以便方法的进行插桩注入

```java
 {
            methodVisitor = classWriter.visitMethod(ACC_PUBLIC | ACC_STATIC, "main", "([Ljava/lang/String;)V", null, new String[]{"java/lang/InterruptedException"});
            methodVisitor.visitCode();
     
     		//long start = System.currentTimeMillis();
            Label label0 = new Label();
            methodVisitor.visitLabel(label0);
            methodVisitor.visitLineNumber(7, label0);
            methodVisitor.visitMethodInsn(INVOKESTATIC, "java/lang/System", "currentTimeMillis", "()J", false);
            methodVisitor.visitVarInsn(LSTORE, 1);
     
     
            // 下面是执行的 Thread.sleep(1_000); 
            Label label1 = new Label();
            methodVisitor.visitLabel(label1);
            methodVisitor.visitLineNumber(9, label1);
            methodVisitor.visitLdcInsn(new Long(1000L));
            methodVisitor.visitMethodInsn(INVOKESTATIC, "java/lang/Thread", "sleep", "(J)V", false);
           
            //下面是执行的 long end = System.currentTimeMillis();
            //        System.out.println("execute:"+(end-start)+" ms.");
            Label label2 = new Label();
            methodVisitor.visitLabel(label2);
            methodVisitor.visitLineNumber(11, label2);
            methodVisitor.visitMethodInsn(INVOKESTATIC, "java/lang/System", "currentTimeMillis", "()J", false);
            methodVisitor.visitVarInsn(LSTORE, 3);
            Label label3 = new Label();
            methodVisitor.visitLabel(label3);
            methodVisitor.visitLineNumber(12, label3);
            methodVisitor.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
            methodVisitor.visitTypeInsn(NEW, "java/lang/StringBuilder");
            methodVisitor.visitInsn(DUP);
            methodVisitor.visitMethodInsn(INVOKESPECIAL, "java/lang/StringBuilder", "<init>", "()V", false);
            methodVisitor.visitLdcInsn("execute:");
            methodVisitor.visitMethodInsn(INVOKEVIRTUAL, "java/lang/StringBuilder", "append", "(Ljava/lang/String;)Ljava/lang/StringBuilder;", false);
            methodVisitor.visitVarInsn(LLOAD, 3);
            methodVisitor.visitVarInsn(LLOAD, 1);
            methodVisitor.visitInsn(LSUB);
            methodVisitor.visitMethodInsn(INVOKEVIRTUAL, "java/lang/StringBuilder", "append", "(J)Ljava/lang/StringBuilder;", false);
            methodVisitor.visitLdcInsn(" ms.");
            methodVisitor.visitMethodInsn(INVOKEVIRTUAL, "java/lang/StringBuilder", "append", "(Ljava/lang/String;)Ljava/lang/StringBuilder;", false);
            methodVisitor.visitMethodInsn(INVOKEVIRTUAL, "java/lang/StringBuilder", "toString", "()Ljava/lang/String;", false);
            methodVisitor.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
     
     
            //其他内容不需要处理
            Label label4 = new Label();
            methodVisitor.visitLabel(label4);
            methodVisitor.visitLineNumber(13, label4);
            methodVisitor.visitInsn(RETURN);
            Label label5 = new Label();
            methodVisitor.visitLabel(label5);
            methodVisitor.visitLocalVariable("args", "[Ljava/lang/String;", null, label0, label5, 0);
            methodVisitor.visitLocalVariable("start", "J", null, label1, label5, 1);
            methodVisitor.visitLocalVariable("end", "J", null, label3, label5, 3);
            methodVisitor.visitMaxs(6, 5);
            methodVisitor.visitEnd();
        }
```

  上边的内容我对 相关的内容 做了注释，分别在方法的执行最前面和最后面插入相关的代码逻辑就好，这部分是辅助内容。



**6.创建方法访问者适配器**

```java
public class MethodAdapterVisitor extends AdviceAdapter {

    private String mClassName;
    private String mMethodName;
    private boolean mInject;
    private int mStart, mEnd;

    protected MethodAdapterVisitor(MethodVisitor mv, int access, String name, String desc,
                                   String className) {
        super(Opcodes.ASM5, mv, access, name, desc);
        mMethodName = name;
        this.mClassName = className;
    }


    /**
     * 拦截注解方法
     *
     * @param desc
     * @param visible
     * @return
     */
    @Override
    public AnnotationVisitor visitAnnotation(String desc, boolean visible) {
        if ("Lcom/meishe/ms_asminject/MSTimeAnalysis;".equals(desc)) {
            mInject = true;
        }
        return super.visitAnnotation(desc, visible);
    }


    /**
     * 方法进入
     */
    @Override
    protected void onMethodEnter() {
        if (mInject) {
            //执行方法currentTimeMillis 得到startTime
            mv.visitMethodInsn(INVOKESTATIC, "java/lang/System", "currentTimeMillis", "()J", false);
            mStart = newLocal(Type.LONG_TYPE);
            mv.visitVarInsn(LSTORE, mStart);

        }
    }


    /**
     * 方法结束
     *
     * @param opcode
     */
    @Override
    protected void onMethodExit(int opcode) {
        if (mInject) {

            //执行 currentTimeMillis 得到end time
            mv.visitMethodInsn(INVOKESTATIC, "java/lang/System", "currentTimeMillis", "()J", false);
            mEnd =newLocal(Type.LONG_TYPE);
            mv.visitVarInsn(LSTORE, mEnd);

            //得到静态成员 out
            mv.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
            //new  //class java/lang/StringBuilder
            mv.visitTypeInsn(NEW, "java/lang/StringBuilder");
            //引入类型 分配内存 并dup压入栈顶让下面的INVOKESPECIAL 知道执行谁的构造方法
            mv.visitInsn(DUP);

            //执行init方法 （构造方法）
            mv.visitMethodInsn(INVOKESPECIAL, "java/lang/StringBuilder", "<init>", "()V", false);
            //把常量压入栈顶
            mv.visitLdcInsn("execute "+ mMethodName +" :");
            //执行append方法，使用栈顶的值作为参数
            mv.visitMethodInsn(INVOKEVIRTUAL, "java/lang/StringBuilder", "append", "(Ljava/lang/String;)Ljava/lang/StringBuilder;", false);

            //  获得存储的本地变量
            mv.visitVarInsn(LLOAD, mEnd);
            mv.visitVarInsn(LLOAD, mStart);
            // lsub   减法指令
            mv.visitInsn(LSUB);

            //把减法结果append
            mv.visitMethodInsn(INVOKEVIRTUAL, "java/lang/StringBuilder", "append", "(J)Ljava/lang/StringBuilder;", false);
            //拼接常量
            mv.visitLdcInsn(" ms.");
            mv.visitMethodInsn(INVOKEVIRTUAL, "java/lang/StringBuilder", "append", "(Ljava/lang/String;)Ljava/lang/StringBuilder;", false);

            //执行StringBuilder 的toString方法
            mv.visitMethodInsn(INVOKEVIRTUAL, "java/lang/StringBuilder", "toString", "()Ljava/lang/String;", false);
            //执行println方法
            mv.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);

        }
    }

}
```

继承AdviceAdapter，并重写相关的方法（org.objectweb.asm.commons.AdviceAdapter）是这个别继承错了。

* visitAnnotation() 这个会输出方法拥有的注解信息，这里我们只对我们添加注解的方法进行注入操作。

* onMethodEnter() 方法进入的时候会回调，在这里插入long start = System.currentTimeMillis();

* onMethodExit(int opcode) 方法结束的时候回调，在这里插入：

   long end = System.currentTimeMillis();
   System.out.println("execute:"+(end-start)+" ms.");



**7.上面用的注解**

```java
@Documented
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.CLASS)
public @interface MSTimeAnalysis {
}
```

声明一个编译器注解即可，由于上边用到了，贴出来。



这样就完成了字节码插桩的全部工作。



**8.进行测试工作**

```java
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        testApp();
    }

    @MSTimeAnalysis
    public void testApp(){
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

声明一个testApp()方法，增加@MSTimeAnalysis方法，并调用。运行就可以看到输出：

2022-05-03 11:08:10.824 27788-27788/com.meishe.ms_asminject I/System.out: execute testApp :1000 ms.

在build/intermediates/transforms/ms_asm/debug/1/com/meishe/ms_asminject/MainActivity.class可以查看编译生成字节码：

```java
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    this.setContentView(2131427356);
    this.testApp();
}

@MSTimeAnalysis
public void testApp() {
    long var1 = System.currentTimeMillis();

    try {
        Thread.sleep(1000L);
    } catch (InterruptedException var6) {
        var6.printStackTrace();
    }

    long var4 = System.currentTimeMillis();
    System.out.println("execute testApp :" + (var4 - var1) + " ms.");
}
```

证明确实插桩成功了。



