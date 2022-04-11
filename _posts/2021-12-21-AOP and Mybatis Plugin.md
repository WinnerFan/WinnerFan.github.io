---
layout: post
title: AOP and Mybatis Plugin
tags: Java
---
## 背景
对mysql和redis监控，mysql有插件plguin，redis使用切面，切面可以扩大到任意方法

## mybatis plguin
### 简介
使用动态代理对方法拦截，拦截对象为拦截对象为org.apache.ibatis包下的接口类的方法，接口类为：
1. ParameterHandler：用来处理传入SQL的参数;
2. ResultSetHandler：用于处理结果集;
3. StatementHandler：用来处理SQL的执行过程，常用于重写SQL;
4. Executor：是SQL执行器，包含了组装参数，组装结果集到返回值以及执行SQL的过程。

### 实践

1. 引入依赖
```
<dependency>
    <groupId>org.mybatis</groupId>
    <artifactId>mybatis</artifactId>
    <version>${mybatis.version}</version>
</dependency>
```

2. 实现org.apache.ibatis.plugin.Interceptor接口
```java
@Intercepts({
        @Signature(
                type = Executor.class,
                method = "update",
                args = {MappedStatement.class, Object.class}),
        @Signature(
                type = Executor.class,
                method = "query",
                args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class}
                ),
        @Signature(
                type = Executor.class,
                method = "query",
                args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class, CacheKey.class, BoundSql.class}
        )
})
@Component
public class TestPlugin implements Interceptor {
    private Logger log = LoggerFactory.getLogger(TestPlugin.class);
    /**
     * 拦截方法
     * @param invocation
     * @return
     * @throws Throwable
     */
    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        MappedStatement mappedStatement = (MappedStatement) invocation.getArgs()[0];
        log.info("before"+mappedStatement.getId() + mappedStatement.getSqlCommandType())
        Object proceed = invocation.proceed();
        log.info("after")
        return proceed;
    }

    /**
     * 生成代理对象
     * @param target
     * @return
     */
    @Override
    public Object plugin(Object target) {
        return Plugin.wrap(target,this);
    }

    /**
     * 设置属性
     * @param properties
     */
    @Override
    public void setProperties(Properties properties) {}
}

```
3. 注入，类首字母小写即可
```xml
<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
    <property name="dataSource" ref="dataSource"/>
    <property name="configLocation" value="classpath:mybatis/mybatis-config.xml"/>
    <property name="mapperLocations" value="classpath:mybatis/sqlmap/*.xml"/>
    
    <property name="plugins" ref="testPlugin"/>
</bean>
```
### 顺序
一个语句一个plugin不会匹配两次， 每匹配一个plugin就会产生一个代理，靠后的plugin会代理前面的plugin，在外层，先执行。当Executor.query执行四个参数拦截后，显式调用六个参数的Executor.query，后面的拦截器需要拦截六个参数的Executor.query。


## AOP
```
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-aspects</artifactId>
    <version>${spring.version}</version>
</dependency>

```
1. @EnableAspectJAutoProxy：启动AOP，等同于`<aop:aspectj-autoproxy/>`
2. @Order(1)：修饰切面类，数值越小，切面越前
3. @Pointcut("execution(\* com.test..\*.\*(..))")：返回类型 包名（..当前包和子包）类名 方法名 方法参数
4. @Pointcut("@annotation(org.springframework.web.bind.annotation.PostMapping)")：所有@PostMapping注解方法做为切面
5. @Pointcut("@annotation(com.test.MyAnnotation)")：使用自定义@MyAnnotation注解方法做为切面
6. @Pointcut("@annotation(com.test.MyAnnotation) &&args(testArg)")：参数为Test（包含子类）才进入切面
7. @Pointcut("@annotation(com.test.MyAnnotation) &&@args(com.test.MyAnnotation1)")：参数的类被@MyAnnotation1（ElementType.TYPE）注释才进入切面，**直接注释参数无效，坑**。注解点（@MyAnnotation1）为父类，入参为子类，则无法匹配。

```
com.test

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface MyAnnotation{
    int test() default 10;
}

@Aspect
@Component
public class MyAdvice {
    @Pointcut("@annotation(com.test.MyAnnotation) && args(testArg)")
    public void myPointCut(Test testArg){}
    
    @Around("myPointCut(testArgForAround)") // Can be different from Pointcut
    public Object around(ProceedingJoinPoint joinPoint, Test testArgForAround) {
        MethodSignature methodSignature = (MethodSignature)joinPoint.getSignature()
        int test = methodSignature.getMethod().getAnnotation(MyAnnotation.class).test() // Get the MyAnnotation.test() value
        Object[] args = joinPoint.getArgs();
        Object ret = joinPoint.proceed(args);
        return ret;
    }
}

@RestController // @ResponseBody For change the http response code is necessary
@RequestMapping("/root")
public class MyController{
    @PostMapping("/test")
    @MyAnnotation(test=100)
    public MyResponse test(Test forTest, HttpServletResponse response){
        response.setStatus(500);
    }
}

```
