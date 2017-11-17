---
title: Traceroute
date: 2017-11-14 19:01:16
categories:
- technique
tags:
- Spring
- Java
---

## 前言
> 前面学习了简单的`Spring Web`知识，接着学习更高阶的`Web`技术。

## 高级技术

### Spring MVC配置的替换方案

#### 自定义DispatcherServlet配置

在第五章我们曾编写过如下代码。

```java

public class SpitterWebInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {

    @Override
    protected Class<?>[] getRootConfigClasses() {
        return new Class<?>[] { RootConfig.class };
    }

    @Override
    protected Class<?>[] getServletConfigClasses() {
        return new Class<?>[] { WebConfig.class };
    }

    @Override
    protected String[] getServletMappings() {
        return new String[] { "/" };
    }

}

```

可以看到`SpitterWebinitializer`实现了`AbstractAnnotationConfigDispatcherServletInitializer`抽象类，并重写了三个必须的方法，实际上还可对更多方法进行重写，以便实现额外的配置，如对`customizeRegistration`方法进行重写，该方法是`AbstractDispatcherServletInitializer`的方法，无实际的方法体。当`AbstractAnnotationConfigDispatcherServletInitializer`将`DispatcherServlet`注册到`Servlet`容器中后，就会调用`customizeRegistration`方法，并将`Servlet`注册后得到的`Registration.Dynamic`传入。可通过重写`customizeRegistration`方法设置`MultipartConfigElement`，如下所示。

```java

    @Override
    protected void customizeRegistration(Dynamic registration) {
        registration.setMultipartConfig(
                new MultipartConfigElement("/tmp/spittr/uploads"));
    }

```

#### 添加其他Servlet和Filter

`AbstractAnnotationConfigDispatcherServletInitializer`会创建`DispatcherServlet`和`ContextLoaderListener`，当需要添加其他`Servlet`和`Filter`时，只需要创建一个新的初始化器即可，最简单的方式是实现`WebApplicationInitializer`接口。

```java

import org.springframework.web.WebApplicationInitializer;

import javax.servlet.FilterRegistration;
import javax.servlet.ServletContext;
import javax.servlet.ServletException;
import javax.servlet.ServletRegistration.Dynamic;

public class MyServletInitializer implements WebApplicationInitializer {

    public void onStartup(ServletContext servletContext) throws ServletException {
        Dynamic servlet = servletContext.addServlet("myServlet", MyServlet.class);
        servlet.addMapping("/custom/**");

        FilterRegistration.Dynamic filter = servletContext.addFilter("myFilter", MyFilter.class);
        filter.addMappingForUrlPatterns(null, false, "/custom/*");

    }
}

```

#### 在xml文件中声明DispatcherServlet

对基本的`Spring MVC`应用而言，需要配置`DispatcherServlet`和`ContextLoaderListener`，`web.xml`配置如下。

```xml

<web-app>
  <display-name>Archetype Created Web Application</display-name>
  <context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>/WEB-INF/spring/root-context.xml</param-value>
  </context-param>
  
  <listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
  </listener>
  
  <servlet>
    <servlet-name>appServlet</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <load-on-startup>1</load-on-startup>
  </servlet>
  
  <servlet-mapping>
    <servlet-name>appServlet</servlet-name>
    <url-pattern>/</url-pattern>
  </servlet-mapping>
  
</web-app>

```

可以看到在`web.xml`中配置了`DispatcherServlet`和`ContextLoaderListener`，并且定义了上下文，该上下文会被`ContextLoaderListener`加载，从中读取`bean`。也可指定`DispatcherServlet`的应用上下文并完成加载，配置`web.xml`如下。

```xml

  <servlet>
    <servlet-name>appServlet</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <init-param>
      <param-name>contextConfigLocation</param-name>
      <param-value>WEB-INF/spring/appServlet/servlet-context.xml</param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
  </servlet>

```

上面使用`DispatcherServlet`和`ContextLoaderListener`加载各自的上下文，但实际情况中，基于`Java`的配置更为通用，此时只需要配置`DispatcherServlet`和`ContextLoaderListener`使用`AnnotationConfigWebApplicationContext`，这样它便可加载`Java`配置类，而非使用`xml`，可设置`contextClass`和`DispathcerServlet`的初始化参数，如下所示。

```xml

<web-app>
  <display-name>Archetype Created Web Application</display-name>
  <context-param>
    <param-name>contextClass</param-name>
    <param-value>org.springframework.web.context.support.AnnotationConfigWebApplicationContext</param-value>
  </context-param>

  <context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>ch7.RootConfig</param-value>
  </context-param>
  
  <listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
  </listener>
  
  <servlet>
    <servlet-name>appServlet</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <init-param>
      <param-name>contextClass</param-name>
      <param-value>org.springframework.web.context.support.AnnotationConfigWebApplicationContext</param-value>
    </init-param>
    
    <init-param>
      <param-name>contextConfigLocation</param-name>
      <param-value>ch7.WebConfig</param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
  </servlet>
  
  <servlet-mapping>
    <servlet-name>appServlet</servlet-name>
    <url-pattern>/</url-pattern>
  </servlet-mapping>

</web-app>

```

### 处理multipart形式数据








