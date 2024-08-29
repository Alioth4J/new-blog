---
title: 手写 MiniSpring 之 MVC
description: Spring Framework 的设计与迭代
date: 2024-08-28
slug: mini-spring-mvc
image: 
categories:
    - Java
    - Spring
---

> *每当仰望山巅，我就感到如坠深渊，于是我便开始攀登*
## 复用：初始的 MVC
MiniSpring 使用 Tomcat 作为服务器，初始的 MVC 与 IoC 类似，可以复用 IoC 的代码  
beans.xml：
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<beans>
    <bean id="/url" class="com.alioth4j.XxxClass" value="methodName"/>
</beans>
```
与 Servlet 相关的 web.xml：
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<web-app xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xmlns:web="http://xmlns.jcp.org/xml/ns/javaee"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_2_5.xsd"
         id="WebApp_ID">
    <servlet>
        <servlet-name>dispatcherServlet</servlet-name>
        <servlet-class>com.alioth4j.web.DispatcherServlet</servlet-class>
        <init-param>    
            <param-name>servletConfigLocation</param-name>
            <param-value>/WEB-INF/servlet.xml</param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>
    <servlet-mapping>
        <servlet-name>dispatcherServlet</servlet-name>
        <url-pattern>/</url-pattern>
    </servlet-mapping>
</web-app>
```
`ClassPathXmlResource`、`XmlConfigReader` 复用 IoC 容器中的同名组件  
`MappingValue` 类似 `BeanDefinition`，`XmlConfigReader` 类似 `XmlBeanDefinitionReader`  
`DispatcherServlet` 类似初始 IoC 中的 `ClassPathXmlAppllicationContext`  

其中 `DispatcherServlet` 需要 extends javax.servlet.http.HttpServlet 并重写 init 方法  
## 增强 MVC：组件扫描和注解支持
servlet.xml：
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<components>
    <component-scan base-package="com.alioth4j.xxx" />
</components>
```
@RequestMapping：
```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface RequestMapping {

    String value() default "";

}
```
工具类 XmlScanComponentUtil：
```java
public class XmlScanComponentUtil {

    public static List<String> getNodeValue(URL xmlPath) {
        List<String> packages = new ArrayList<>();
        // dom4j
        SAXReader saxReader = new SAXReader();
        Document document = null;
        try {
            document = saxReader.read(xmlPath);
        } catch (DocumentException e) {
            e.printStackTrace();
        }
        Element root = document.getRootElement();
        Iterator it = root.elementIterator();
        while (it.hasNext()) {
            Element element = (Element) it.next();
            packages.add(element.attributeValue("base-package"));
        }
        return packages;
    }

}
```
DispatcherServlet 中增加集合：
```java
public class DispatcherServlet extends HttpServlet {

    private List<String> packageNames = new ArrayList<>();
    private List<String> controllerNames = new ArrayList<>();
    private Map<String,Class<?>> controllerClasses = new HashMap<>();
    private Map<String,Object> controllerObjs = new HashMap<>();
    private List<String> urlMappingNames = new ArrayList<>();
    private Map<String,Object> mappingObjs = new HashMap<>();
    private Map<String,Method> mappingMethods = new HashMap<>();

    // ...

}
```
DispatcherServlet#init 中增加使用工具类获取要扫描的包名集合的操作，最后调用 refresh 方法：
```java
public class DispatcherServlet extends HttpServlet {

    // ...

    @Override
    public void init(ServletConfig config) throws ServletException {
        super.init(config);

        // ...

        this.packageNames = XmlScanComponentHelper.getNodeValue(xmlPath);

        // ...

        refresh();
    }

    // ...

}
```
DispatcherServlet#refresh：
```java
public class DispatcherServlet extends HttpServlet {

    // ...

    protected void refresh() {
        initController(); // 初始化 controller     
        initMapping(); // 初始化 url 映射
    }

}
```
DispatcherServlet#initController 装填 Controller 相关集合：
```java
public class DispatcherServlet extends HttpServlet {

    // ...

    protected void initController() {
        this.controllerNames = scanPackages(this.packageNames);
        for (String controllerName : this.controllerNames) {
            Class<?> clz = null;
            try {
                clz = Class.forName(controllerName);
            } catch (ClassNotFoundException e) {
                e.printStackTrace();
            }
            this.controllerClasses.put(controllerName, clz);
            Object obj = null;
            try {
                obj = clz.newInstance();
            } catch (InstantiationException e) {
                e.printStackTrace();
            } catch (IllegalAccessException e) {
                e.printStackTrace();
            }
            this.controllerObjs.put(controllerName, obj);
        }
    }

}
```
DispatcherServlet#initMapping 检查所有 Controller 的所有方法是否带有 `@RequestMapping` 注解，有的话加入到相关集合中：
```java
public class DispatcherServlet extends HttpServlet {

    // ...

    protected void initMapping() {
        for (String controllerName : this.controllerNames) {
            Class<?> clazz = this.controllerClasses.get(controllerName);
            Object obj = this.controllerObjs.get(controllerName);
            Method[] methods = clazz.getDeclaredMethods();
            if(methods!=null){
                for(Method method : methods){
                    boolean isRequestMapping = method.isAnnotationPresent(RequestMapping.class);
                    if (isRequestMapping){
                        String methodName = method.getName();
                        String urlmapping = method.getAnnotation(RequestMapping.class).value();
                        this.urlMappingNames.add(urlmapping);
                        this.mappingObjs.put(urlmapping, obj);
                        this.mappingMethods.put(urlmapping, method);
                    }
                }
            }
        }
    }

}
```
使用递归根据包名获取所有 Controller 名称：
```java
public class DispatcherServlet extends HttpServlet {

    // ...

    private List<String> scanPackages(List<String> packageNames) {
        List<String> tempControllerNames = new ArrayList<>();
        for (String packageName : packageNames){
            tempControllerNames.addAll(scanPackage(packageName));
        }
        return tempControllerNames;
    }

    private List<String> scanPackage(String packageName) {
        List<String> tempControllerNames = new ArrayList<>();
        URL url = this.getClass().getClassLoader().getResource("/" + packageName.replaceAll("\\.", "/"));
        File dir = new File(url.getFile());
        for (File file : dir.listFiles()) {
            if (file.isDirectory()) {
                scanPackage(packageName + "." + file.getName());
            } else {
                // -6 是去掉最后的 ".class"
                String controllerName = packageName + "." + file.getName().substring(0, file.getName() - 6);
                tempControllerNames.add(controllerName);
            }
        }
        return tempControllerNames;
    }

}
```
重写 doGet 方法，用于处理 Http Get 请求，从 HttpServletRequest 中取得请求路径，到集合中取得对应的对象，进行反射调用，最后把结果写会 HttpServletResponse：
```java
public class DispatcherServlet extends HttpServlet {

    // ...

    @Override
    protected void doGet(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        String sPath = request.getServletPath();
        if (!this.urlMappingNames.contains(sPath)) {
            return;
        }
        Object obj = null;
        Object objResult = null;
        try {
            Method method = this.mappingMethods.get(sPath);
            obj = this.mappingObjs.get(sPath);
            objResult = method.invoke(obj);
        } catch (SecurityException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (IllegalArgumentException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        }
        response.getWriter().append(objResult.toString());
    }

}
```
## JavaEE 规范：Servlet 服务器的时序
1. 读取 web.xml 中的配置
2. 启动 Listener
3. 启动 DispatcherServlet

Spring 通过这个时序实现 MVC 的功能  

第一步，web.xml：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://www.w3.org/2001/XMLSchema-instance"
         xmlns:web="http://xmlns.jcp.org/xml/ns/javaee"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/nx/javaee http://java.sun.com/xml/ns/javaee/web-app_2_5.xsd"
         id="WebApp_ID">
    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>applicationContext.xml</param-value>
    </context-param>
    <listener>
        <listener-class>com.alioth4j.web.ContextLoaderListener</listener-class>
    </listener>
    <servlet>
        <servlet-name>dispatcherServlet</servlet-name>
        <servlet-class>com.alioth4j.web.DispatcherServlet</servlet-class>
        <init-param>
            <param-name>servletConfigLocation</param-name>
            <param-value>/WEB-INF/servlet.xml</param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>
    <servlet-mapping>
        <servlet-name>dispatcherServlet</servlet-name>
        <url-pattern>/</url-pattern>
    </servlet-mapping>
</web-app>
```
第二步与第三步，先看 WebApplicationContext 家族  
WebApplicationContext 接口：
```java
public interface WebApplicationContext extends ApplicationContext {

    String ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE = WebApplicationContext.class.getName() + ".ROOT";

    ServletContext getServletContext();
    void setServletContext(ServletContext servletContext);

}
```
Listener 和 DispatcherServlet 中是两级 WebApplicationContext，因此用两个实现类 XmlWebApplicationContext 和 ApplicationConfigWebApplicationContext 分别与之对应  

**这两个 WebApplicationContext 的实现类 `extends ClassPathXmlApplicationContext implements WebApplicationContext`，由此将 IoC 和 MVC 关联起来**  

XmlWebApplicationContext:
```java

public class XmlWebApplicationContext extends ClassPathXmlApplicationContext implements WebApplicationContext {

    private ServletContext servletContext;

    public XmlWebApplicationContext(String fileName) {
        super(fileName);
    }


    @Override
    public ServletContext getServletContext() {
        return this.servletContext;
    }

    @Override
    public void setServletContext(ServletContext servletContext) {
        this.servletContext = servletContext;
    }

}
```
ApplicationConfigWebApplicationContext：
```java
public class AnnotationConfigWebApplicationContext extends ClassPathXmlApplicationContext implements WebApplicationContext {

    private WebApplicationContext parentApplicationContext;
    private ServletContext servletContext;
    DefaultListableBeanFactory beanFactory;
    private final List<BeanFactoryPostProcessor> beanFactoryPostProcessors = new ArrayList<>();

    public AnnotationConfigWebApplicationContext(String fileName) {
        this(fileName, null);
    }

    public AnnotationConfigWebAPplicationContext(String fileName, WebApplicationContext parentApplicationContext) {
        this.parentApplicationContext = parentApplicationContext;
        this.servletContext = parentApplicationContext.getServletContext();
        URL xmlPath = null;
        try {
            xmlPath = this.getServletContext().getResource(fileName);
        } catch (MalformedURLException e) {
            e.printStackTrace();
        }
        List<String> packageNames = XmlScanComponentHelper.getNodeValue(xmlPath);
        List<String> controllerNames = scanPackages(packageNames);
        DefaultListableBeanFactory bf = new DefaultListableBeanFactory();
        this.beanFactory = bf;
        this.beanFactory.setParent(this.parentApplicationContext.getBeanFactory());
        loadBeanDefinitions(controllerNames);
        if (true) {
            refresh();
        }
    }

    private void loadBeanDefinitions(List<String> controllerNames) {
        for (String controllerName : controllerNames) {
            String beanID = controllerName;
            String beanClassName = controllerName;
            BeanDefinition beanDefinition = new BeanDefinition(beanID, beanClassName);
            this.beanFactory.registerBeanDefinition(beanID, beanDefinition);
        }
    }

    private List<String> scanPackages(List<String> packageNames) {
        List<String> tempControllerNames = new ArrayList<>();
        for (String packageName : packageNames){
            tempControllerNames.addAll(scanPackage(packageName));
        }
        return tempControllerNames;
    }

    private List<String> scanPackage(String packageName) {
        List<String> tempControllerNames = new ArrayList<>();
        URL url = this.getClass().getClassLoader().getResource("/" + packageName.replaceAll("\\.", "/"));
        File dir = new File(url.getFile());
        for (File file : dir.listFiles()) {
            if (file.isDirectory()) {
                scanPackage(packageName + "." + file.getName());
            } else {
                String controllerName = packageName + "." + file.getName().replace(".class", "");
                tempControllerNames.add(controllerName);
            }
        }
        return tempControllerNames;
    }

    @Override
    public ServletContext getServletContext() {
        return this.servletContext;
    }

    @Override
    public void setServletContext(ServletContext servletContext) {
        this.servletContext = servletContext;
    }

    @Override
    public ConfigurableListableBeanFactory getBeanFactory() throws IllegalStateException {
        return this.beanFactory;
    }

}
```
因此第二步中的 Listener：
```java
public class ContextLoaderListener implements ServletContextListener {

    private static final String CONFIG_LOCATION_PARAM = "contextConfigLocation";

    private WebApplicationContext context;

    public ContextLoaderListener() {
    }

    public ContextLoaderListener(WebApplicationContext context) {
        this.context = context;
    }

    @Override
    public void contextInitialized(ServletContextEvent event) {
        initWebApplicationContext(event.getServletContext());
    }

    private void initWebApplicationContext(ServletContext servletContext) {
        String sContextLocation = servletContext.getInitParameter(CONFIG_LOCATION_PARAM);
        WebApplicationContext wac = new XmlWebApplicationContext(sContextLocation);
        wac.setServletContext(servletContext);
        this.context = wac;
        servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.context);
    }

    @Override
    public void contextDestroyed(ServletContextEvent event) {
    }

}
```
第三步中的 DispatcherServlet：
```java
public class DispatcherServlet extends HttpServlet {

    public static final String WEB_APPLICATION_CONTEXT_ATTRIBUTE = DispatcherServlet.class.getName() + ".CONTEXT";

    private String servletConfigLocation;

    private WebApplicationContext parentApplicationContext;
    private WebApplicationContext webApplicationContext;

    // ...

    @Override
    public void init(ServletConfig config) throws ServletException {
        super.init(config);
        this.parentApplicationContext = (WebApplicationContext) this.getServletContext().getAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE);
        servletConfigLocation = config.getInitParameter("servletConfigLocation");
        URL xmlPath = null;
        try {
            xmlPath = this.getServletContext().getResource(servletConfigLocation);
        } catch (MalformedURLException e) {
            e.printStackTrace();
        }
        this.packageNames = XmlScanComponentUtil.getNodeValue(xmlPath);
        this.webApplicationContext = new AnnotationConfigWebApplicationContext(servletConfigLocation, this.parentApplicationContext);
        refresh();
    }

    // ...

}
```
## 职责分离：抽取出 url 映射与反射调用代码
MappingRegistry 中存储集合：
```java
public class MappingRegistry {

    private List<String> urlMappingNames = new ArrayList<>();
    private Map<String, Object> mappingObjs = new HashMap<>();
    private Map<String, Method> mappingMethods = new HashMap<>();

    // getter and setter omitted

}
```
HandlerMapping 用于找到对应的反射相关对象：
```java
public interface HandlerMapping {

    HandlerMethod getHandler(HttpServletRequest request) throws Exception;

}
```
实现类 RequestMappingHandlerMapping：
```java
public class RequestMappingHandlerMapping implements HandlerMapping {

    WebApplicationContext wac;

    private final MappingRegistry mappingRegistry = new MappingRegistry();
    
    public RequestMappingHandlerMapping(WebApplicationContext wac) {
        this.wac = wac;
        initMapping();
    }
    
    protected void initMapping() {
        Class<?> clz = null;
        Object obj = null;
        String[] controllerNames = this.wac.getBeanDefinitionNames();
        for (String controllerName : controllerNames) {
            try {
                clz = Class.forName(controllerName);
            } catch (ClassNotFoundException e1) {
                e1.printStackTrace();
            }
            try {
                obj = this.wac.getBean(controllerName);
            } catch (BeansException e) {
                e.printStackTrace();
            }
            Method[] methods = clz.getDeclaredMethods();
            if(methods!=null){
                for(Method method : methods){
                    boolean isRequestMapping = method.isAnnotationPresent(RequestMapping.class);
                    if (isRequestMapping){
                        String methodName = method.getName();
                        String urlmapping = method.getAnnotation(RequestMapping.class).value();
                        this.mappingRegistry.getUrlMappingNames().add(urlmapping);
                        this.mappingRegistry.getMappingObjs().put(urlmapping, obj);
                        this.mappingRegistry.getMappingMethods().put(urlmapping, method);
                    }
                }
            }
        }
    }


    @Override
    public HandlerMethod getHandler(HttpServletRequest request) throws Exception {
        String sPath = request.getServletPath();
    
        if (!this.mappingRegistry.getUrlMappingNames().contains(sPath)) {
            return null;
        }

        Method method = this.mappingRegistry.getMappingMethods().get(sPath);
        Object obj = this.mappingRegistry.getMappingObjs().get(sPath);
        HandlerMethod handlerMethod = new HandlerMethod(method, obj);
        
        return handlerMethod;
    }

}
```
反射相关对象封装在 HandlerMethod 中：
```java
public class HandlerMethod {

    private  Object bean;
    private  Class<?> beanType;
    private  Method method;
    private  MethodParameter[] parameters;
    private  Class<?> returnType;
    private  String description;
    private  String className;
    private  String methodName;
    
    public HandlerMethod(Method method, Object obj) {
        this.setMethod(method);
        this.setBean(obj);
    }

    public Method getMethod() {
        return method;
    }

    public void setMethod(Method method) {
        this.method = method;
    }

    public Object getBean() {
        return bean;
    }

    public void setBean(Object bean) {
        this.bean = bean;
    }

}
```
HandlerAdapter 用于反射调用：
```java
public interface HandlerAdapter {

    void handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception;

}
```
具体实现类：
```java
public class RequestMappingHandlerAdapter implements HandlerAdapter {

    WebApplicationContext wac;

    public RequestMappingHandlerAdapter(WebApplicationContext wac) {
        this.wac = wac;
    }

    @Override
    public void handle(HttpServletRequest request, HttpServletResponse response, Object handler)
            throws Exception {
        handleInternal(request, response, (HandlerMethod) handler);
    }

    private void handleInternal(HttpServletRequest request, HttpServletResponse response, HandlerMethod handler) {
        Method method = handler.getMethod();
        Object obj = handler.getBean();
        Object objResult = null;
        try {
            objResult = method.invoke(obj);
        } catch (IllegalAccessException | IllegalArgumentException| InvocationTargetException e) {
            e.printStackTrace();
        }
        try {
            response.getWriter().append(objResult.toString());
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

}
```
DispatcherServlet 中增加相关代码：
```java
public class DispatcherServlet extends HttpServlet {

    private HandlerMapping handlerMapping;
    private HandlerAdapter handlerAdapter;

    // ...

    @Override
    protected void service(HttpServletRequest request, HttpServletResponse response) {
        request.setAttribute(WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.webApplicationContext);
        try {
            doDispatch(request, response);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
    
    protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
        HttpServletRequest processedRequest = request;
        HandlerMethod handlerMethod = null;
        handlerMethod = this.handlerMapping.getHandler(processedRequest);
        if (handlerMethod == null) {
            return;
        }
        HandlerAdapter ha = this.handlerAdapter;
        ha.handle(processedRequest, response, handlerMethod);
    }

}
```
##  增强 MVC：传入参数自动绑定
增强处为 RequestMappingHandlerAdapter 中反射调用之前，new 出 WebDataBinderFactory 对象，从 HandlerMethod 对象中获取方法参数，对每一个参数执行下列逻辑：
- WebDataBinderFactory#createBinder 得到对应参数的 WebDataBinder  
- WebDataBinder#bind 方法首先将 request 中的参数转换成一个 PropertyValues 对象，之后调用 BeanWrapperImpl#setPropertyValues 方法  
- BeanWrapperImpl#setPropertyValues 中对每一个 propertyValues.getPropertyValues() 调用 setPropertyValue 方法  
- BeanWrapperImpl#setPropertyValue 去获取 PropertyEditor，优先获取自定义 PropertyEditor ，不存在再区获取默认的，接着转换参数类型，并借助内部类 BeanPropertyHandler 通过反射读写参数的值  

PropertyEditorRegistrySupport 中分别存储默认和自定义 PropertyEditor，构造器中初始化默认 PropertyEditor，用户可以通过 registerCustomEditor 方法注册自定义 PropertyEditor  
## 增强 MVC：处理返回结果
返回结果分为两种，视图和 @ResponseBody  
增强处为 RequestMappingHandlerAdapter 中反射条用之后，判断 `@ResponseBody` 是否存在，进入对应逻辑  
### @ResponseBody
反射的返回值进行序列化，写入 response 中即可  
```java
public class RequestMappingHandlerAdapter {

    // ...

    @Override
    protected ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        // ...

        if (invocableMethod.isAnnotationPresent(ResponseBody.class)) {
            // 序列化

        } else {
            // ...
        }
        // ...

    }

}
```
### 视图
Model 本质上是一个 Map  
ModelAndView 中的 view 可能为 View 对象，也可能为逻辑视图名（String）  
ModelAndView：
```java
public class ModelAndView {

    private Map<String, Object> model = new HashMap<>();
    private Object view;
    
    public ModelAndView() {
    }

    public ModelAndView(String viewName) {
        this.view = viewName;
    }

    public ModelAndView(View view) {
        this.view = view;
    }

    public ModelAndView(String viewName, Map<String, ?> modelData) {
        this.view = viewName;
        if (modelData != null) {
            addAllAttributes(modelData);
        }
    }

    public ModelAndView(View view, Map<String, ?> model) {
        this.view = view;
        if (model != null) {
            addAllAttributes(model);
        }
    }

    public ModelAndView(String viewName, String modelName, Object modelObject) {
        this.view = viewName;
        addObject(modelName, modelObject);
    }

    // ...

}
```
没有 `@ResponseBody` 就认为返回的是视图，如果是 ModelAndView 直接返回，是 String 认为是逻辑视图名，构造出 ModelAndView 并返回  
```java
public class RequestMappingHandlerAdapter {

    // ...

    @Override
    protected ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        // ...
                    
        ModelAndView mav = null;
        if (invocableMethod.isAnnotationPresent(ResponseBody.class)) {
            // 序列化

        } else {
            if (returnObj instanceof ModelAndView) {
                mav = (ModelAndView) returnObj;
            } else if(returnObj instanceof String) {
                String viewName = (String) returnObj;
                mav = new ModelAndView();
                mav.setViewName(viewName);
            }
        }
        return mav;
    }

}
```
返回到 DispatcherServlet 中，调用 render 方法对视图进行渲染：  
```java
public class DispatcherServlet extends HttpServlet {

    // ...

    protected void render(HttpServletRequest request, HttpServletResponse response, ModelAndView mv) throws Exception {
        Map<String, Object> modelMap = mv.getModel();
        for (Map.Entry<String, Object> e : modelMap.entrySet()) {
            request.setAttribute(e.getKey(), e.getValue());
        }
        String path = "/" + mv.getViewName() + ".jsp";
        request.getRequestDispatcher(path).forward(request, response);
  }

}
```

至此 MVC 完成  
