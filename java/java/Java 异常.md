# Java 异常

## 1. 异常继承体系

java中异常体系的核心类是Throwable。他有两个子类Error和Exception。

Error代表一些非常严重的错误。我们一般不必特意在代码中处理他们。

Exception相当于一些小错误。可以用来提示我们出现了什么问题。我们后面主要讲的就是Exception。

异常主要分为两种：

- 运行时异常（编译期间不会去做检查，不需要在代码中进行预处理）
  - 运行时异常都是RuntimeException的子类。例如：NullPointerException、ArrayIndexOutOfBoundsException

![image-20211227011626688](http://badwomen.asia/image-20211227011626688.png)

![image-20211227011649808](http://badwomen.asia/image-20211227011649808.png)



- 编译时异常 （编译时就会做检查，如果一段代码中可能出现编译时异常必须在代码中做预处理）
  - 编译时异常是指非继承自RuntimeException的Exception的子类。例如：FileNotFoundException

![image-20211227011824479](http://badwomen.asia/image-20211227011824479.png)

![image-20211227011855014](http://badwomen.asia/image-20211227011855014.png)

编译时期就无法通过，需要经过我们处理才能通过编译。

## 2.异常处理

### 2.1 throws声明抛出异常

有些时候我们需要把异常抛出，在适当得地方去处理异常。这个时候就可以使用throws抛出异常，把异常交给方法的调用者处理。

格式：

​	在方法声明处加上throws异常类型，如果有多个异常就用逗号隔开，表示抛个调用者去处理这个异常

实例：

抛出一个异常：

~~~java
import java.io.FileInputStream;
import java.io.FileNotFoundException;

public class exc {
    public static void main(String[] args) {
        test();
    }

    public static void test() throws FileNotFoundException {
        FileInputStream fileInputStream = new FileInputStream("a.txt");
    }
}

~~~

![image-20211227014334948](http://badwomen.asia/image-20211227014334948.png)

在main方法中依然会出现异常。因为throw是将异常抛给调用者。

![image-20211227014404739](http://badwomen.asia/image-20211227014404739.png)

main方法是由jvm调用。jvm处理异常的方式是直接打印程序的错误信息然后结束程序。

![image-20211227014448039](http://badwomen.asia/image-20211227014448039.png)

并不会进行打印。

### 2.2 try-catch-finally

另一种处理方式就是进行try-catch捕捉，然后进行异常处理。

```java
import java.io.FileInputStream;
import java.io.FileNotFoundException;

public class exc {
    public static void main(String[] args) {
        test();
        System.out.println(11);
    }

    public static void test() {
        try {
            FileInputStream fileInputStream = new FileInputStream("a.txt");
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } finally {
            System.out.println("finally中打印");
        }
    }
}
```

![image-20211227132642395](http://badwomen.asia/image-20211227132642395.png)

> finally中的代码块无论如何都会被执行。一般用来关闭某些资源

#### 2.2.1 字节码角度看try-catch

简单案例：

```java
public class TryCatch {

    public static void main(String[] args) {
        int i = 0;
        try{
            i = 10;
        }catch (Exception e) {
            i = 20;
        }
    }
}
```

他所生成的字节码对象（截取部分）：

~~~shell
public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: (0x0009) ACC_PUBLIC, ACC_STATIC
    Code:
      stack=1, locals=3, args_size=1
         0: iconst_0 	# 初始化常量0 这个i和我们定义的变量没有任何关系 i = 0
         1: istore_1	# 将0放入局部变量表slot1
         2: bipush        10 # 将一个byte压入操作数栈 
         4: istore_1		# 将操作数栈栈顶元素弹出，放入局部变量表的slot 1中 i = 10
         5: goto          12 #跳转到12
         8: astore_2		# 走到这里说明出现异常 将异常信息放入局部变量表slot2
         9: bipush        20 # 将一个 byte 压入操作数栈 
        11: istore_1		# i = 20
        12: return			# 程序结束
      Exception table:
         from    to  target type
             2     5     8   Class java/lang/Exception
~~~

- 字节码会有一个对应的Exception table的结构，[from, to) 是**前闭后开**（也就是检测2~4行）的检测范围,一旦这个范围的字节码出现了异常，则通过type匹配异常，如果一致，进入 target 所指示行号。这里跳转至第8行
- 8行的字节码指令 astore_2 是将异常对象引用存入局部变量表的2号位置（为e）

> 对应的代码应该很好去理解这段字节码的意思 为了更好的理解我在第一个写了注释 后面都差不太多 如果还不清楚那就是jvm没学好，可以跳过这段去看jvm

**多个single-catch**

~~~java
public class TryCatch {

    public static void main(String[] args) {
        int i = 0;
        try{
            i = 10;
        } catch (ArithmeticException e) {
            i = 20;
        }catch (Exception e) {
            i = 30;
        }
    }
}

~~~

字节码：

~~~shell
public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: (0x0009) ACC_PUBLIC, ACC_STATIC
    Code:
      stack=1, locals=3, args_size=1
         0: iconst_0
         1: istore_1
         2: bipush        10
         4: istore_1
         5: goto          19
         8: astore_2
         9: bipush        20
        11: istore_1
        12: goto          19
        15: astore_2
        16: bipush        30
        18: istore_1
        19: return
      Exception table:
         from    to  target type
             2     5     8   Class java/lang/ArithmeticException
             2     5    15   Class java/lang/Exception
~~~

- 正如上面所说，字节码会将异常信息e放入局部变量表的2号槽位（对应astore_2）
- 因为异常出现时，**只能进入** Exception table 中**一个分支**，所以局部变量表 slot 2 位置**被共用**

**finally**

~~~java
public class TryCatch {

    public static void main(String[] args) {
        int i = 0;
        try {
            i = 10;
        } catch (Exception e) {
            i = 20;
        } finally {
            i = 30;
        }
    }
}

~~~

字节码：

~~~shell
  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: (0x0009) ACC_PUBLIC, ACC_STATIC
    Code:
      stack=1, locals=4, args_size=1
         0: iconst_0
         1: istore_1
         2: bipush        10 	#try块
         4: istore_1			#try块	
         5: bipush        30	#finally块
         7: istore_1			#finally块
         8: goto          27	#跳转
        11: astore_2			#catch块
        12: bipush        20	#catch块
        14: istore_1			#catch块
        15: bipush        30	#finally块
        17: istore_1			#finally块
        18: goto          27	#跳转
        21: astore_3			
        22: bipush        30	
        24: istore_1			
        25: aload_3				
        26: athrow				#抛出异常
        27: return
      Exception table:
         from    to  target type
             2     5    11   Class java/lang/Exception
             2     5    21   any
            11    15    21   any
~~~

可以看到 ﬁnally 中的代码被**复制了 3 份**，分别放入 try 流程，catch 流程以及 catch剩余的异常类型流程

**注意**：虽然从字节码指令看来，每个块中都有finally块，但是finally块中的代码**只会被执行一次**，都会goto到return语句



**finally中的return**

```java
public class TryCatch {

    public static void main(String[] args) {
        int i = TryCatch.test();
        System.out.println(i); // 运行结果20
    }

    public static int test() {
        int i;
        try {
            i = 10;
            return i;
        } finally {
            i = 20;
            return i;
        }
    }
}
```

字节码：

~~~shell
  public static int test();
    descriptor: ()I
    flags: (0x0009) ACC_PUBLIC, ACC_STATIC
    Code:
      stack=1, locals=3, args_size=0
         0: bipush        10
         2: istore_0
         3: iload_0
         4: istore_1	# 暂存返回值
         5: bipush        20
         7: istore_0
         8: iload_0
         9: ireturn		# ireturn会返回操作数栈顶的整型值20
        #  如果出现异常，还是会执行finally块中的内容，没有抛出异常
        10: astore_2
        11: bipush        20
        13: istore_0
        14: iload_0
        15: ireturn # 这里没有athrow了，也就是如果在finally块中如果有返回操作的话，且try块中出现异常，会吞掉异常！
      Exception table:
         from    to  target type
             0     5    10   any
~~~

- 由于 ﬁnally 中的 **ireturn** 被插入了所有可能的流程，因此返回结果肯定以ﬁnally的为准
- 跟上例中的 ﬁnally 相比，发现**没有 athrow 了**，这告诉我们：如果在 ﬁnally 中出现了 return，会**吞掉异常**
- 所以**不要在finally中进行返回操作**

**被吞掉的异常**

来个案例：

~~~java
public class TryCatch {

    public static void main(String[] args) {
        int i = TryCatch.test();
        System.out.println(i);
    }

    public static int test() {
        int i;
        try {
            i = 10/0;
            return i;
        } finally {
            i = 20;
            return i;
        }
    }
}

~~~

![image-20211227142747860](http://badwomen.asia/image-20211227142747860.png)



但如果我们去掉finally中的return：

~~~java
public class TryCatch {

    public static void main(String[] args) {
        int i = TryCatch.test();
        System.out.println(i);
    }

    public static int test() {
        int i;
        try {
            i = 10/0;
            return i;
        } finally {
            i = 20;
//            return i;
        }
    }
}

~~~

![image-20211227142817512](http://badwomen.asia/image-20211227142817512.png)

正常的抛出异常

**finally不带return**

字节码：

~~~shell
  public static int test();
    descriptor: ()I
    flags: (0x0009) ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=3, args_size=0
         0: bipush        10
         2: iconst_0	# 常量0
         3: idiv		#相除
         4: istore_0	#计算结果暂时存入局部变量表slot0
         5: iload_0		#加载
         6: istore_1	#放入slot1 等价于赋值
         7: bipush        20 # 进入finally块
         9: istore_0	# 存入slot0
        10: iload_1		# 加载slot1中的值
        11: ireturn		# 返回栈顶(相当于slot1)的值
        12: astore_2	#异常信息加载
        13: bipush        20
        15: istore_0
        16: aload_2		#加载异常
        17: athrow		#抛出异常
      Exception table:
         from    to  target type
             0     7    12   any
~~~



### 2.3 try-with-resource

JAVA的一大特性就是JVM会对内部资源实现自动回收，即自动GC，给开发者带来了极大的便利。但是JVM对外部资源的引用却无法自动回收，例如数据库连接，网络连接以及输入输出IO流等，这些连接就需要我们手动去关闭，不然会导致外部资源泄露，连接池溢出以及文件被异常占用等。

传统的手动释放外部资源一般放在一般放在try{}catch(){}finally{}机制的finally代码块中，因为finally代码块中语句是肯定会被执行的，即保证了外部资源最后一定会被释放。同时考虑到finally代码块中也有可能出现异常，finally代码块中也有一个try{}catch(){}，这种写法是经典的传统释放外部资源方法，显然是非常繁琐的。

```java
public void FileReader01() {
    // 输出e盘下 story.txt的内容
    String fileName = "e:\\story.txt";
    FileReader fileReader = null;
    try {
         fileReader = new FileReader(fileName);
         int ReadData;
         char[] cbuf = new char[1024];
         while ((ReadData = fileReader.read(cbuf)) != -1) {
             System.out.print(new String(cbuf,0,cbuf.length));
         }
    } catch (IOException e) {
        e.printStackTrace();
    } finally {
        try {
            fileReader.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

JDK1.7之后有了try-with-resource处理机制。首先被自动关闭的资源需要实现Closeable或者AutoCloseable接口，因为只有实现了这两个接口才可以自动调用close()方法去自动关闭资源。写法为try(){}catch(){}，将要关闭的外部资源在try()中创建，catch()捕获处理异常。其实try-with-resource机制是一种语法糖，其底层实现原理仍然是try{}catch(){}finally{}写法，不过在catch(){}代码块中有一个addSuppressed()方法，即异常抑制方法。如果业务处理和关闭连接都出现了异常，业务处理的异常会抑制关闭连接的异常，只抛出处理中的异常，仍然可以通过getSuppressed()方法获得关闭连接的异常。
**使用try-with-resource后：**

~~~java
    @Test
    public void FileReader01() throws FileNotFoundException {
        // 输出e盘下 story.txt的内容
        String fileName = "e:\\story.txt";
        FileReader fileReader = new FileReader(fileName);
        try (fileReader){
            char[] cbuf = new char[1024];
             while (fileReader.read(cbuf) != -1) {
                 System.out.print(new String(cbuf,0,cbuf.length));
             }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
~~~

和传统的try{}catch(){}finally{}机制相比，try-with-resource处理机制有了这个异常抑制方法就是帮助我们简化了关闭连接时出现异常的处理。



## 3.自定义异常

有时我们需要自己去定义一下异常去使用。如果我们需要自定义运行时异常只需要继承RuntimeException，定义构造方法。如果是编译时异常改成继承Exception。

自定义异常类：TooSmallAgeException

~~~java
public class TooSmallAgeException extends RuntimeException{

    public TooSmallAgeException() {
    }

    public TooSmallAgeException(String message) {
        super(message);
    }
}

~~~

![image-20211227154000325](http://badwomen.asia/image-20211227154000325.png)

编译时异常同理。就不展示了。

## 4.异常的作用

1. 异常可以帮助我们知道具体的错误原因。
2. 异常可以让我们在方法调用出现问题的时候，把具体的问题反馈到方法调用处。

