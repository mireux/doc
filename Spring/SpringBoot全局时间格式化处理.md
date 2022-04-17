# SpringBoot全局时间格式化

## 初始化项目

pox.xml

~~~xml
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>
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
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>fastjson</artifactId>
            <version>1.2.76</version>
        </dependency>
    </dependencies>
~~~

创建一个测试实体类OrderInfo.java：

~~~java
@Data
@AllArgsConstructor
@NoArgsConstructor
public class OrderInfo {

    private LocalDateTime localDateTime;
    
    private Date updateTime;

}
~~~

创建接口测试类 TestController.java:

~~~java
@RestController
public class TestController {


    @GetMapping("/test/time")
    public OrderInfo testTimeFormat() {
        OrderInfo order = new OrderInfo();
        order.setLocalDateTime(LocalDateTime.now());
        order.setUpdateTime(new Date());
        return order;
    }


}
~~~

测试接口：

![image-20220103180242531](http://badwomen.asia/image-20220103180242531.png)

## 全局格式化

### 1. @JsonFormat注解

在OrderInfo.java的成员上加上注解：

~~~java
@Data
@AllArgsConstructor
@NoArgsConstructor
public class OrderInfo {
    @JsonFormat(locale = "zh",timezone = "GMT+8",pattern = "yyyy-MM-dd HH:mm:ss")
    private LocalDateTime localDateTime;
    @JsonFormat(locale = "zh",timezone = "GMT+8",pattern = "yyyy-MM-dd HH:mm:ss")
    private Date updateTime;

}
~~~

测试接口：

![image-20220103180843120](http://badwomen.asia/image-20220103180843120.png)

格式化成功！

### 2. **全局配置（1）**

虽然@JsonFormat足够用了，但是如果有很多个字段，每个都需要手动加上注解，工作量还是很大的。

`Springboot`已经为我们提供了日期格式化`${spring.jackson.date-format:yyyy-MM-dd HH:mm:ss}`，这里我们需要进行全局配置，配置比较简单，也无需在实体类属性上添加`@JsonFormat`注解。

**如果只需要格式化LocalDateTime：**

```java
@Configuration
public class LocalDateTimeSerializerConfig {

    @Value("${spring.jackson.date-format:yyyy-MM-dd HH:mm:ss}")
    private String pattern;

    @Bean
    public LocalDateTimeSerializer localDateTimeSerializer() {
        return new LocalDateTimeSerializer(DateTimeFormatter.ofPattern(pattern));
    }

    @Bean
    public Jackson2ObjectMapperBuilderCustomizer jackson2ObjectMapperBuilderCustomizer() {
        return jacksonObjectMapperBuilder -> jacksonObjectMapperBuilder.serializerByType(LocalDateTime.class,localDateTimeSerializer());
    }
    
}
```

![image-20220103181501597](http://badwomen.asia/image-20220103181501597.png)

格式化成功而且只有LocalDateTime被格式化。

**如果需要同时格式化Date和LocalDateTime：**

~~~java
@Configuration
public class LocalDateTimeSerializerConfig {

    @Value("${spring.jackson.date-format:yyyy-MM-dd HH:mm:ss}")
    private String pattern;

    @Bean
    public LocalDateTimeSerializer localDateTimeSerializer() {
        return new LocalDateTimeSerializer(DateTimeFormatter.ofPattern(pattern));
    }

    @Bean
    public DateSerializer dateSerializer() {
        return new DateSerializer(false,new SimpleDateFormat(pattern));
    }

    @Bean
    public Jackson2ObjectMapperBuilderCustomizer jackson2ObjectMapperBuilderCustomizer() {
        return jacksonObjectMapperBuilder -> jacksonObjectMapperBuilder.serializers(localDateTimeSerializer(),dateSerializer());
    }

//    @Bean
//    public Jackson2ObjectMapperBuilderCustomizer jackson2ObjectMapperBuilderCustomizer() {
//        return jacksonObjectMapperBuilder -> jacksonObjectMapperBuilder.serializerByType(LocalDateTime.class,localDateTimeSerializer());
//    }

}
~~~

![image-20220103181757318](http://badwomen.asia/image-20220103181757318.png)

这种方式可支持 `Date` 类型和 `LocalDateTime` 类型并存，那么有一个问题就是现在全局时间格式是`yyyy-MM-dd HH:mm:ss`，但有的字段却需要`yyyy-MM-dd`格式咋整？

那就需要配合`@JsonFormat`注解使用，在特定的字段属性添加`@JsonFormat`注解即可，因为`@JsonFormat`注解优先级比较高，会以`@JsonFormat`注解标注的时间格式为主。

~~~java
@Data
@AllArgsConstructor
@NoArgsConstructor
public class OrderInfo {
//    @JsonFormat(locale = "zh",timezone = "GMT+8",pattern = "yyyy-MM-dd HH:mm:ss")
    private LocalDateTime localDateTime;
    @JsonFormat(locale = "zh",timezone = "GMT+8",pattern = "yyyy-MM-dd")
    private Date updateTime;

}

~~~

![image-20220103182002370](http://badwomen.asia/image-20220103182002370.png)



### 3. 全局配置（2）

这种全局配置的实现方式与上边的效果是一样的，不过，要注意的是使用这种配置后，字段手动配置`@JsonFormat`注解将不再生效。

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
public class OrderInfo {
    @JsonFormat(locale = "zh",timezone = "GMT+8",pattern = "yyyy-MM-dd")
    private LocalDateTime localDateTime;
//    @JsonFormat(locale = "zh",timezone = "GMT+8",pattern = "yyyy-MM-dd")
    private Date updateTime;
}
```

```java
@Configuration
public class LocalDateTimeSerializerConfig {

    @Value("${spring.jackson.date-format:yyyy-MM-dd HH:mm:ss}")
    private String pattern;

    @Bean
    @Primary
    public ObjectMapper serializingObjectMapper() {
        ObjectMapper objectMapper = new ObjectMapper();
        JavaTimeModule javaTimeModule = new JavaTimeModule();
        javaTimeModule.addSerializer(LocalDateTime.class, new LocalDateTimeSerializer());
        javaTimeModule.addDeserializer(LocalDateTime.class, new LocalDateTimeDeserializer());
        objectMapper.registerModule(javaTimeModule);
        return objectMapper;
    }


    public class LocalDateTimeSerializer extends JsonSerializer<LocalDateTime> {
        @Override
        public void serialize(LocalDateTime value, JsonGenerator gen, SerializerProvider serializers) throws IOException {
            gen.writeString(value.format(ofPattern(pattern)));
        }
    }

    public class LocalDateTimeDeserializer extends JsonDeserializer<LocalDateTime> {
        @Override
        public LocalDateTime deserialize(JsonParser p, DeserializationContext deserializationContext) throws IOException {
            return LocalDateTime.parse(p.getValueAsString(), ofPattern(pattern));
        }
    }

}
```

![image-20220103183223162](http://badwomen.asia/image-20220103183223162.png)

