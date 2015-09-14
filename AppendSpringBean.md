## Notice: ##

You **MUST** configure guzz for spring in [TutorialConfig](TutorialConfig#Web_Application_with_Spring_IOC_support:.md)'s way.

## How to get the spring bean in guzz? ##

**By Injection：**

For almost all components you may need to write yourself, such as business interpreter, custom loader, shadow table view, custom table view, etc, you can declare the interface:

```
org.guzz.web.context.ExtendedBeanFactoryAware
```

to get injected on startup.

> The injected org.guzz.web.context.ExtendedBeanFactory has two methods:

> public Object getFactoryImpl() ; returns the outer container. For spring, it is org.springframework.context.ApplicationContext.

> public Object getBean(String beanName) ; returns the bean instance defined in spring.

**By GuzzContext：**

> If you are holding the reference of GuzzContext, you can call its two methods to fetch the outer bean.

> public ExtendedBeanFactory getExtendedBeanFactory() ; returns the ExtendedBeanFactory that is injected in the above section.

> public Object getExtendedBean(String beanName) ; returns the bean instance.

> Be careful, guzz is initialized as a bean in spring, so the ExtendedBeanFactory won't be ready until guzz has been fully initialized. These two methods can only be called in runtime, not startup stages, in guzz.

## Call Sequences ##

> The interface

```
org.guzz.web.context.ExtendedBeanFactoryAware
```

> is the last method that would be called by guzz. It is later even than the startup() methods in your user-defined components. Be very very careful!

## How to get guzz's service in spring? ##

> Sample:
```
<bean id="slowUpdateService" class="org.guzz.web.context.spring.GuzzServiceFactoryBean">
   	<property name="serviceName" value="guzzSlowUpdate" />
</bean>
```

> Define a spring bean for org.guzz.web.context.spring.GuzzServiceFactoryBean, and set the serviceName property to guzz's service name.

> One bean, one service.

## How to get guzz's DataSource in spring? ##

> Sample:
```
<bean id="dataSource" class="org.guzz.web.context.spring.GuzzDataSourceFactoryBean">
	<property name="masterDB" value="true" />
</bean>
```

> Define a spring bean for org.guzz.web.context.spring.GuzzDataSourceFactoryBean with the following 4 properties:

| **Property** | **Note** |
|:-------------|:---------|
| dbGroup      | dbgroup name in guzz. |
| tableCondition | The tableCondition for VirtualDB. |
| masterDB     | Select the master database's DataSource? true or false |
| slaveDB      | Select the slave database's DataSource? true or false. |



