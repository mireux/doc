# SpringSecurity 认证

> 第一章是简单的应用，也差不多算是一个完整的登录接口的demo，真正的登录接口差不多都是这样的了。

## 0. 简介

​	**Spring Security** 是 Spring 家族中的一个安全管理框架。相比与另外一个安全框架**Shiro**，它提供了更丰富的功能，社区资源也比Shiro丰富。

​	一般来说中大型的项目都是使用**SpringSecurity** 来做安全框架。小项目有Shiro的比较多，因为相比与SpringSecurity，Shiro的上手更加的简单。

​	 一般Web应用的需要进行**认证**和**授权**。

​		**认证：验证当前访问系统的是不是本系统的用户，并且要确认具体是哪个用户**

​		**授权：经过认证后判断当前用户是否有权限进行某个操作**

​	而认证和授权也是SpringSecurity作为安全框架的核心功能。

## 1.快速入门

创建一个SpringBoot项目 这里就不赘述了

SpringSecurity我们只需要引入依赖即可实现入门案例。

~~~~xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>
~~~~

​	引入依赖后我们在尝试去访问之前的接口就会自动跳转到一个SpringSecurity的默认登陆页面

![image-20211221021058977](http://badwomen.asia/image-20211221021058977.png)

默认用户名是user,密码会输出在控制台。

​	![image-20211221021114905](http://badwomen.asia/image-20211221021114905.png)

必须登陆之后才能对接口进行访问。

![image-20211221021130629](http://badwomen.asia/image-20211221021130629.png)

## 2.认证

![image-20211215094003288](http://badwomen.asia/image-20211215094003288.png)

### 2.2 原理初探

​	想要知道如何实现自己的登陆流程就必须要先知道入门案例中SpringSecurity的流程。



#### 2.2.1 SpringSecurity完整流程

​	SpringSecurity的原理其实就是一个过滤器链，内部包含了提供各种功能的过滤器。这里我们可以看看入门案例中的过滤器。

其中最核心的三个过滤器：

**UsernamePasswordAuthenticationFilter**:负责处理我们在登陆页面填写了用户名密码后的登陆请求。入门案例的认证工作主要有它负责。

**ExceptionTranslationFilter：**处理过滤器链中抛出的任何AccessDeniedException和AuthenticationException 。

**FilterSecurityInterceptor：**负责权限校验的过滤器。



通过debug断点我们可以看到SpringSecurity中的所有过滤器

![image-20211221021426564](http://badwomen.asia/image-20211221021426564.png)

#### 2.2.2 认证流程详解

![image-20211214151515385](http://badwomen.asia/image-20211214151515385.png)

**概念速查**:

Authentication接口: 它的实现类，表示当前访问系统的用户，封装了用户相关信息。

AuthenticationManager接口：定义了认证Authentication的方法 

UserDetailsService接口：加载用户特定数据的核心接口。里面定义了一个根据用户名查询用户信息的方法。

UserDetails接口：提供核心用户信息。通过UserDetailsService根据用户名获取处理的用户信息要封装成UserDetails对象返回。然后将这些信息封装到Authentication对象中。

就是UsernamePasswordAuthenticationFilter 然后一层层调用到最后 -> InMemoryUserDetaillsManager 中的方法。但SpringSecurity原始是从内存中查找账号。所以我们要重写这个方法。如果查询成功则返回一个包装类UserDetails对象。然后层层返回到UsernamePasswordAuthenticationFilter。但为了最后返回需要Token。所以我们需要重写这个接口类。



### 2.3 解决问题

#### 2.3.1 思路分析

登录

​	①自定义登录接口  

​				调用ProviderManager的方法进行认证 如果认证通过生成jwt

​				把用户信息存入redis中

​	②自定义UserDetailsService 

​				在这个实现类中去查询数据库

校验：

​	①定义Jwt认证过滤器

​				获取token

​				解析token获取其中的userid

​				从redis中获取用户信息

​				存入SecurityContextHolder

> 简单思路 看代码注释

#### 2.3.2 准备工作

添加依赖：

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
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>fastjson</artifactId>
            <version>1.2.76</version>
        </dependency>
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt</artifactId>
            <version>0.9.1</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
        <dependency>
            <groupId>com.baomidou</groupId>
            <artifactId>mybatis-plus-boot-starter</artifactId>
            <version>3.4.3</version>
        </dependency>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
        </dependency>
        <dependency>
            <groupId>javax.xml.bind</groupId>
            <artifactId>jaxb-api</artifactId>
            <version>2.3.0</version>
        </dependency>
    </dependencies>
~~~



添加响应类： 

~~~java
package com.example.springbootandsecuritytest.lang;

import com.fasterxml.jackson.annotation.JsonInclude;

@JsonInclude(JsonInclude.Include.NON_NULL)
public class Result<T> {
    /**
     * 状态码
     */
    private Integer code;
    /**
     * 提示信息，如果有错误时，前端可以获取该字段进行提示
     */
    private String msg;
    /**
     * 查询到的结果数据，
     */
    private T data;

    public Result(Integer code, String msg) {
        this.code = code;
        this.msg = msg;
    }

    public Result(Integer code, T data) {
        this.code = code;
        this.data = data;
    }

    public Integer getCode() {
        return code;
    }

    public void setCode(Integer code) {
        this.code = code;
    }

    public String getMsg() {
        return msg;
    }

    public void setMsg(String msg) {
        this.msg = msg;
    }

    public T getData() {
        return data;
    }

    public void setData(T data) {
        this.data = data;
    }

    public Result(Integer code, String msg, T data) {
        this.code = code;
        this.msg = msg;
        this.data = data;
    }
}
~~~



工具类

~~~java
package com.example.springsecuritytest.utils;

import io.jsonwebtoken.Claims;
import io.jsonwebtoken.JwtBuilder;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.SignatureAlgorithm;

import javax.crypto.SecretKey;
import javax.crypto.spec.SecretKeySpec;
import java.util.Base64;
import java.util.Date;
import java.util.UUID;

/**
 * JWT工具类
 */
public class JwtUtil {

    //有效期为
    public static final Long JWT_TTL = 60 * 60 *1000L;// 60 * 60 *1000  一个小时
    //设置秘钥明文
    public static final String JWT_KEY = "lhjlhj";

    public static String getUUID(){
        String token = UUID.randomUUID().toString().replaceAll("-", "");
        return token;
    }
    
    /**
     * 生成jwt
     * @param subject token中要存放的数据（json格式）
     * @return
     */
    public static String createJWT(String subject) {
        JwtBuilder builder = getJwtBuilder(subject, null, getUUID());// 设置过期时间
        return builder.compact();
    }

    /**
     * 生成jwt
     * @param subject token中要存放的数据（json格式）
     * @param ttlMillis token超时时间
     * @return
     */
    public static String createJWT(String subject, Long ttlMillis) {
        JwtBuilder builder = getJwtBuilder(subject, ttlMillis, getUUID());// 设置过期时间
        return builder.compact();
    }

    private static JwtBuilder getJwtBuilder(String subject, Long ttlMillis, String uuid) {
        SignatureAlgorithm signatureAlgorithm = SignatureAlgorithm.HS256;
        SecretKey secretKey = generalKey();
        long nowMillis = System.currentTimeMillis();
        Date now = new Date(nowMillis);
        if(ttlMillis==null){
            ttlMillis=JwtUtil.JWT_TTL;
        }
        long expMillis = nowMillis + ttlMillis;
        Date expDate = new Date(expMillis);
        return Jwts.builder()
                .setId(uuid)              //唯一的ID
                .setSubject(subject)   // 主题  可以是JSON数据
                .setIssuer("lhj")     // 签发者
                .setIssuedAt(now)      // 签发时间
                .signWith(signatureAlgorithm, secretKey) //使用HS256对称加密算法签名, 第二个参数为秘钥
                .setExpiration(expDate);
    }

    /**
     * 创建token
     * @param id
     * @param subject
     * @param ttlMillis
     * @return
     */
    public static String createJWT(String id, String subject, Long ttlMillis) {
        JwtBuilder builder = getJwtBuilder(subject, ttlMillis, id);// 设置过期时间
        return builder.compact();
    }

    /**
     * 生成加密后的秘钥 secretKey
     * @return
     */
    public static SecretKey generalKey() {
        byte[] encodedKey = Base64.getDecoder().decode(JwtUtil.JWT_KEY);
        SecretKey key = new SecretKeySpec(encodedKey, 0, encodedKey.length, "AES");
        return key;
    }
    
    /**
     * 解析
     *
     * @param jwt
     * @return
     * @throws Exception
     */
    public static Claims parseJWT(String jwt) throws Exception {
        SecretKey secretKey = generalKey();
        return Jwts.parser()
                .setSigningKey(secretKey)
                .parseClaimsJws(jwt)
                .getBody();
    }


}
~~~

建表：

~~~sql
CREATE TABLE `sys_user` (
  `id` BIGINT(20) NOT NULL AUTO_INCREMENT COMMENT '主键',
  `user_name` VARCHAR(64) NOT NULL DEFAULT 'NULL' COMMENT '用户名',
  `nick_name` VARCHAR(64) NOT NULL DEFAULT 'NULL' COMMENT '昵称',
  `password` VARCHAR(64) NOT NULL DEFAULT 'NULL' COMMENT '密码',
  `status` CHAR(1) DEFAULT '0' COMMENT '账号状态（0正常 1停用）',
  `email` VARCHAR(64) DEFAULT NULL COMMENT '邮箱',
  `phonenumber` VARCHAR(32) DEFAULT NULL COMMENT '手机号',
  `sex` CHAR(1) DEFAULT NULL COMMENT '用户性别（0男，1女，2未知）',
  `avatar` VARCHAR(128) DEFAULT NULL COMMENT '头像',
  `user_type` CHAR(1) NOT NULL DEFAULT '1' COMMENT '用户类型（0管理员，1普通用户）',
  `create_by` BIGINT(20) DEFAULT NULL COMMENT '创建人的用户id',
  `create_time` DATETIME DEFAULT NULL COMMENT '创建时间',
  `update_by` BIGINT(20) DEFAULT NULL COMMENT '更新人',
  `update_time` DATETIME DEFAULT NULL COMMENT '更新时间',
  `del_flag` INT(11) DEFAULT '0' COMMENT '删除标志（0代表未删除，1代表已删除）',
  PRIMARY KEY (`id`)
) ENGINE=INNODB AUTO_INCREMENT=2 DEFAULT CHARSET=utf8mb4 COMMENT='用户表'
~~~



- 重写 USerDetailService

~~~java
@Service
public class UserDetailsServiceImpl implements UserDetailsService {

    @Autowired
    private UserMapper userMapper;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        // 根据用户名查找用户
        LambdaQueryWrapper<User> lambdaQueryWrapper = new LambdaQueryWrapper<>();
        lambdaQueryWrapper.eq(User::getUserName,username);
        User user = userMapper.selectOne(lambdaQueryWrapper);
        // 如果查找不到抛出异常来给出提示
        if(Objects.isNull(user)) {
            throw new RuntimeException("登录失败");
        }
        // TODO 根据用户查询权限信息 添加到LoginUser中

        // 封装成UserDetails返回
        return new LoginUser(user);
    }
}



~~~

- LoginUser类 实现UserDetails 接口

 ~~~java
 package com.example.springsecuritytest.param;
 
 import com.example.springsecuritytest.Entity.User;
 import lombok.AllArgsConstructor;
 import lombok.Data;
 import lombok.NoArgsConstructor;
 import org.springframework.beans.factory.annotation.Autowired;
 import org.springframework.security.core.GrantedAuthority;
 import org.springframework.security.core.userdetails.UserDetails;
 
 import java.util.Collection;
 
 
 @Data
 @NoArgsConstructor
 @AllArgsConstructor
 public class LoginUser implements UserDetails {
 
 
     @Autowired
     private User user;
 
     /**
      * 获取权限
      * @return
      */
     @Override
     public Collection<? extends GrantedAuthority> getAuthorities() {
         return null;
     }
 
     /**
      * 获取用户密码用于回调
      * @return
      */
     @Override
     public String getPassword() {
         return user.getPassword();
     }
 
     /**
      * 获取用户名
      * @return
      */
     @Override
     public String getUsername() {
         return user.getUserName();
     }
 
     /**
      * 账号是否过期
      * @return
      */
     @Override
     public boolean isAccountNonExpired() {
         return true;
     }
 
     /**
      * 账号是否被锁定
      * @return
      */
     @Override
     public boolean isAccountNonLocked() {
         return true;
     }
 
 
     /**
      * 凭证是否过期
      * @return
      */
     @Override
     public boolean isCredentialsNonExpired() {
         return true;
     }
 
     /**
      * 是否禁用
      * @return
      */
     @Override
     public boolean isEnabled() {
         return true;
     }
 }
 
 ~~~

注意：如果要测试，需要往用户表中写入用户数据，并且如果你想让用户的密码是明文存储，需要在密码前加{noop}。例如

![image-20211221023333600](http://badwomen.asia/image-20211221023333600.png)

这样登陆的时候就可以用lhj作为用户名，1234作为密码来登陆了。

##### 2.3.3.2 密码加密存储

​	实际项目中我们不会把密码明文存储在数据库中。

​	默认使用的PasswordEncoder要求数据库中的密码格式为：{id}password 。它会根据id去判断密码的加密方式。但是我们一般不会采用这种方式。所以就需要替换PasswordEncoder。

​	我们一般使用SpringSecurity为我们提供的BCryptPasswordEncoder。

![image-20211221023527383](http://badwomen.asia/image-20211221023527383.png)

​	我们只需要使用把BCryptPasswordEncoder对象注入Spring容器中，SpringSecurity就会使用该PasswordEncoder来进行密码校验。

​	我们可以定义一个SpringSecurity的配置类，SpringSecurity要求这个配置类要继承WebSecurityConfigurerAdapter。

~~~java
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {


    /**
     * 默认使用的PasswordEncoder要求数据库中的密码格式为：{id}password 。它会根据id去判断密码的加密方式。但是我们一般不会采用这种方式。所以就需要替换PasswordEncoder。
     * 我们一般使用SpringSecurity为我们提供的BCryptPasswordEncoder。
     * @return
     */

    @Bean
    public PasswordEncoder passwordEncoder(){
        return new BCryptPasswordEncoder();
    }
}
~~~

##### 2.3.3.3 登陆接口

​	接下我们需要自定义登陆接口，然后让SpringSecurity对这个接口放行。

​	在接口中我们通过AuthenticationManager的authenticate方法来进行用户认证,所以需要在SecurityConfig中配置把AuthenticationManager注入容器。

​	认证成功的话要生成一个jwt，放入响应中返回。并且为了让用户下回请求时能通过jwt识别出具体的是哪个用户，我们需要把用户信息存入redis，可以把用户id作为key。

​	//把authenticaton存入ContextHolder 待定 等校验写完再验证

```java
@RestController
public class LoginController {


    @Autowired
    private LoginService loginService;

    @PostMapping("/user/login")
    public Result login(@RequestBody LoginParam loginParam) {
        return loginService.login(loginParam);
    }


}
```

```java
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {


    /**
     * 默认使用的PasswordEncoder要求数据库中的密码格式为：{id}password 。它会根据id去判断密码的加密方式。但是我们一般不会采用这种方式。所以就需要替换PasswordEncoder。
     * 我们一般使用SpringSecurity为我们提供的BCryptPasswordEncoder。
     * @return
     */

    @Bean
    public PasswordEncoder passwordEncoder(){
        return new BCryptPasswordEncoder();
    }


    /**
     * 放行Login接口
     * @param http
     * @throws Exception
     */
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
                //关闭csrf
                .csrf().disable()
                //不通过Session获取SecurityContext
                .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS)
                .and()
                .authorizeRequests()
                // 对于登录接口 允许匿名访问
                .antMatchers("/user/login").anonymous()
                // 除上面外的所有请求全部需要鉴权认证
                .anyRequest().authenticated();
    }

    @Bean
    @Override
    public AuthenticationManager authenticationManagerBean() throws Exception {
        return super.authenticationManagerBean();
    }
}
```

```java
package com.example.springsecuritytest.service.Impl;

import com.example.springsecuritytest.Entity.User;
import com.example.springsecuritytest.lang.Result;
import com.example.springsecuritytest.param.LoginParam;
import com.example.springsecuritytest.param.LoginUser;
import com.example.springsecuritytest.service.LoginService;
import com.example.springsecuritytest.utils.JwtUtil;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.Authentication;
import org.springframework.stereotype.Service;

import java.util.HashMap;
import java.util.Map;
import java.util.Objects;

@Service
public class LoginServiceImpl implements LoginService {


    private AuthenticationManager authenticationManager;

    private RedisTemplate redisTemplate;

    @Autowired
    public LoginServiceImpl(AuthenticationManager authenticationManager,RedisTemplate redisTemplate) {
        this.authenticationManager = authenticationManager;
        this.redisTemplate = redisTemplate;
    }

    @Override
    public Result login(LoginParam loginParam) {
        // 调用自定义的登录接口 调用ProviderManager的方法进行认证 如果认证通过生成jwt
        UsernamePasswordAuthenticationToken authenticationToken = new UsernamePasswordAuthenticationToken(loginParam.getUsername(),loginParam.getPassword());
        Authentication authenticate = authenticationManager.authenticate(authenticationToken);
        // 如果通过 则authenticate不为空
        if(Objects.isNull(authenticate)) {
            throw new RuntimeException("用户名或密码错误");
        }
        //使用userid生成token
        LoginUser loginUser = (LoginUser) authenticate.getPrincipal();
        String userid = loginUser.getUser().getId().toString();
        System.out.println(userid);
        String jwt = JwtUtil.createJWT(userid);
        // 放入redis缓存
        redisTemplate.opsForValue().set("login:" + userid,loginUser);
        Map<String,Object> map = new HashMap<>();
        map.put("token",jwt);
        return new Result(200,"登录成功",map);
    }
}
```



认证功能到这就差不多成功了 Postman 调试也没问题![image-20211221023847947](http://badwomen.asia/image-20211221023847947.png)



##### 2.3.3.4 认证过滤器

JwtAuthenticationTokenFilter类：

```java
package com.example.springsecuritytest.filter;

import com.example.springsecuritytest.param.LoginUser;
import com.example.springsecuritytest.utils.JwtUtil;
import io.jsonwebtoken.Claims;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.stereotype.Component;
import org.springframework.util.StringUtils;
import org.springframework.web.filter.OncePerRequestFilter;

import javax.servlet.FilterChain;
import javax.servlet.ServletException;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.util.Objects;

@Component
public class JwtAuthenticationTokenFilter extends OncePerRequestFilter {

    @Autowired
    private RedisTemplate redisTemplate;

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
        // 获取token
        String token = request.getHeader("token");
        if (!StringUtils.hasText(token)) {
            // 放行
            filterChain.doFilter(request, response);
            return;
        }
        // 解析token
        String userId;
        try {
            Claims claims = JwtUtil.parseJWT(token);
            userId = claims.getSubject();
        } catch (Exception e) {
            e.printStackTrace();
            throw new RuntimeException("Token非法");
        }
        // 如果token解析成功 从redis中获取用户信息
        String RedisKey = "login:" + userId;
        LoginUser loginUser = (LoginUser) redisTemplate.opsForValue().get(RedisKey);
        if(Objects.isNull(loginUser)) {
            throw new RuntimeException("用户信息不存在");
        }
        // 存放用户信息到SecurityContextHolder 因为setAuthentication需要一个Authentication对象 所以我们需要进行封装
        /**
         * 这里需要使用三个函数的方法 方法内有super.setAuthenticated(true); 表示已经认证
         * 可以跳过后面的一些认证
         * TODO 获取权限信息封装到Authentication
         */
        UsernamePasswordAuthenticationToken authenticationToken = new UsernamePasswordAuthenticationToken(loginUser,null,null);
        SecurityContextHolder.getContext().setAuthentication(authenticationToken);
        // 放行
        filterChain.doFilter(request,response);
    }
}
```

到这里我们只是配置了一个JwtAuthenticationTokenFilter过滤器，但并没有放入SpringSecurity的过滤器链中，所以我们还需要再SecurityConfig中配置。

```java
/**
 * 放行Login接口
 * @param http
 * @throws Exception
 */
@Override
protected void configure(HttpSecurity http) throws Exception {
    http
            //关闭csrf
            .csrf().disable()
            //不通过Session获取SecurityContext
            .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            .and()
            .authorizeRequests()
            // 对于登录接口 允许匿名访问
            .antMatchers("/user/login").anonymous()
            // 除上面外的所有请求全部需要鉴权认证
            .anyRequest().authenticated();

    // 表示希望把自定义的过滤器添加到SpringSecurity过滤器链中
    http.addFilterBefore(jwtAuthenticationTokenFilter, UsernamePasswordAuthenticationFilter.class);
}
```

##### 2.3.3.5 退出登陆

```java
@Override
public Result logout() {
    // 从SecurityContextHolder中获取用户信息 然后删除 Redis中存放的用户的值
    UsernamePasswordAuthenticationToken authentication = (UsernamePasswordAuthenticationToken) SecurityContextHolder.getContext().getAuthentication();
    LoginUser loginUser = (LoginUser) authentication.getPrincipal();
    String userId = loginUser.getUser().getId().toString();
    // 删除redis中的用户信息
    redisTemplate.delete("login:" + userId);
    return new Result(200,"退出登录成功");
}
```





## 3. 授权

### 3.1 授权基本流程

​	在SpringSecurity中，会使用默认的FilterSecurityInterceptor来进行权限校验。在FilterSecurityInterceptor中会从SecurityContextHolder获取其中的Authentication，然后获取其中的权限信息。当前其中是否拥有访问当前资源所需的权限。

​	所以我们在项目中只需要把当前登录用户的权限信息也存入Authentication。

​	然后设置我们的资源所需要的权限即可。

### 3.2 基于注解的权限控制

​	SpringSecurity为我们提供了基于注解的权限控制方案，这也是我们项目中主要采用的方式。我们可以使用注解去指定访问对应的资源所需的权限。

​	但是要使用它我们需要先开启相关配置。

~~~~java
@EnableGlobalMethodSecurity(prePostEnabled = true)
~~~~

​	然后就可以使用对应的注解。@PreAuthorize



```java
package com.example.springsecuritytest.service.Impl;

import com.baomidou.mybatisplus.core.conditions.query.LambdaQueryWrapper;
import com.example.springsecuritytest.Entity.User;
import com.example.springsecuritytest.mapper.UserMapper;
import com.example.springsecuritytest.param.LoginUser;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.core.userdetails.UsernameNotFoundException;
import org.springframework.stereotype.Service;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;
import java.util.Objects;

@Service
public class UserDetailsServiceImpl implements UserDetailsService {

    @Autowired
    private UserMapper userMapper;


    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        // 根据用户名查找用户
        LambdaQueryWrapper<User> lambdaQueryWrapper = new LambdaQueryWrapper<>();
        lambdaQueryWrapper.eq(User::getUserName,username);
        User user = userMapper.selectOne(lambdaQueryWrapper);
        // 如果查找不到抛出异常来给出提示
        if(Objects.isNull(user)) {
            throw new RuntimeException("登录失败");
        }
        // 根据用户查询权限信息 添加到LoginUser中
        List<String> permissions = new ArrayList<>(Arrays.asList("admin","test")); // 实际从数据库中获取对应的权限信息
        // 封装成UserDetails返回

        return new LoginUser(user,permissions);
    }
}
```





```java
package com.example.springsecuritytest.filter;

import com.example.springsecuritytest.param.LoginUser;
import com.example.springsecuritytest.utils.JwtUtil;
import io.jsonwebtoken.Claims;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.stereotype.Component;
import org.springframework.util.StringUtils;
import org.springframework.web.filter.OncePerRequestFilter;

import javax.servlet.FilterChain;
import javax.servlet.ServletException;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.util.Objects;

@Component
public class JwtAuthenticationTokenFilter extends OncePerRequestFilter {

    @Autowired
    private RedisTemplate redisTemplate;

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
        // 获取token
        String token = request.getHeader("token");
        if (!StringUtils.hasText(token)) {
            // 放行
            filterChain.doFilter(request, response);
            return;
        }
        // 解析token
        String userId;
        try {
            Claims claims = JwtUtil.parseJWT(token);
            userId = claims.getSubject();
        } catch (Exception e) {
            e.printStackTrace();
            throw new RuntimeException("Token非法");
        }
        // 如果token解析成功 从redis中获取用户信息
        String RedisKey = "login:" + userId;
        LoginUser loginUser = (LoginUser) redisTemplate.opsForValue().get(RedisKey);
        if(Objects.isNull(loginUser)) {
            throw new RuntimeException("用户信息不存在");
        }
        // 存放用户信息到SecurityContextHolder 因为setAuthentication需要一个Authentication对象 所以我们需要进行封装
        /**
         * 这里需要使用三个函数的方法 方法内有super.setAuthenticated(true); 表示已经认证
         * 可以跳过后面的一些认证
         * 获取权限信息封装到Authentication
         */
        UsernamePasswordAuthenticationToken authenticationToken = new UsernamePasswordAuthenticationToken(loginUser,null,loginUser.getAuthorities());
        SecurityContextHolder.getContext().setAuthentication(authenticationToken);
        // 放行
        filterChain.doFilter(request,response);
    }
}
```

