---
layout: post
title: "Spring Expression Language (SpEL)在多线程环境下的一个问题(Spring 3.0)"
description: "Spring issue with multiple threads"
category: Spring
tags: [Spring]
---
{% include JB/setup %}

##问题描述
项目基于Spring 3.0， 在Bean文件的定义中使用了SpEL,系统在运行初始化的时候会首先载入一些Bean。在新定义了一个Bean后偶尔会因为如下异常导致初始化失败。

	org.springframework.beans.factory.BeanExpressionException: Expression parsing failed; nested exception is org.springframework.expression.spel.SpelEvaluationException: EL1004E:(pos 7): Method call: Method getP(java.lang.String) cannot be found on com.ecy.Entity type


思考点：

* 多线程环境
* 新定义的Bean和一个已经存在的Bean有相同的SpEL定义 

##背景介绍
SpEL是从Spring 3.0开始引入的一个功能，用于支持运行时查询和操作对象图的表达式语言。Spring Docs描述如下：

> The Spring Expression Language (SpEL for short) is a powerful expression language that supports querying and manipulating an object graph at runtime. The language syntax is similar to Unified EL but offers additional features, most notably method invocation and basic string templating functionality.

通常在定义Bean的xml中用‘#{}’包含一段SpEL， 如：

	<bean id="BeanA" class="com.ecy.CA" scope="prototype">
		<property name="name" value="#{Entity.getP(T(com.ecy.Entity).PA)}"/>
	</bean>

可以通过如下方式来得到一个Bean的实例

	ApplicationContext context = new ClassPathXmlApplicationContext("spring-beans.xml");
	context.getBean(String beanName);

SpEL主要是要在getBean的过程中得到值，如上文BeanA的例子中，Entity的定义如下：

    public class Entity {
		public static final String PA = "PA";	
		public Map<String, String> configs = new ConcurrentHashMap<String, String>();
	
		public String getP(String key) {
			return configs.get(key);
		}
	}

由上可以看出BeanA的这个SpEL实际上是要在创建的过程中拿到Entity对象中configs中key为"PA"的值来做为自己的name.


##代码分析
这个过程可以简化为以下4步：

1. >
	ExpressionParser parser = new SpelExpressionParser(); //get expression parser

   >	Entity entity = new Entity();	
   
2. >
	EvaluationContext ec = new StandardEvaluationContext(entity); //get evaluation context 
3. >
	Expression expression = parser.parseExpression("getP(T(com.ecy.Entity).PA)"); //根据SpEL解析出表达式 
4. >
	value = expression.getValue(ec); //根据表达式取出值

看看第二步的详细代码：

org.springframework.context.expression.StandardBeanExpressionResolver.java

    private final Map<BeanExpressionContext, StandardEvaluationContext> evaluationCache = new ConcurrentHashMap<BeanExpressionContext, StandardEvaluationContext>();
    Expression expr = this.expressionCache.get(value);

    if (expr == null) {
        expr = this.expressionParser.parseExpression(value, this.beanExpressionParserContext);
        this.expressionCache.put(value, expr);
    }
    
    StandardEvaluationContext sec = this.evaluationCache.get(evalContext); 
    if (sec == null) { 
        sec = new StandardEvaluationContext(); 
        sec.setRootObject(evalContext);
        ...
        this.evaluationCache.put(evalContext, sec); 
    }
    return expr.getValue(sec);

可以看出一样的SpEL表达式总是得到同一个的Expression对象，并且这个类StandardBeanExpressionResolver是非线程安全的，多线程环境下，同样的evalContext对象可能会得到不同的StandardEvaluationContext(sec)对象。第四步正是Expression对象通过sec得到表达式值的。而StandardEvaluationContext
也是一个非线程安全的类，代码如下：

    public List<MethodResolver> getMethodResolvers() {
        ensureMethodResolversInitialized();
        return this.methodResolvers;
    }

    private void ensureMethodResolversInitialized() {
        if (this.methodResolvers == null) {
            this.methodResolvers = new ArrayList<MethodResolver>();
            this.methodResolvers.add(new ReflectiveMethodResolver());
        }
    }

可以看出，多线程环境下，methodResolvers可能返回为一个空的ArrayList.

这样就导致了如下的异常：

org.springframework.beans.factory.BeanExpressionException: Expression parsing failed; nested exception is org.springframework.expression.spel.SpelEvaluationException: EL1004E:(pos 7): Method call: Method getP(java.lang.String) cannot be found on com.ecy.Entity type


同样的道理，在这个类中还有同样的非线程安全的代码同样会导致相关的异常：

    public List<PropertyAccessor> getPropertyAccessors() {
        ensurePropertyAccessorsInitialized();
        return this.propertyAccessors;
    }

    private void ensurePropertyAccessorsInitialized() {
        if (this.propertyAccessors == null) {
            this.propertyAccessors = new ArrayList<PropertyAccessor>();
            this.propertyAccessors.add(new ReflectivePropertyAccessor());
        }
    }


##问题总结
在多线程环境下使用Spring 3.0中的SpEL时要注意避免：

* 多线程通过getBean去取得同一个Bean对象，特别是当这个Bean的scope是prototype时。

另外，也要避免多线程创建有相同SpEL表达式的Bean。


##解决方式
* 在Spring 3.0中， 可以封装getBean(String name), 将方法同步化，避免多线程同时创建：

    public class BeanUtil {
        private static final ApplicationContext FACTORY = new ClassPathXmlApplicationContext("spring-beans.xml");
  
        public synchronized static Object getBean(String name){
            return FACTORY.getBean(name); 
        }
    }

* Spring 3.1通过修改了org.springframework.expression.spel.support.StandardEvaluationContext也到达了线程安全的目的：

>

    private void ensureMethodResolversInitialized() {
        if (this.methodResolvers == null) {
            initializeMethodResolvers();
        }
    }
    
    private synchronized void initializeMethodResolvers() {
        if (this.methodResolvers == null) {
            List<MethodResolver> defaultResolvers = new ArrayList<MethodResolver>();
            defaultResolvers.add(reflectiveMethodResolver = new ReflectiveMethodResolver());
            this.methodResolvers = defaultResolvers;
        }
    }
    

