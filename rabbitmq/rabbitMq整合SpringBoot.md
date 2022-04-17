# SpringBoot整合RabbitMq

通过整合其实可以更好的了解RabbitMq过程和一些组件的使用。

## RabbitMq五种模型

RabbitMq其实提供了六种模型，但最后一种是RPC模型，已经不属于消息队列，所以就不展开讲了。

其他五种模型分别是：Hello World模型（基本模型）、work模型、Fanout模型、Direct模型、Topic模型

其中前两个是不基于交换机的，简单模型。

后三种基于交换机。

另外还有 Header Exchange 头交换机 ，Default Exchange 默认交换机，Dead Letter Exchange 死信交换机，这几个该篇暂不做讲述。



## SpringBoot整合RabbitMq

本次实例教程需要创建2个springboot项目，一个 rabbitmq-provider （生产者），一个rabbitmq-consumer（消费者）。

首先创建 rabbitmq-provider，

pom.xml里用到的jar依赖：

```xml
 		<!--rabbitmq-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-amqp</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
```

然后application.yml：

```yml
server:
  port: 8888

spring:
  rabbitmq:
    host: localhost 
    port: 5672
    username: lhj 
    password: 123
    virtual-host: /test
    publisher-confirm-type: correlated #是否开启确认模式 设置此属性配置可以确保消息成功发送到交换器
    publisher-returns: true #是否开启回调函数 可以确保消息在未被队列接收时返回
```



### direct exchange （直连交换机）

创建DirectRabbitConfig.java

```java
import org.springframework.amqp.core.*;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class DirectRabbitConfig {

    

    // 队列 起名：TestDirectQueue
    @Bean
    public Queue TestDirectQueue() {
        // durable:是否持久化,默认是false,持久化队列：会被存储在磁盘上，当消息代理重启时仍然存在，暂存队列：当前连接有效
        // exclusive:默认也是false，只能被当前创建的连接使用，而且当连接关闭后队列即被删除。此参考优先级高于durable
        // autoDelete:是否自动删除，当没有生产者或者消费者使用此队列，该队列会自动删除。
        //   return new Queue("TestDirectQueue",true,true,false);

        //一般设置一下队列的持久化就好,其余两个就是默认false
        return new Queue("TestDirectQueue",true,false,false);
    }


    // Direct交换机 起名：TestDirectExchange
    @Bean
    DirectExchange TestDirectExchange() {
        return new DirectExchange("TestDirectExchange",true,false);
    }

    // 绑定 将队列和交换机绑定 并设置用于匹配键 ： TestDirectRouting
    @Bean
    Binding bindingDirect() {
        return BindingBuilder.bind(TestDirectQueue()).to(TestDirectExchange()).with("TestDirectRouting");
    }


    @Bean
    DirectExchange lonelyDirectExchange() {
        return new DirectExchange("lonelyDirectExchange");
    }




}
```

然后写个简单的接口进行消息推送（根据需求也可以改为定时任务等等，具体看需求），SendMessageController.java：

```java
    @GetMapping("/sendDirectMessage")
    public String sendDirectMessage() {
        String messageId = String.valueOf(UUID.randomUUID());
        String messageData = "test message,hello!";
        String createTime = LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"));
        Map<String,Object> map = new HashMap<>();
        map.put("messageId",messageId);
        map.put("messageData",messageData);
        map.put("createTime",createTime);
        //将消息携带绑定键值：TestDirectRouting 发送到交换机TestDirectExchange
        rabbitTemplate.convertAndSend("TestDirectExchange", "TestDirectRouting", map);
        return "sendDirectMessage ok";
    }
```

测试接口：

![image-20211208164538478](http://badwomen.asia/image-20211208164538478.png)

![image-20211208164633320](http://badwomen.asia/image-20211208164633320.png)

接下来，创建rabbitmq-consumer项目：

pom.xml里的jar依赖：

```xml
    <!--rabbitmq-->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-amqp</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
    </dependency>
```
然后是 application.yml：

```yaml
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: lhj
    password: 123
    virtual-host: /test
    publisher-confirm-type: correlated
    publisher-returns: true
server:
  port: 8787
```

然后一样，创建DirectRabbitConfig.java（**消费者单纯的使用，其实可以不用添加这个配置，直接建后面的监听就好，使用注解来让监听器监听对应的队列即可。配置上了的话，其实消费者也是生成者的身份，也能推送该消息。**）：

```java
import org.springframework.amqp.core.Binding;
import org.springframework.amqp.core.BindingBuilder;
import org.springframework.amqp.core.DirectExchange;
import org.springframework.amqp.core.Queue;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class DirectRabbitConfig {
 
    //队列 起名：TestDirectQueue
    @Bean
    public Queue TestDirectQueue() {
        return new Queue("TestDirectQueue",true,false,false);
    }
 
    //Direct交换机 起名：TestDirectExchange
    @Bean
    DirectExchange TestDirectExchange() {
        return new DirectExchange("TestDirectExchange");
    }
 
    //绑定  将队列和交换机绑定, 并设置用于匹配键：TestDirectRouting
    @Bean
    Binding bindingDirect() {
        return BindingBuilder.bind(TestDirectQueue()).to(TestDirectExchange()).with("TestDirectRouting");
    }
}
```

然后是创建消息接收监听类，DirectReceiver.java：

```java

@Component
@RabbitListener(queues = "TestDirectQueue")
public class DirectConsumer {
    @RabbitHandler
    public void process(Map testMessage) {
        System.out.println("第一个DirectReceiver消费者收到消息: " + testMessage.toString());
    }
}

```

然后将rabbitmq-consumer项目运行起来，可以看到把之前推送的那条消息消费下来了：

![image-20211208165005845](http://badwomen.asia/image-20211208165005845.png)

直连交换机是一对一，那如果咱们配置多台监听绑定到同一个直连交互的同一个队列，则会轮询方式进行消费

而且不存在重复消费

![image-20211208165120588](http://badwomen.asia/image-20211208165120588.png)



基于注解：

相当于只是单纯的使用

```java
package com.example.rabbit.Consumer;


import org.springframework.amqp.rabbit.annotation.*;
import org.springframework.stereotype.Component;

import java.util.Map;

@Component
//@RabbitListener(queues = "TestDirectQueue")
public class DirectConsumer {
//    @RabbitHandler
    @RabbitListener(bindings = @QueueBinding(
            value = @Queue(name = "TestDirectQueue"),
            exchange = @Exchange(value = "TestDirectExchange",type = "direct")
    ))
    public void process(Map testMessage) {
        System.out.println("第一个DirectReceiver消费者收到消息: " + testMessage.toString());
    }
}

```

效果是一样的，就不上效果图了。



## topic exchange（主题交换机）



在rabbitmq-provider项目里面创建TopicRabbitConfig.java：

```java
package com.example.rabbitproducer.config;


import org.springframework.amqp.core.Binding;
import org.springframework.amqp.core.BindingBuilder;
import org.springframework.amqp.core.Queue;
import org.springframework.amqp.core.TopicExchange;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class TopicRabbitConfig {

    //绑定键
    public final static String man = "topic.man";
    public final static String woman = "topic.woman";

    @Bean
    public Queue firstQueue() {
        return new Queue(TopicRabbitConfig.man);
    }

    @Bean
    public Queue secondQueue() {
        return new Queue(TopicRabbitConfig.woman);
    }

    @Bean
    TopicExchange exchange() {
        return new TopicExchange("topicExchange");
    }


    //将firstQueue和topicExchange绑定,而且绑定的键值为topic.man
    //这样只要是消息携带的路由键是topic.man,才会分发到该队列
    @Bean
    Binding bindingExchangeMessage() {
        return BindingBuilder.bind(firstQueue()).to(exchange()).with(man);
    }

    //将secondQueue和topicExchange绑定,而且绑定的键值为用上通配路由键规则topic.#
    // 这样只要是消息携带的路由键是以topic.开头,都会分发到该队列
    @Bean
    Binding bindingExchangeMessage2() {
        return BindingBuilder.bind(secondQueue()).to(exchange()).with("topic.#");
    }



}

```

测试接口：

```java
    @GetMapping("/sendTopicMessage1")
    public String sendTopicMessage1() {
        String messageId = String.valueOf(UUID.randomUUID());
        String messageData = "message: M A N ";
        String createTime = LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"));
        Map<String, Object> manMap = new HashMap<>();
        manMap.put("messageId", messageId);
        manMap.put("messageData", messageData);
        manMap.put("createTime", createTime);
        rabbitTemplate.convertAndSend("topicExchange", "topic.man", manMap);
        return "sendTopicMessage1 ok";
    }

    @GetMapping("/sendTopicMessage2")
    public String sendTopicMessage2() {
        String messageId = String.valueOf(UUID.randomUUID());
        String messageData = "message: woman is all ";
        String createTime = LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"));
        Map<String, Object> womanMap = new HashMap<>();
        womanMap.put("messageId", messageId);
        womanMap.put("messageData", messageData);
        womanMap.put("createTime", createTime);
        rabbitTemplate.convertAndSend("topicExchange", "topic.woman", womanMap);
//        rabbitTemplate.convertAndSend("topicExchange","topic.woman","testtesting");
        return "sendTopicMessage2 ok";
    }
```

测试：

![image-20211208170304416](http://badwomen.asia/image-20211208170304416.png)



在Consumer中

TopicRabbitConfig.java（同理于上，基于注解也是可以直接使用的）

```java
package com.example.rabbit.config;

import org.springframework.amqp.core.Binding;
import org.springframework.amqp.core.BindingBuilder;
import org.springframework.amqp.core.Queue;
import org.springframework.amqp.core.TopicExchange;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class TopicRabbitConfig {
    //绑定键
    public final static String man = "topic.man";
    public final static String woman = "topic.woman";
 
    @Bean
    public Queue firstQueue() {
        return new Queue(TopicRabbitConfig.man);
    }
 
    @Bean
    public Queue secondQueue() {
        return new Queue(TopicRabbitConfig.woman);
    }
 
    @Bean
    TopicExchange exchange() {
        return new TopicExchange("topicExchange");
    }
 
 
    //将firstQueue和topicExchange绑定,而且绑定的键值为topic.man
    //这样只要是消息携带的路由键是topic.man,才会分发到该队列
    @Bean
    Binding bindingExchangeMessage() {
        return BindingBuilder.bind(firstQueue()).to(exchange()).with(man);
    }
 
    //将secondQueue和topicExchange绑定,而且绑定的键值为用上通配路由键规则topic.#
    // 这样只要是消息携带的路由键是以topic.开头,都会分发到该队列
    @Bean
    Binding bindingExchangeMessage2() {
        return BindingBuilder.bind(secondQueue()).to(exchange()).with("topic.#");
    }
 
}

```



TopicManReciver.java

```java
package com.example.rabbit.Consumer;

import org.springframework.amqp.rabbit.annotation.RabbitHandler;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.stereotype.Component;

import java.util.Map;

@Component
@RabbitListener(queues = "topic.man")
public class TopicManReceiver {
 
    @RabbitHandler
    public void process(Map testMessage) {
        System.out.println("TopicManReceiver消费者收到消息  : " + testMessage.toString());
    }
}

```



TopicTotalReciver.java

```java
package com.example.rabbit.Consumer;

import org.springframework.amqp.rabbit.annotation.RabbitHandler;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.stereotype.Component;

import java.util.Map;

@Component
@RabbitListener(queues = "topic.woman")
public class TopicTotalReceiver {
 
    @RabbitHandler
    public void process(Map testMessage) {
        System.out.println("TopicTotalReceiver消费者收到Map消息  : " + testMessage.toString());
    }

    
}

```



结果：

1.当访问sendTopicMessage1接口时

![image-20211208170326506](http://badwomen.asia/image-20211208170326506.png)

2.当访问sendTopicMessage2接口时

![image-20211208170753110](http://badwomen.asia/image-20211208170753110.png)



**基于注解：**

```java
package com.example.rabbit.Consumer;

import org.springframework.amqp.rabbit.annotation.*;
import org.springframework.stereotype.Component;

import java.util.Map;

@Component
//@RabbitListener(queues = "topic.man")
public class TopicManReceiver {
 
//    @RabbitHandler
    @RabbitListener(bindings = @QueueBinding(
            value = @Queue(name = "topic.man"),
            exchange = @Exchange(name = "topicExchange",type = "topic"),
        	key = {topic.man}
    ))
    // key 中 *表示一个字符
    public void process(Map testMessage) {
        System.out.println("TopicManReceiver消费者收到消息  : " + testMessage.toString());
    }
}
```





## Fanout exchange（扇形交换机）

同样地，先在rabbitmq-provider项目上创建FanoutRabbitConfig.java：

```java
package com.example.rabbitproducer.config;


import org.springframework.amqp.core.Binding;
import org.springframework.amqp.core.BindingBuilder;
import org.springframework.amqp.core.FanoutExchange;
import org.springframework.amqp.core.Queue;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class FanoutRabbitConfig {

    /**
     *  创建三个队列 ：fanout.A   fanout.B  fanout.C
     *  将三个队列都绑定在交换机 fanoutExchange 上
     *  因为是扇型交换机, 路由键无需配置,配置也不起作用
     */
    @Bean
    public Queue queueA() {
        return new Queue("fanout.A");
    }

    @Bean
    public Queue queueB() {
        return new Queue("fanout.B");
    }

    @Bean
    public Queue queueC() {
        return new Queue("fanout.C");
    }

    @Bean
    FanoutExchange fanoutExchange() {
        return new FanoutExchange("fanoutExchange");
    }

    @Bean
    Binding bindingExchangeA() {
        return BindingBuilder.bind(queueA()).to(fanoutExchange());
    }

    @Bean
    Binding bindingExchangeB() {
        return BindingBuilder.bind(queueB()).to(fanoutExchange());
    }

    @Bean
    Binding bindingExchangeC() {
        return BindingBuilder.bind(queueC()).to(fanoutExchange());
    }
    
}
```



测试接口：

```java
@GetMapping("/sendFanoutMessage")
public String sendFanoutMessage() {
    String messageId = String.valueOf(UUID.randomUUID());
    String messageData = "message: testFanoutMessage ";
    String createTime = LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"));
    Map<String, Object> map = new HashMap<>();
    map.put("messageId", messageId);
    map.put("messageData", messageData);
    map.put("createTime", createTime);
    rabbitTemplate.convertAndSend("fanoutExchange", null, map);
    return "sendFanoutMessage  ok";
}
```



接着在Consumer项目中：

FanoutReceiverA.java：

```java
package com.example.rabbit.Consumer;

import org.springframework.amqp.rabbit.annotation.RabbitHandler;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.stereotype.Component;

import java.util.Map;

@Component
@RabbitListener(queues = "fanout.A")
public class FanoutReceiverA {
 
    @RabbitHandler
    public void process(Map testMessage) {
        System.out.println("FanoutReceiverA消费者收到消息  : " +testMessage.toString());
    }
 
}
```



FanoutReceiverB.java：

```java
package com.example.rabbit.Consumer;

import org.springframework.amqp.rabbit.annotation.RabbitHandler;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.stereotype.Component;

import java.util.Map;

@Component
@RabbitListener(queues = "fanout.B")
public class FanoutReceiverB {
 
    @RabbitHandler
    public void process(Map testMessage) {
        System.out.println("FanoutReceiverB消费者收到消息  : " +testMessage.toString());
    }
 
}
```

FanoutReceiverC.java：

```java
package com.example.rabbit.Consumer;

import org.springframework.amqp.rabbit.annotation.RabbitHandler;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.stereotype.Component;

import java.util.Map;

@Component
@RabbitListener(queues = "fanout.C")
public class FanoutReceiverC {
 
    @RabbitHandler
    public void process(Map testMessage) {
        System.out.println("FanoutReceiverC消费者收到消息  : " +testMessage.toString());
    }
 
}
```

同上，注解形式也是可以的

```java
package com.example.rabbit.config;


import org.springframework.amqp.core.Binding;
import org.springframework.amqp.core.BindingBuilder;
import org.springframework.amqp.core.FanoutExchange;
import org.springframework.amqp.core.Queue;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class FanoutRabbitConfig {

    /**
     *  创建三个队列 ：fanout.A   fanout.B  fanout.C
     *  将三个队列都绑定在交换机 fanoutExchange 上
     *  因为是扇型交换机, 路由键无需配置,配置也不起作用
     */
    @Bean
    public Queue queueA() {
        return new Queue("fanout.A");
    }

    @Bean
    public Queue queueB() {
        return new Queue("fanout.B");
    }

    @Bean
    public Queue queueC() {
        return new Queue("fanout.C");
    }

    @Bean
    FanoutExchange fanoutExchange() {
        return new FanoutExchange("fanoutExchange");
    }

    @Bean
    Binding bindingExchangeA() {
        return BindingBuilder.bind(queueA()).to(fanoutExchange());
    }

    @Bean
    Binding bindingExchangeB() {
        return BindingBuilder.bind(queueB()).to(fanoutExchange());
    }

    @Bean
    Binding bindingExchangeC() {
        return BindingBuilder.bind(queueC()).to(fanoutExchange());
    }
}
```

![image-20211208172659717](http://badwomen.asia/image-20211208172659717.png)



**基于注解：**

```java
package com.example.rabbit.Consumer;

import org.springframework.amqp.rabbit.annotation.*;
import org.springframework.stereotype.Component;

import java.util.Map;

@Component
//@RabbitListener(queues = "fanout.A")
public class FanoutReceiverA {
 
//    @RabbitHandler
    @RabbitListener(bindings = @QueueBinding(
            value = @Queue(name = "fanout.A"),
            exchange = @Exchange(name = "fanoutExchange",type = "fanout")
    )
    )
    public void process(Map testMessage) {
        System.out.println("FanoutReceiverA消费者收到消息  : " +testMessage.toString());
    }
 
}
```







## 确认机制

到了这里其实三个常用的交换机的使用我们已经完毕了，那么接下来我们继续讲讲消息的回调，其实就是消息确认（生产者推送消息成功，消费者接收消息成功）。这样才能保证消息不丢失。

在rabbitmq-provider项目的application.yml文件上，加上消息确认的配置项后

```yaml
  #确认消息已发送到交换机(Exchange)
    publisher-confirms: true
    #确认消息已发送到队列(Queue)
    publisher-returns: true
```

然后是配置相关的消息确认回调函数，RabbitConfig.java：

```java
package com.example.rabbit.config;


import org.springframework.amqp.core.ReturnedMessage;
import org.springframework.amqp.rabbit.connection.ConnectionFactory;
import org.springframework.amqp.rabbit.connection.CorrelationData;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class RabbitConfig {


    @Bean
    public RabbitTemplate createRabbitTemplate(ConnectionFactory connectionFactory) {
        RabbitTemplate rabbitTemplate = new RabbitTemplate();
        rabbitTemplate.setConnectionFactory(connectionFactory);
        //设置开启Mandatory,才能触发回调函数,无论消息推送结果怎么样都强制调用回调函数
        rabbitTemplate.setMandatory(true);

        rabbitTemplate.setConfirmCallback(new RabbitTemplate.ConfirmCallback() {
            @Override
            public void confirm(CorrelationData correlationData, boolean ack, String cause) {
                System.out.println("ConfirmCallback:     " + "相关数据：" + correlationData);
                System.out.println("ConfirmCallback:     " + "确认情况：" + ack);
                System.out.println("ConfirmCallback:     " + "原因：" + cause);
            }
        });

        rabbitTemplate.setReturnsCallback(new RabbitTemplate.ReturnsCallback() {
            @Override
            public void returnedMessage(ReturnedMessage returnedMessage) {
                System.out.println("ReturnCallback:     " + "消息：" + returnedMessage.toString());
            }
        });
        return rabbitTemplate;
    }


}
```

到这里，生产者推送消息的消息确认调用回调函数已经完毕。
可以看到上面写了两个回调函数，一个叫 ConfirmCallback ，一个叫 RetrunCallback；
那么以上这两种回调函数都是在什么情况会触发呢？

先从总体的情况分析，推送消息存在四种情况：

①消息推送到server，但是在server里找不到交换机
②消息推送到server，找到交换机了，但是没找到队列
③消息推送到sever，交换机和队列啥都没找到
④消息推送成功

那么我先写几个接口来分别测试和认证下以上4种情况，消息确认触发回调函数的情况：



#### 消息推送到server，但是在server里找不到交换机

写个测试接口，把消息推送到名为‘non-existent-exchange’的交换机上（这个交换机是没有创建没有配置的）：

```java
//1.消息推送到server，但是在server里找不到交换机
@GetMapping("/TestMessageAck")
public String TestMessageAck() {
    String messageId = String.valueOf(UUID.randomUUID());
    String messageData = "message: non-existent-exchange test message ";
    String createTime = LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"));
    Map<String, Object> map = new HashMap<>();
    map.put("messageId", messageId);
    map.put("messageData", messageData);
    map.put("createTime", createTime);
    rabbitTemplate.convertAndSend("non-existent-exchange", "TestDirectRouting", map);
    return "TestMessageAck ok";
}
```



![image-20211208173659188](http://badwomen.asia/image-20211208173659188.png)

  结论： ①这种情况触发的是 ConfirmCallback 回调函数。



#### 消息推送到server，找到交换机了，但是没找到队列

这种情况就是需要新增一个交换机，但是不给这个交换机绑定队列，我来简单地在DirectRabitConfig里面新增一个直连交换机，名叫‘lonelyDirectExchange’，但没给它做任何绑定配置操作：

```java
@Bean
DirectExchange lonelyDirectExchange() {
    return new DirectExchange("lonelyDirectExchange");
}
```
然后写个测试接口，把消息推送到名为‘lonelyDirectExchange’的交换机上（这个交换机是没有任何队列配置的）：

```java
@GetMapping("/TestMessageAck2")
public String TestMessageAck2() {
    String messageId = String.valueOf(UUID.randomUUID());
    String messageData = "message: lonelyDirectExchange test message ";
    String createTime = LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"));
    Map<String, Object> map = new HashMap<>();
    map.put("messageId", messageId);
    map.put("messageData", messageData);
    map.put("createTime", createTime);
    rabbitTemplate.convertAndSend("lonelyDirectExchange", "TestDirectRouting", map);
    return "ok";
}
```
调用接口，查看rabbitmq-provuder项目的控制台输出情况：


![image-20211208173756135](http://badwomen.asia/image-20211208173756135.png)

可以看到这种情况，两个函数都被调用了；
这种情况下，消息是推送成功到服务器了的，所以ConfirmCallback对消息确认情况是true；
而在RetrunCallback回调函数的打印参数里面可以看到，消息是推送到了交换机成功了，但是在路由分发给队列的时候，找不到队列，所以报了错误 NO_ROUTE 。
  结论：②这种情况触发的是 ConfirmCallback和RetrunCallback两个回调函数。

#### **消息推送到sever，交换机和队列啥都没找到** 

这种情况其实一看就觉得跟①很像，没错 ，③和①情况回调是一致的，所以不做结果说明了。
 结论： ③这种情况触发的是 ConfirmCallback 回调函数。



#### **消息推送成功**

那么测试下，按照正常调用之前消息推送的接口就行，就调用下 /sendFanoutMessage接口，可以看到控制台输出：

![image-20211208173909933](http://badwomen.asia/image-20211208173909933.png)

结论： ④这种情况触发的是 ConfirmCallback 回调函数。


以上是生产者推送消息的消息确认 回调函数的使用介绍（可以在回调函数根据需求做对应的扩展或者业务数据处理）。





**接下来我们继续， 消费者接收到消息的消息确认机制。**

和生产者的消息确认机制不同，因为消息接收本来就是在监听消息，符合条件的消息就会消费下来。
所以，消息接收的确认机制主要存在三种模式：

①自动确认， 这也是默认的消息确认情况。  AcknowledgeMode.NONE
RabbitMQ成功将消息发出（即将消息成功写入TCP Socket）中立即认为本次投递已经被正确处理，不管消费者端是否成功处理本次投递。
所以这种情况如果消费端消费逻辑抛出异常，也就是消费端没有处理成功这条消息，那么就相当于丢失了消息。
一般这种情况我们都是使用try catch捕捉异常后，打印日志用于追踪数据，这样找出对应数据再做后续处理。

② 根据情况确认， 这个不做介绍

③ 手动确认 ， 这个比较关键，也是我们配置接收消息确认机制时，多数选择的模式。
消费者收到消息后，手动调用basic.ack/basic.nack/basic.reject后，RabbitMQ收到这些消息后，才认为本次投递成功。
basic.ack用于肯定确认 
basic.nack用于否定确认（注意：这是AMQP 0-9-1的RabbitMQ扩展） 
basic.reject用于否定确认，但与basic.nack相比有一个限制:一次只能拒绝单条消息 



消费者端以上的3个方法都表示消息已经被正确投递，但是basic.ack表示消息已经被正确处理。
而basic.nack,basic.reject表示没有被正确处理：

着重讲下reject，因为有时候一些场景是需要重新入列的。

`channel.basicReject(deliveryTag, true); `

拒绝消费当前消息，如果第二参数传入true，就是将数据重新丢回队列里，那么下次还会消费这消息。设置false，就是告诉服务器，我已经知道这条消息数据了，因为一些原因拒绝它，而且服务器也把这个消息丢掉就行。 下次不想再消费这条消息了。

使用拒绝后重新入列这个确认模式要谨慎，因为一般都是出现异常的时候，catch异常再拒绝入列，选择是否重入列。

但是如果使用不当会导致一些每次都被你重入列的消息一直消费-入列-消费-入列这样循环，会导致消息积压。

 

顺便也简单讲讲 nack，这个也是相当于设置不消费某条消息。

`channel.basicNack(deliveryTag, false, true);`
第一个参数依然是当前消息到的数据的唯一id;
第二个参数是指是否针对多条消息；如果是true，也就是说一次性针对当前通道的消息的tagID小于当前这条消息的，都拒绝确认。
第三个参数是指是否重新入列，也就是指不确认的消息是否重新丢回到队列里面去。

同样使用不确认后重新入列这个确认模式要谨慎，因为这里也可能因为考虑不周出现消息一直被重新丢回去的情况，导致积压。

在消费者项目里，
新建MessageListenerConfig.java上添加代码相关的配置代码：

```java
package com.example.rabbit.config;

import com.example.rabbit.MyAckReceiver;
import org.springframework.amqp.core.AcknowledgeMode;
import org.springframework.amqp.rabbit.connection.CachingConnectionFactory;
import org.springframework.amqp.rabbit.listener.SimpleMessageListenerContainer;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MessageListenerConfig {
    @Autowired
    private CachingConnectionFactory connectionFactory;
    @Autowired
    private MyAckReceiver myAckReceiver;//消息接收处理类


    @Bean
    public SimpleMessageListenerContainer simpleMessageListenerContainer() {
        SimpleMessageListenerContainer container = new SimpleMessageListenerContainer(connectionFactory);
        container.setConcurrentConsumers(1);
        container.setMaxConcurrentConsumers(1);
        container.setAcknowledgeMode(AcknowledgeMode.MANUAL); // rabbitmq 默认为自动确认 这里更改为手动确认

        // 设置一个队列
//          container.setQueueNames("TestDirectQueue");
        //如果同时设置多个如下： 前提是队列都是必须已经创建存在的
          container.setQueueNames("TestDirectQueue","fanout.A");

        //另一种设置队列的方法,如果使用这种情况,那么要设置多个,就使用addQueues
        //container.setQueues(new Queue("TestDirectQueue",true));
        //container.addQueues(new Queue("TestDirectQueue2",true));
        //container.addQueues(new Queue("TestDirectQueue3",true));
        container.setMessageListener(myAckReceiver);
        return container;
    }

}
```



```java
package com.example.rabbit;


import com.rabbitmq.client.Channel;
import lombok.extern.slf4j.Slf4j;
import org.springframework.amqp.core.Message;
import org.springframework.amqp.rabbit.listener.api.ChannelAwareMessageListener;
import org.springframework.amqp.utils.SerializationUtils;
import org.springframework.stereotype.Component;

import java.io.ByteArrayInputStream;
import java.util.Arrays;
import java.util.HashMap;
import java.util.Map;

@Component
@Slf4j
public class MyAckReceiver implements ChannelAwareMessageListener {


    @Override
    public void onMessage(Message message, Channel channel) throws Exception {
        long deliveryTag = message.getMessageProperties().getDeliveryTag();
        try {
            Object content = SerializationUtils.deserialize(message.getBody());
            //因为传递消息的时候用的map传递,所以将Map从Message内取出需要做些处理
            // entryNums表示切割的次数
            Map<String, String> msgMap = StringtoMap(content.toString().trim(), 3);
            String messageId = msgMap.get("messageId");
            String messageData = msgMap.get("messageData");
            String createTime = msgMap.get("createTime");
            if ("TestDirectQueue".equals(message.getMessageProperties().getConsumerQueue())) {
                log.info("消费的消息来自的队列名为:{}",message.getMessageProperties().getConsumerQueue());
                log.info("消息成功消费到  messageId:{} , messageData:{} , createTime:{}",messageId,messageData,createTime);
                log.info("执行TestDirectQueue中的消息的业务处理流程......");

            }

            if ("fanout.A".equals(message.getMessageProperties().getConsumerQueue())) {
                log.info("消费的消息来自的队列名为:{}",message.getMessageProperties().getConsumerQueue());
                log.info("消息成功消费到  messageId:{} , messageData:{} , createTime:{}",messageId,messageData,createTime);
                log.info("执行fanout.A中的消息的业务处理流程......");
            }
            channel.basicAck(deliveryTag, true); //第二个参数，手动确认可以被批处理，当该参数为 true 时，则可以一次性确认 delivery_tag 小于等于传入值的所有消息
//			channel.basicReject(deliveryTag, true);//第二个参数，true会重新放回队列，所以需要自己根据业务逻辑判断什么时候使用拒绝

        } catch (Exception e) {
            channel.basicReject(deliveryTag, false);
            e.printStackTrace();
        }
    }

    public Map<String, String> StringtoMap(String str, int entryNums) {
        str = str.substring(1, str.length() - 1);
        String[] split = str.split(",", entryNums);
        Map<String, String> map = new HashMap<>();
        for (String s : split) {
            String key = s.split("=")[0].trim();
            String value = s.split("=")[1].trim();
            map.put(key, value);
        }
        return map;
    }

}

```

![image-20211208174413269](http://badwomen.asia/image-20211208174413269.png)

![image-20211208174440583](http://badwomen.asia/image-20211208174440583.png)

注意：

`channel.basicAck(deliveryTag, true);`

如果消费者端没有进行确认，那么消息就会被放在unacked中，每次重连之后还会读取这些数据直到你确认为止。



# 尾声

RabbitMq就算是完结了，rabbitMq对于我而言只是一个加分项，所以可能也不是复习的很细致。

下一个复习的应该是JVM或者JUC或者JAVA基础选一个把，其实三个算一类的，我可能会穿插的写。

Redis面试题还在写，面试题写完Redis也算完结了。接下来就是Mysql和ES的内容了。

# 参考文章

https://blog.csdn.net/qq_35387940/article/details/100514134

