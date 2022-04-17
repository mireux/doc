# SpringBoot AOP

> 通篇是基于注解完成的，xml配置大同小异

## 1 AOP基本使用

### 1.1 准备工作

先搭建一个基本的SpringBoot的框架。

pom.xml

~~~xml
  <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <!--    aop     -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-aop</artifactId>
        </dependency>
        <!--    aop end     -->
    </dependencies>
~~~

cglib包是用来动态代理用的,基于类的代理；
aspectjrt和aspectjweaver是与aspectj相关的包,用来支持切面编程的；
aspectjrt包是aspectj的runtime包；
aspectjweaver是aspectj的织入包；



AopApplication.java

~~~java
package com.example.aop;
import com.example.aop.service.Test1Service;
import com.example.aop.service.Test2Service;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.ConfigurableApplicationContext;

@SpringBootApplication
public class AopApplication {

    public static void main(String[] args) {
        ConfigurableApplicationContext run = SpringApplication.run(AopApplication.class, args);
        Test1Service test1Service = run.getBean(Test1Service.class);
        Test2Service test2Service = run.getBean(Test2Service.class);

        // 调用方法
        test1Service.deleteAll();
        test2Service.GetAll();
    }

}

~~~

Test1Service.java

```java
package com.example.aop.service;
import org.springframework.stereotype.Service;

@Service
public class Test1Service {


    public void deleteAll() {
        System.out.println("deleteAll()的核心业务代码");
    }

}
```

Test2Service.java

```java
package com.example.aop.service;
import org.springframework.stereotype.Service;

@Service
public class Test2Service {

    public void GetAll() {
        System.out.println("GetAll()的核心业务代码");
    }
}
```

> 两个测试类

运行没问题！

![image-20211228015317424](http://badwomen.asia/image-20211228015317424.png)



### 1.2 AOP基本使用

这里展示AOP的基本使用，说明AOP的作用，具体的在后面会一步步将

创建MyAspect.java

```java
package com.example.aop.aop;
import org.aspectj.lang.annotation.After;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;
import org.springframework.stereotype.Component;

@Aspect
@Component
public class MyAspect {

    @Pointcut("execution(* com.example.aop.service.*.*(..))")
    public void pt() {
    }

    @After("pt()")
    public void testAop() {
        System.out.println("方法被增强了");
    }


}
```

这里定义了一个切面，也就是实现AOP的主要程序。

说明：

- 如果要完成AOP程序，加上注解@Aspect、@Component
- AOP需要一个切点，也就是注解@Pointcut，这个注解需要一个方法，而里面的`execution(* com.example.aop.service.*.*(..))`是切点表达式 后面细讲
- @After表示加强方法的位置，后面会详细讲述 这里是在目标方法后进行加强



![image-20211228015933909](http://badwomen.asia/image-20211228015933909.png)

可以看到在每个方法都后面都增加了一条输出语句，也就是我们希望加强的方法。



## 2. AOP的核心概念

- Joinpoint(连接点)：所谓连接点是指可以被增强到的点。在Spring中，这些点是指方法，比如案例中deleteAll()或者GetAll() 这些方法，都是连接点。因为Spring只支持方法类型的连接点。
- Pointcut(切点)：所谓切入点是指被增强的连接点。比如案例中deleteAll()或者GetAll()，这些方法被增强了，就是切点。
- Advice(通知/增强)：所谓通知是指被具体增强的代码。
- Target(目标对象)：被增强的对象就是目标对象
- Aspect(切面)：是切入点和通知的结合。这里就是MyAspect类
- Proxy(代理)：一个类被AOP增强之后，就产生了一个代理类

关于代理类做一个小小的证明

![image-20211228151525731](http://badwomen.asia/image-20211228151525731.png)

我们通过断点调试可以看出，实际返回的对象并不单单是一个Test1Service。

如果我们去掉@Aspect注解，就只是一个Test1Service类了。

![image-20211228151631738](http://badwomen.asia/image-20211228151631738.png)



## 3. 切点的确定

确定切点，也就是说要确定哪些方法需要被增强。这里有两种方式。一个是切点表达式，一个是切点函数。

### 3.1 切点表达式

**写法：** execution({修饰符} 返回值类型 包名.类名.方法名.(参数))

- 访问修饰符可以省略
- 返回值类型、包名、类名、方法名可以使用*表示任意
- 包名与类名之间一个点.，表示当前包下的任意类，两个点..表示当前包及其子包下的类
- 参数列表可以使用两个点..表示任意参数个数，任意类型的参数列表



举个例子：

~~~mar
execution(* com.example.aop.service.*.*(..))  com.example.aop.service包下的任意类，方法名任意，参数列表任意，返回值任意 

execution(* com.example.aop.service..*.*(..)) com.example.aop.service包及其子包下的任意类，方法名任意，参数列表任意，返回值任意 

execution(* com.example.aop.service.*.*()) com.example.aop.service包下的任意类，方法名任意，参数列表为空，返回值任意

execution(* com.example.aop.service.*.delete*(..)) com.example.aop.service包下的任意类，方法名以delete开头的，参数列表任意，返回值任意
~~~

> 可以自行去做实验 这里就不多做演示。



### 3.2 切点函数

我们也可以在要增强的方法上添加注解，然会在@Pointcut中使用@annotation表示对加了什么注解的方式进行增强。



举个例子：

新建一个注解AspectLog

~~~java
package com.example.aop.aop;


import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface AspectLog {  
}

~~~



写上我们的切点表达式：

~~~java
    @Pointcut("@annotation(com.example.aop.aop.AspectLog)")
	public void pt() {
    }
~~~

会去寻找带有@AspectLog注解的切点。

如果我们这时候不加任何注解

![image-20211228154504603](http://badwomen.asia/image-20211228154504603.png)

没有方法被增强。

我们在Test1Service中的deleteAll()上加上注解@ApsectLog

```java
@Service
public class Test1Service {

    @AspectLog
    public void deleteAll() {
        System.out.println("deleteAll()的核心业务代码");
    }
    
}
```

![image-20211228154602306](http://badwomen.asia/image-20211228154602306.png)

就只有deleteAll()方法被增强了。

这种方式更加灵活。



## 4. 通知分类

- @Before：前置通知，在方法执行前执行
- @AfterReturning：返回后通知，在目标方法执行后执行，如果出现异常则不会执行
- @After：后置通知，在目标方法返回结果之后执行，无论是否异常都会执行
- @AfterThrowing：异常通知，在目标方法抛出异常后执行
- @Around：环绕通知，围绕着方法通知。(最重要的通知)

下面一段伪代码能更好的理解不同通知执行的时机。（**注意！注意！注意！注意！**下面的伪代码是理解单个通知的执行时机的，但实际上如果你想组合使用，和下面的时机的执行是不一样的！！如果想要组合使用通知更建议使用@Around，清晰且好用）

~~~java
public Object test() {
    before(); // @Before 前置通知
    try {
        Object res = 目标方法();
        afterReturning(); // @AfterReturing 返回通知
    } catch(Throwable throwable) {
        throwable.printStackTrace();
        afterThrowing(); // @AfterThrowing 异常通知
    }finally {
        after(); // @After 后置通知
    }
    return res;
}
~~~



### 4.1 @Before

> 为了方便调试查看我们只增强一个方法

```java
@Before("pt()")
public void testAop() {
    System.out.println("before");
}
```

![image-20211228162116291](http://badwomen.asia/image-20211228162116291.png)

### 4.2 @AfterReturning

```java
@AfterReturning("pt()")
public void testAop() {
    System.out.println("AfterReturing");
}
```

![image-20211228162231800](http://badwomen.asia/image-20211228162231800.png)

> 但如果有异常 那么就不会增强方法

```java
@Service
public class Test1Service {

    @AspectLog
    public void deleteAll() {
        int i = 10 / 0; // 加入异常
        System.out.println("deleteAll()的核心业务代码");
    }
}
```

![image-20211228162327599](http://badwomen.asia/image-20211228162327599.png)

### 4.3 @AfterThrowing

```java
@AfterThrowing("pt()")
public void testAop() {
    System.out.println("AfterThrowing");
}
```

![image-20211228162445784](http://badwomen.asia/image-20211228162445784.png)

> 如果没有异常就不会报错

![image-20211228162553878](http://badwomen.asia/image-20211228162553878.png)

### 4.4 @After

>  无论有无异常都会输出

```java
@After("pt()")
public void testAop() {
    System.out.println("After");
}
```

![image-20211228162628567](http://badwomen.asia/image-20211228162628567.png)

![image-20211228162642367](http://badwomen.asia/image-20211228162642367.png)

### 4.5 @Around

环绕通知就比较特别。

如果我们这样写：

```java
@Around("pt()")
public void testAop() {
    System.out.println("Around");
}
```

运行你会只调用了增强方法，而目标方法没有调用。

![image-20211228162917858](http://badwomen.asia/image-20211228162917858.png)

我们需要在方法参数加入ProceedingJoinPoint

```java
@Around("pt()")
public void testAop(ProceedingJoinPoint pjp) {
    System.out.println("Around"); // 目标方法前
    try {
        pjp.proceed(); // 目标方法执行
     //     System.out.println("Around"); // 目标方法后	
    } catch (Throwable e) {
        e.printStackTrace();
    }
}
```

这样就可以目标方法执行，而且可以自定义增强的位置。

![image-20211228163130674](http://badwomen.asia/image-20211228163130674.png)



## 5. 获取被增强方法的相关信息

其实就是通过反射去获取的。

除了环绕通知之外的所有通知方法中增加一个方法参数**JoinPoint类型**。这个参数封装了被增强方法的相关信息。**我们可以通过这个参数获取到除了异常对象和返回值之外的所有信息**

举个例子：

```java
@AfterReturning(value = "pt()",returning = "res")
public void testAop(JoinPoint joinPoint,Object res) {
    MethodSignature signature = (MethodSignature) joinPoint.getSignature(); // 实际上getSignature()底层返回的就是MethodSignature
    System.out.println("方法参数:" + Arrays.toString(joinPoint.getArgs()));
    System.out.println("方法签名：" + signature);
    System.out.println("返回的结果是：" + res);
}
```

> 也只有AfterReturning能获取返回结果，AfterThrowing能获取异常结果

![image-20211228171221769](http://badwomen.asia/image-20211228171221769.png)



### 5.1 环绕通知获取被增强的相关信息

使用环绕通知，你可以获取所有的信息，不同于上面的需要分类。ProceedingJoinPoint类型封装了被增强方法的相关信息。

该参数的proceed()方法被调用相当于被增强方法被执行，调用后的返回值就是被调用方法的返回值。

例如：

```java
@Around("pt()")
public void testAop(ProceedingJoinPoint pjp) {
    Object[] args = pjp.getArgs();
    MethodSignature signature = (MethodSignature) pjp.getSignature();
    Object target = pjp.getTarget();
    Object res = null;
    try {
        res = pjp.proceed();
        System.out.println(Arrays.toString(args));
        System.out.println(signature);
        System.out.println(target.getClass().getName());
        System.out.println(res);
    } catch (Throwable e) {
        System.out.println(e);
        e.printStackTrace();
    }
}
```

> 提示： 如果被增强方法有返回值，那么环绕通知也需要有返回值 否则就会接收不到返回值

## 6. AOP应用案例

完成一个打印日志的AOP切面

目标是这样的

![image-20211228194217603](http://badwomen.asia/image-20211228194217603.png)



定义注解@LogAnnotation

~~~java
package com.example.aop.aop.LogAspect;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface LogAnnotation {
    // 哪个模块
    String module() default "";
    // 哪个操作
    String operation() default "";
}
~~~

定义切面类LogAnnotation

~~~java
package com.example.aop.aop.LogAspect;



import com.alibaba.fastjson.JSON;
import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.Signature;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;
import org.aspectj.lang.reflect.MethodSignature;
import org.springframework.stereotype.Component;

import java.lang.annotation.Annotation;
import java.lang.reflect.Method;

@Aspect
@Component
@Slf4j
public class LogAspect {

    @Pointcut("@annotation(com.example.aop.aop.LogAspect.LogAnnotation)")
    public void pt() {

    }

    @Around("pt()")
    public Object log(ProceedingJoinPoint pjp) throws Throwable {
        long startTime = System.currentTimeMillis();
        Object proceed = pjp.proceed();
        long endTime = System.currentTimeMillis();
        // 执行市场
        long totalTime = endTime - startTime;
        recordLog(pjp,totalTime);
        return proceed;
    }

    // 输出日志的方法
    private void recordLog(ProceedingJoinPoint pjp, long totalTime) {
        MethodSignature signature = (MethodSignature) pjp.getSignature();
        Method method = signature.getMethod();
        LogAnnotation logAnnotation = method.getAnnotation(LogAnnotation.class);
        log.info("=====================log start================================");
        log.info("module:{}", logAnnotation.module());
        log.info("operation:{}", logAnnotation.operation());

        // 请求的方法名
        String name = pjp.getTarget().getClass().getName();
        String methodName = signature.getName();
        log.info("request method:{}", name + "." + methodName + "()");

        // 方法参数
        Object[] args = pjp.getArgs();
        if (args.length != 0) {
            String params = JSON.toJSONString(args[0]);
            log.info("params:{}", params);
        } else {
            log.info("params:");
        }

        log.info("execute time : {} ms", totalTime);
        log.info("=====================log end================================");
    }
}
~~~

![image-20211228200205447](http://badwomen.asia/image-20211228200205447.png)

