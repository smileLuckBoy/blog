---
title: Spring的注解@Qualifier小结
date: 2016-08-29 22:34:53
tags: spring
categories: JAVA
toc: true
---

近期在捯饬spring的注解，现将遇到的问题记录下来，以供遇到同样问题的童鞋解决~

<!-- more -->
### 场景
 
有如下接口：

```java
public interface EmployeeService {
    public EmployeeDto getEmployeeById(Long id); 
}
```

同时有下述两个实现类 EmployeeServiceImpl和EmployeeServiceImpl1：

```java
@Service("service")
public class EmployeeServiceImpl implements EmployeeService {
    public EmployeeDto getEmployeeById(Long id) {
        return new EmployeeDto();
    }
}

@Service("service1")
public class EmployeeServiceImpl1 implements EmployeeService {
    public EmployeeDto getEmployeeById(Long id) {
        return new EmployeeDto();
    }
}
```

调用代码如下：
```java
@Controller
@RequestMapping("/emplayee.do")
public class EmployeeInfoControl {
    @Autowired
    EmployeeService employeeService;
    
}
```

### 结果
在启动tomcat时报如下错误：



```exception
org.springframework.beans.factory.BeanCreationException: 
nested exception is org.springframework.beans.factory.NoSuchBeanDefinitionException: 
No unique bean of type [com.test.service.EmployeeService] is defined: 
expected single matching bean but found 2: [service1, service2]
```


### 分析
其实报错信息已经说得很明确了,在autoware时，由于有两个类实现了EmployeeService接口，所以Spring不知道应该绑定哪个实现类，所以抛出了如上错误。

这个时候就要用到**<font color="red">@Qualifier</font>**注解了，Qualifier的意思是合格者，通过这个标示，表明了哪个实现类才是我们所需要的，我们修改调用代码，添加@Qualifier注解，需要注意的是@Qualifier的参数名称必须为我们之前定义@Service注解的名称之一！


### 结论
当出现上述情况时，只需增加@Qualifier注解即可，代码如下(表示实例化EmployeeServiceImpl类)：
```java
@Controller
@RequestMapping("/emplayee.do")
public class EmployeeInfoControl {
    @Autowired
    @Qualifier("service")
    EmployeeService employeeService;
    
}

```

问题解决！


<br />
