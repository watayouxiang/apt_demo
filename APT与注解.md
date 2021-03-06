# APT与注解

## 注解语法

```java
/**
 * 注解，单独是没有意义的。如下是具体应用场景：
 *
 * 注解+APT：					 用于生成一些java文件，如：butterknife, dagger2, hilt, databinding
 * 注解+代码埋点：			  AspactJ, ARouter
 * 注解+反射+动态代理：	 XUtils, Lifecycle
 */
@Target({ElementType.FIELD, ElementType.METHOD})// 作用在什么地方
@Retention(RetentionPolicy.RUNTIME)							// 作用范围
public @interface BindView {
    String value();
    int id();
}

public enum ElementType {
    TYPE,               /* 类、接口（包括注释类型）或枚举声明 */
    FIELD,              /* 字段声明（包括枚举常量）*/
    METHOD,             /* 方法声明 */
    PARAMETER,          /* 参数声明 */
    CONSTRUCTOR,        /* 构造方法声明 */
    LOCAL_VARIABLE,     /* 局部变量声明 */
    ANNOTATION_TYPE,    /* 注释类型声明 */
    PACKAGE             /* 包声明 */
}

public enum RetentionPolicy {
    SOURCE,            /* Annotation信息仅存在于编译器处理期间，编译器处理完之后就没有该Annotation信息了 */
    CLASS,             /* 编译器将Annotation存储于类对应的.class文件中。默认行为 */
    RUNTIME            /* 编译器将Annotation存储于class文件中，并且可由JVM读入 */
}
```

## @IntDef代替枚举

```java
class Test {
    private static final int SUNDAY = 0;
    private static final int MONDAY = 1;

    // 注解代替枚举，可以节省内存空间
    @IntDef({SUNDAY, MONDAY})
    @Target(ElementType.PARAMETER)
    @Retention(RetentionPolicy.SOURCE)
    @interface WeekDay{
    }

    public static void setCurrDay(@WeekDay int currDay) {
    }

    public static void main(String[] args) {
        setCurrDay(SUNDAY);
    }
}
```

## APT注解处理器使用

> javac 把 java 文件编译成 class 文件，APT就是作用在javac上。

1、新建一个名为 annotation_compiler 的 java library 工程

2、annotation_compiler 工程的 gradle 中添加如下依赖

```java
// 在 java library 的 gradle 文件中添加如下：
annotationProcessor 'com.google.auto.service:auto-service:1.0-rc4'
compileOnly 'com.google.auto.service:auto-service:1.0-rc4'
```

3、开始注解处理器编码

```java
/**
 * 注解处理器，用来生成代码的
 * 使用前需要注册
 */
@AutoService(Processor.class)
public class AnnotationsCompiler extends AbstractProcessor {
    // 1.支持的版本
    @Override
    public SourceVersion getSupportedSourceVersion() {
        return SourceVersion.latestSupported();
    }

    // 2.能用来处理哪些注解
    @Override
    public Set<String> getSupportedAnnotationTypes() {
        // 只能处理 @BindView 类型的注解    
        Set<String> types = new HashSet<>();
        types.add(BindView.class.getCanonicalName());
        return types;
    }
  
    // 3.定义一个用来生成APT目录下面的文件的对象
    Filer filer;

    @Override
    public synchronized void init(ProcessingEnvironment processingEnvironment) {
        super.init(processingEnvironment);
        filer = processingEnvironment.getFiler();
    }
  
    // 4、处理逻辑，生成所需的代码
    @Override
    public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {
      	// 日志打印
      	processingEnv.getMessager().printMessage(Diagnostic.Kind.NOTE, "jett---------------" + set);
      
        return false;
    }
}
```

## APT实现butterknife实战

### 1、新建一个 java-library 类型的注解工程 “annotation”

```java
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.SOURCE)
public @interface BindView {
    int value();
}
```

### 2、新建一个 application 类型工程

```java
public interface IBinder<T> {
    void bind(T target);
}

public class JettButterknife {
    public static void bind(Activity activity) {
        String name = activity.getClass().getName() + "_ViewBinding";
        try {
            Class<?> aClass = Class.forName(name);
            IBinder iBinder = (IBinder) aClass.newInstance();
            iBinder.bind(activity);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}

public class MainActivity extends AppCompatActivity {
    @BindView(R.id.tvText)
    TextView textView;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        JettButterknife.bind(this);
        textView.setText("你好");
    }
}

// gradle 文件中添加如下依赖：
implementation project(path: ':annotations')
annotationProcessor project(path: ':annotation_compiler')
```

### 3、目的是在 application 工程的 `/build/generated/ap_generated_sources/debug/out/...` 路径下生成如下代码：

```java
package com.example.dn_butterknife;

import com.example.dn_butterknife.IBinder;

public class MainActivity_ViewBinding implements IBinder<com.example.dn_butterknife.MainActivity> {
    @Override
    public void bind(com.example.dn_butterknife.MainActivity target) {
        target.textView = (android.widget.TextView) target.findViewById(2131165359);
    }
}
```

### 4、因此 APT 工程的代码应该这么编码

新建 java-ibrary 类型的 annotation_compiler 工程

```java
package com.example.annotation_compiler;

import com.example.annotations.BindView;
import com.google.auto.service.AutoService;
import com.google.errorprone.annotations.Var;

import java.io.Writer;
import java.lang.reflect.Type;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.HashSet;
import java.util.Iterator;
import java.util.List;
import java.util.Map;
import java.util.Set;

import javax.annotation.processing.AbstractProcessor;
import javax.annotation.processing.Filer;
import javax.annotation.processing.Messager;
import javax.annotation.processing.ProcessingEnvironment;
import javax.annotation.processing.Processor;
import javax.annotation.processing.RoundEnvironment;
import javax.lang.model.SourceVersion;
import javax.lang.model.element.Element;
import javax.lang.model.element.ExecutableElement;
import javax.lang.model.element.TypeElement;
import javax.lang.model.element.VariableElement;
import javax.lang.model.type.TypeMirror;
import javax.tools.Diagnostic;
import javax.tools.JavaFileObject;

@AutoService(Processor.class)
public class AnnotationsCompiler extends AbstractProcessor {
    Filer filer;

    @Override
    public synchronized void init(ProcessingEnvironment processingEnvironment) {
        super.init(processingEnvironment);
        filer = processingEnvironment.getFiler();
    }

    @Override
    public SourceVersion getSupportedSourceVersion() {
        return SourceVersion.latestSupported();
    }

    @Override
    public Set<String> getSupportedAnnotationTypes() {
        Set<String> types = new HashSet<>();
        types.add(BindView.class.getCanonicalName());
        return types;
    }

    @Override
    public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {
        processingEnv.getMessager().printMessage(Diagnostic.Kind.NOTE, "jett---------------" + set);

        // 1、获取APP中所有用到了BindView注解的对象
        Set<? extends Element> elementsAnnotatedWith = 
          	roundEnvironment.getElementsAnnotatedWith(BindView.class);


        // 2、开始对elementsAnnotatedWith进行分类
        // 
        // Element 的子类有如下：
        //
        // TypeElement //类
        // ExecutableElement //方法
        // VariableElement //属性
        //
        Map<String, List<VariableElement>> map = new HashMap<>();
        for (Element element : elementsAnnotatedWith) {
            // @BindView 是属性类型，所以直接强转        
            VariableElement variableElement = (VariableElement) element;
            // 获取 @BindView 所在的作用域，也就是Activity类，拿到Activity的名称
            String activityName = variableElement.getEnclosingElement().getSimpleName().toString();
            // 获取 @BindView 所在的作用域，也就是Activity类，拿到Activity的字节码对象
            Class aClass = variableElement.getEnclosingElement().getClass();
            // [
            //   "MainActivity" : {VariableElement1, VariableElement2, ...}
            //   "TwoActivity" : {VariableElement1, VariableElement2}
            //   "ThreeActivity" : {VariableElement1}
            // ]
            List<VariableElement> variableElements = map.get(activityName);
            if (variableElements == null) {
                variableElements = new ArrayList<>();
                map.put(activityName, variableElements);
            }
            variableElements.add(variableElement);
        }


        // 3、开始生成文件
        //
        // package com.example.dn_butterknife;
        // import com.example.dn_butterknife.IBinder;
        // public class MainActivity_ViewBinding implements IBinder<com.example.dn_butterknife.MainActivity> {
        //     @Override
        //     public void bind(com.example.dn_butterknife.MainActivity target) {
        //         target.textView = (android.widget.TextView) target.findViewById(2131165359);
        //     }
        // }
        //
        if (map.size() > 0) {
            Writer writer = null;
            for (String activityName : map.keySet()) {
                // 拿到某个 Activity 中的所有注解
                List<VariableElement> variableElements = map.get(activityName);
                // 拿到包名
                TypeElement enclosingElement = (TypeElement) variableElements.get(0).getEnclosingElement();
                String packageName = processingEnv.getElementUtils().getPackageOf(enclosingElement).toString();
                try {
                    // 开始生成 MainActivity_ViewBinding.java 文件
                    // 创建名为 com.example.dn_butterknife.MainActivity 的.java文件               
                    JavaFileObject sourceFile = filer.createSourceFile(packageName + "." 
                                                                       + activityName + "_ViewBinding");
                    writer = sourceFile.openWriter();
                    // package com.example.dn_butterknife;
                    writer.write("package " + packageName + ";\n");
                    // import com.example.dn_butterknife.IBinder;
                    writer.write("import " + packageName + ".IBinder;\n");
                    // public class MainActivity_ViewBinding implements IBinder
                    // 									<com.example.dn_butterknife.MainActivity>{
                    writer.write("public class " + activityName + "_ViewBinding implements IBinder<" + 
                                 packageName + "." + activityName + ">{\n");
                    // @Override
                    // public void bind(com.example.dn_butterknife.MainActivity target) {
                    writer.write(" @Override\n" +
                            " public void bind(" + packageName + "." + activityName + " target){");
                    // target.tvText=(android.widget.TextView)target.findViewById(2131165325);
                    for (VariableElement variableElement : variableElements) {
                        // 得到名字
                        String variableName = variableElement.getSimpleName().toString();
                        // 得到ID
                        int id = variableElement.getAnnotation(BindView.class).value();
                        // 得到类型
                        TypeMirror typeMirror = variableElement.asType();
                        writer.write("target." + variableName + "=(" + typeMirror + ")target.findViewById(" 
                                     + id + ");\n");
                    }
                    // }}
                    writer.write("\n}}");
                } catch (Exception e) {
                    e.printStackTrace();
                } finally {
                    if (writer != null) {
                        try {
                            writer.close();
                        } catch (Exception e) {
                            e.printStackTrace();
                        }
                    }
                }
            }
        }
        return false;
    }
}


// 在 gradle 文件中添加如下依赖：
// annotationProcessor 'com.google.auto.service:auto-service:1.0-rc4'
// compileOnly 'com.google.auto.service:auto-service:1.0-rc4'
```

