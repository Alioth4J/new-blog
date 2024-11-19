---
title: 手写 MiniSpring 之 IoC
description: Spring Framework 的设计与迭代
date: 2024-08-24
slug: mini-spring-ioc
image: 
categories:
    - Java
    - Spring
---

> *每当仰望山巅，我就感到如坠深渊，于是我便开始攀登*
## 配置与代码分离：XML
在 XML 文件中配置 Bean，可以将应用程序的可变部分与固定的框架代码分开，增加框架的灵活性，同时便于后续框架的扩展。  
一个简单的例子：
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<beans>
    <bean id="xxxid" class="com.alioth4j.minispring.XxxClass" />
</beans>
```
## 面向对象：XML 对应的 Java 类
框架代码中需要一个对应于 XML 配置文件的类，因此创建 `ClassPathXmlResource`，我们的 mini-spring 为了简便，在构造器中使用 dom4j 完成对 XML 文件的装载工作，实际上 Spring Framework 使用 `ResourceLoader` 加载资源。
```java
public class ClassPathXmlResource {

    Document document;
    Element rootElement;
    Iterator<Element> elementIterator; // root element iterator

    public ClassPathXmlResource(String fileName) {
        SAXReader saxReader = new SAXReader();
        URL xmlPath = this.getClass().getClassLoader().getResource(fileName);
        try {
            this.document = saxReader.read(xmlPath);
            this.rootElement = document.getRootElement();
            this.elementIterator = this.rootElement.elementIterator();
        } catch (DocumentException e) {
            e.printStackTrace();
        }
    }

}
```
## 策略模式：类的家族
对于同一项工作，由于上下文的不同，具体策略也不同。  
采用 **Interface - Abstract Class - Class 模式** 对每一种类的家族结构进行设计。以后所有的类都要遵循这个结构。  
例如 Resource 家族：
```java
public interface Resource {...}
public abstract class AbstractResource implements Resource {...}
public class ClassPathXmlResource extends AbstractResource {...}
```
## 面向对象：XML 解析后 Bean 对应的类
创建 `BeanDefinition`，用于描述一个 Bean 的元数据。这个类的属性与 `<bean>` 标签中的属性对应，并提供构造器和 getter、setter 方法
```java
public class BeanDefinition {

    private String id;
    private String className;

    public BeanDefinition(String id, String className) {
        this.id = id;
        this.className = className;
    }

    // 省略 getter 和 setter

}
```
## 单一职责原则：专注 Resource -> BeanDefinition 的类
专门的工作应该交给专门的类进行，同时一个类应该只有一个职责，职责分离以降低耦合。  
创建 `XmlBeanDefinitionReader`：
```java
public class XmlBeanDefinitionReader {

    SimpleBeanFactory simpleBeanFactory;

    public XmlBeanDefinitionReader(SimpleBeanFactory simpleBeanFactory) {
        this.simpleBeanFactory = simpleBeanFactory;
    }

    public void loadBeanDefinitions(Resource resource) {
        while (resource.hasNext()) {
            Element element = (Element) resource.next();
            String beanID = element.attributeValue("id");
            String beanClassName = element.attributeValue("class");
            BeanDefinition beanDefinition = new BeanDefinition(beanID, beanClassName);
            this.simpleBeanFactory.registerBeanDefinition(beanDefinition);
        }
    }

}
```
## IoC 容器：创建和管理 Bean
BeanFactory 接口：
```java
public interface BeanFactory {

    void registerBean(String beanName, Object obj);
    Object getBean(String beanName) throws BeansException;
    Boolean containsBean(String name);
    
}
```
BeanDefinitionRegistry 接口：
```java
public interface BeanDefinitionRegistry {

    void registerBeanDefinition(String name, BeanDefinition bd);
    BeanDefinition getBeanDefinition(String name);
    boolean containsBeanDefinition(String name);
    void removeBeanDefinition(String name);

}
```
SingletonBeanRegistry 接口：
```java
public interface SingletonBeanRegistry {

    void registerSingleton(String beanName, Object singletonObject);
    Object getSingleton(String beanName);
    boolean containsSingleton(String beanName);
    String[] getSingletonNames();

}
```
DefaultSingletonBeanRegistry 类：
```java
public class DefaultSingletonBeanRegistry implements SingletonBeanRegistry {

    // 存放单例的集合
    protected Map<String, Object> singletons = new ConcurrentHashMap<>(256);
    protected List<String> beanNames = new ArrayList<>();
    
    public void registerSingleton(String beanName, Object singletonObject) {
        synchronized (this.singletons) {
            Object oldObject = this.singletons.putIfAbsent(beanName, singletonObject);
            if (oldObject != null) {
                throw new ...
            }
            this.beanNames.add(beanName);
        }
    }

    public Object getSingleton(String beanName) {
        return this.singletons.get(beanName);
    }

    public boolean containsSingleton(String beanName) {
        return this.singletons.containsKey(beanName);
    }

    public String[] getSingletonNames() {
        return (String[]) this.beanNames.toArray();
    }

    protected void removeSingleton(String beanName) {
        synchronized (this.singletons) {
            this.beanNames.remove(beanName);
            this.singletons.remove(beanName);
        }
    }

}
```
SimpleBeanFactory 类：
```java
public class SimpleBeanFactory extends DefaultSingletonBeanRegistry implements BeanFactory, BeanDefinitionRegistry {

    // 存放 BeanDefinition 的集合
    private Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>(256);
    private List<String> beanDefinitionNames = new ArrayList<>();

    public SimpleBeanFactory() {
    }

    // 容器的核心方法，暂时还没有使用三级缓存
    public Object getBean(String beanName) throws BeansException {
        // 先尝试从直接 singletons 中直接拿 Bean 实例
        Object singleton = this.getSingleton(beanName);
        // 没有则获取 BeanDefinition 创建实例
        if (singleton == null) {
            BeanDefinition beanDefinition = beanDefinitions.get(beanName);
            if (beanDefinition == null) {
                throw new BeansException("No such bean: " + beanName);
            }
            try {
                singleton = Class.forName(beanDefinition.getClassName()).newInstance();
            } catch (ClassNotFoundException e) {
                e.printStackTrace();
            }
            this.registerSingleton(beanName, singleton);
        }
        return singleton;
    }

    // Override methods omitted

}
```
## 整合组件：容器的启动过程
ClassPathXmlApplicationContext：
```java
public class ClassPathXmlApplicationContext implements BeanFactory {

    BeanFactory beanFactory;

    // Context 负责整合容器的启动过程
    public ClassPathXmlApplicationContext(String fileName) {
        Resource resource = new ClassPathXmlResource(fileName);
        BeanFactory beanFactory = new SimpleBeanFactory();
        XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(beanFactory);
        reader.loadBeanDefinitions(resource);
        this.beanFactory = beanFactory;
    }

    // 对外提供一个 getBean 方法
    @Override
    public Object getBean(String beanName) throws BeansException {
        return this.beanFactory.getBean(beanName);
    }

    // other Override methods omitted

}
```
## 增强 IoC：构造器注入和 setter 注入
### 增加 XML 中标签和属性
`ref` 属性表示注入另一个 Bean
```xml
<beans>
    <bean id="id1" class="com.alioth4j.minispring.ExampleClass1" />
    <bean id="id2" class="com.alioth4j.minispring.ExampleClass2">
        <!-- 构造器注入 -->
        <constructor-arg type="String" name="name" value="value1" />
        <!-- setter 注入 -->
        <property type="int" name="property" ref="id1" />
    </bean>
</beans>
```
### 增加对应的类
ArgumentValue（构造器注入）：
```java
public class ArgumentValue {

    private String type;
    private String name;
    private Object value;
    private boolean isRef;

    public ArgumentValue(String type, String name, Object value, boolean isRef) {
        this.type = type;
        this.name = name;
        this.value = value;
        this.isRef = isRef;
    }

    // getter and setter omitted

}
```
PropertyValue（setter 注入）：
```java
public class PropertyValue {

    private String type;
    private String name;
    private Object value;
    private boolean isRef;

    public PropertyValue(String type, String name, Object value, boolean isRef) {
        this.type = type;
        this.name = name;
        this.value = value;
        this.isRef = isRef
    }

    // getter and setter omitted

}
```
ArgumentValues（构造器注入）：
```java
public class ArgumentValues {

    private final List<ArgumentValue> argumentValueList = new ArrayList<>();

    public ArgumentValues() {
    }

    public void addArgumentValue(ArgumentValue argumentValue) {
        this.argumentValueList.add(argumentValue);
    }

    // ...

}
```
PropertyValues（setter 注入）：
```java
public class PropertyValues {

    private final List<PropertyValue> propertyValueList = new ArrayList<>();

    public PropertyValues() {
    }

    public void addPropertyValue(PropertyValue pv) {
        this.propertyValueList.add(pv);
    }

    // ...

}
```
### BeanDefinition 中增加属性
```java
public class BeanDefinition {

    // ...

    private ArgumentValues constructorArgumentValues;
    private PropertyValues propertyValues;

    // ...

}
```
### XmlBeanDefinitionReader 中增加解析过程
```java
public class XmlBeanDefinitionReader {

    // ...

    public void loadBeanDefinitions(Resource resource) {
        while (resource.hasNext()) {
            // 对应一整个 <bean> 标签
            Element element = (Element) resource.next();
            // <bean> 内的属性
            String beanID = element.attributeValue("id");
            String beanClassName = element.attributeValue("class");
            BeanDefinition beanDefinition = new BeanDefinition(beanID, beanClassName);

            // <bean> 中嵌套的标签
            // <constructor-arg>
            List<Element> constructorElements = element.elements("constructor-arg");
            ArgumentValues avs = new ArgumentValues();
            for (Element e : constructorElements) {
                String aType = e.attributeValue("type");
                String aName = e.attributeValue("name");
                String aValue = e.attributeValue("value");
                avs.addArgumentValue(new ArgumentValue(aType, aName, aValue));
            }
            beanDefinition.setConstructorArgumentValues(avs);

            // <property>
            List<Element> propertyElements = element.elements("property");
            PropertyValues pvs = new PropertyValues();
            List<String> refs = new ArrayList<>();
            for (Element e : propertyElements) {
                String pType = e.attributeValue("type");
                String pName = e.attributeValue("name");
                String pValue = e.attributeValue("value");
                String pRef = e.attributeValue("ref");
                String pV = "";
                boolean isRef = false;
                if (pValue != null && !pValue.equals("")) {
                    isRef = false;
                    pV = pValue;
                } else if (pRef != null && !pRef.equals("")) {
                    isRef = true;
                    pV = pRef;
                    refs.add(pRef);
                }
                pvs.addPropertyValue(new PropertyValue(pType, pName, pValue, isRef));
            }
            beanDefinition.setPropertyValues(pvs);
            beanDefinition.setDependsOn(refs.toArray(new String[0]));

            this.beanFactory.registerBeanDefinition(beanID, beanDefinition);
        }
    }

}
```
### SimpleBeanFactory#getBean 进行注入  
为了解决循环依赖问题，使用三级缓存（具体是第二级缓存解决了 setter 注入的循环依赖问题）：  
第一级缓存：singletonObjects - 完全创建好的单例 Bean 实例  
第二级缓存：earlySingletonObjects - 早期的未完全初始化（尚未填充属性）的 Bean 实例  
第三级缓存：objectFactories - 创建 Bean 的工厂对象  

在 MiniSpring 中，第三级缓存使用 `BeanDefinition + 反射` 直接创建 Bean  
```java
public class SimpleBeanFactory extends DefaultSingletonBeanRegistry implements BeanFactory, BeanDefinitionRegistry {

    private Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>();
    private List<String> beanDefinitionNames = new ArrayList<>();

    // 二级缓存
    private final Map<String, Object> earlySingletonObjects = new HashMap<>(16);

    public SimpleBeanFactory() {
    }

    /**
     * 激活 IoC 容器
     */
    public void refresh() {
        for (String beanName : beanDefinitionNames) {
            try {
                getBean(beanName);
            } catch (BeansException e) {
                e.printStackTrace();
            }
        }
    }

    @Override
    public Object getBean(String beanName) throws BeansException {
        Object singleton = this.getSingleton(beanName);

        if (singleton == null) {
            singleton = this.earlySingletonObjects.get(beanName);
            if (singleton == null) {
                BeanDefinition bd = this.beanDefinitionMap.get(beanName);
                singleton = createBean(bd);
                this.registerBean(beanName, singleton);

                // BeanPostProcessor
                // step1: postProcessBeforeInitialization
                // step2: afterPropertiesSet
                // step3: init-method
                // step4: postProcessAfterInitialization

            }
        }
        if (singleton == null) {
            throw new BeansException("No such bean: " + beanName);
        }
        return singleton;
    }

    private Object createBean(BeanDefinition bd) {
        Object obj = doCreateBean(bd);
        // 放入earlySingletonObjects中
        this.earlySingletonObjects.put(bd.getId(), obj);
        Class<?> clz = null;
        try {
            clz = Class.forName(bd.getClassName());
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
        handleProperties(bd, clz, obj);
        return obj;
    }

    /**
     * 构造器注入创建Bean实例
     * @param bd
     * @return
     */
    private Object doCreateBean(BeanDefinition bd) {
        Class<?> clz = null;
        try {
            clz = Class.forName(bd.getClassName());
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
        Object obj = null;

        // 构造器注入相关
        ArgumentValues argumentValues = bd.getConstructorArgumentValues();
        if (!argumentValues.isEmpty()) {
            // 构造器注入
            Class<?>[] paramTypes = new Class<?>[argumentValues.getArgumentCount()];
            Object[] paramValues = new Object[argumentValues.getArgumentCount()];
            for (int i = 0; i < argumentValues.getArgumentCount(); i++) {
                ArgumentValue argumentValue = argumentValues.getIndexedArgumentValue(i);
                if ("String".equals(argumentValue.getType()) || "java.lang.String".equals(argumentValue.getType())) {
                    paramTypes[i] = String.class;
                    paramValues[i] = argumentValue.getValue();
                } else if ("Integer".equals(argumentValue.getType()) || "java.lang.Integer".equals(argumentValue.getType())) {
                    paramTypes[i] = Integer.class;
                    paramValues[i] = argumentValue.getValue();
                } else if ("int".equals(argumentValue.getType())) {
                    paramTypes[i] = int.class;
                    paramValues[i] = argumentValue.getValue();
                } else {
                    paramTypes[i] = String.class;
                    paramValues[i] = argumentValue.getValue();
                }
            }
            Constructor<?> con = null;
            try {
                con = clz.getConstructor(paramTypes);
            } catch (NoSuchMethodException e) {
                e.printStackTrace();
            }
            try {
                obj = con.newInstance(paramValues);
            } catch (InstantiationException e) {
                e.printStackTrace();
            } catch (IllegalAccessException e) {
                e.printStackTrace();
            } catch (InvocationTargetException e) {
                e.printStackTrace();
            }
        } else {
            try {
                obj = clz.newInstance();
            } catch (InstantiationException e) {
                e.printStackTrace();
            } catch (IllegalAccessException e) {
                e.printStackTrace();
            }
        }
        return obj;
    }

    /**
     * setter 注入相关
     */
    private void handleProperties(BeanDefinition bd, Class<?> clz, Object obj) {
        PropertyValues propertyValues = bd.getPropertyValues();
        if (!propertyValues.isEmpty()) {
            // setter方法注入
            for (int i = 0; i < propertyValues.size(); i++) {
                PropertyValue propertyValue = propertyValues.getPropertyValueList().get(i);
                String name = propertyValue.getName();
                String type = propertyValue.getType();
                Object value = propertyValue.getValue();
                boolean isRef = propertyValue.getIsRef();
                Class<?>[] paramTypes = new Class<?>[1];
                Object[] paramValues = new Object[1];
                if (!isRef) {
                    if ("String".equals(type) || "java.lang.String".equals(type)) {
                        paramTypes[0] = String.class;
                    } else if ("Integer".equals(type) || "java.lang.Integer".equals(type)) {
                        paramTypes[0] = Integer.class;
                    } else if ("int".equals(type)) {
                        paramTypes[0] = int.class;
                    } else {
                        paramTypes[0] = String.class;
                    }
                    paramValues[0] = value;
                } else {
                    try {
                        paramTypes[0] = Class.forName(type);
                    } catch (ClassNotFoundException e) {
                        e.printStackTrace();
                    }
                    try {
                        paramValues[0] = getBean((String)value);
                    } catch (BeansException e) {
                        e.printStackTrace();
                    }
                }
                String methodName = "set" + name.substring(0, 1).toUpperCase() + name.substring(1);
                Method setter = null;
                try {
                    setter = clz.getMethod(methodName, paramTypes);
                } catch (NoSuchMethodException e) {
                    e.printStackTrace();
                }
                try {
                    setter.invoke(obj, paramValues);
                } catch (IllegalAccessException e) {
                    e.printStackTrace();
                } catch (InvocationTargetException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    // other methods omitted        

}
```
### 创建所有的 Bean
SimpleBeanFactory 中添加 refresh 方法：
```java
public class SimpleBeanFactory {

    // ...

    public void refresh() {
        for (String beanName : this.beanDefinitionNames) {
            try {
                getBean(beanName);
            } catch (BeansException e) {
                e.printStackTrace();
            }
        }
    }

    // ...

}
```
在 ClassPathXmlApplicationContext 中加上刷新相关代码：
```java
public class ClassPathXmlApplicationContext {

    // ...
    
    public ClassPathXmlApplicationContext(String fileName, boolean isRefresh) {
        Resource resource = new ClassPathXmlResource(fileName);
        SimpleBeanFactory simpleBeanFactory = new SimpleBeanFactory();
        XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(simpleBeanFactory);
        reader.loadBeanDefinitions(resource);
        this.beanFactory = simpleBeanFactory;
        if (isRefresh) {
            this.beanFactory.refresh();
        }
    }

    // ...

}
```
## 增强 IoC：注解注入
使用 BeanPostProcessor，反射获取类上所有属性，逐个判断有没有 `@Autowired` 注解，如果有，从 BeanFactory 中获取要注入的 Bean 实例，使用反射注入。  

### @Autowired 注解
```java
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Autowired {
}
```
### BeanPostProcessor 家族
BeanPostProcessor：
```java
public interface BeanPostProcessor {

    Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;
    Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;
    void setBeanFactory(BeanFactory beanFactory);

}
```
AutowiredAnnotationBeanPostProcessor：
```java
public class AutowiredAnnotationBeanPostProcessor implements BeanPostProcessor {

    private BeanFactory beanFactory;

    public BeanFactory getBeanFactory() {
        return beanFactory;
    }

    @Override
    public void setBeanFactory(BeanFactory beanFactory) {
        this.beanFactory = beanFactory;
    }

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        Field[] fields = bean.getClass().getDeclaredFields();
        if (fields != null) {
            for (Field field : fields) {
                boolean isAutowired = field.isAnnotationPresent(Autowired.class);
                if (isAutowired) {
                    String fieldName = field.getName();
                    Object autowiredObj = this.getBeanFactory().getBean(fieldName);
                    field.setAccessible(true);
                    try {
                        field.set(bean, autowiredObj);
                    } catch (IllegalAccessException e) {
                        e.printStackTrace();
                    }
                }
            }
        }
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        return null;
    }

}
```
### BeanFactory 家族
原来的 SimpleBeanFactory 变为 AbstractBeanFactory，目的是把 postProcess 相关的两个方法作为抽象方法交给具体实现类实现：
```java
public abstract class AbstractBeanFactory {

    // ...

    @Override
    public Object getBean(String beanName) throws BeansException {
        Object singleton = this.getSingleton(beanName);

        if (singleton == null) {
            singleton = this.earlySingletonObjects.get(beanName);
            if (singleton == null) {
                BeanDefinition bd = this.beanDefinitionMap.get(beanName);
                singleton = createBean(bd);
                this.registerBean(beanName, singleton);

                // BeanPostProcessor
                // step1: postProcessBeforeInitialization
                singleton = applyBeanPostProcessorsBeforeInitialization(singleton, beanName);
                // step2: afterPropertiesSet

                // step3: init-method
                if (bd.getInitMethodName() != null && !bd.getInitMethodName().equals("")) {
                    invokeInitMethod(bd, singleton);
                }
                // step4: postProcessAfterInitialization
                applyBeanPostProcessorsAfterInitialization(singleton, beanName);

                this.removeSingleton(beanName);
                this.registerBean(beanName, singleton);
            }
        }
        if (singleton == null) {
            throw new BeansException("No such bean: " + beanName);
        }
        return singleton;
    }

    public abstract Object applyBeanPostProcessorsBeforeInitialization(Object singleton, String beanName);

    public abstract Object applyBeanPostProcessorsAfterInitialization(Object singleton, String beanName);

    // ...

}
```
AutowireCapableBeanFactory：
```java
public class AutowireCapableBeanFactory extends AbstractBeanFactory {

    private final List<AutowiredAnnotationBeanPostProcessor> beanPostProcessors = new ArrayList<>();

    @Override
    public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName) throws BeansException {
        Object result = existingBean;
        for (AutowiredAnnotationBeanPostProcessor beanProcessor : getBeanPostProcessors()) {
            beanProcessor.setBeanFactory(this);
            result = beanProcessor.postProcessBeforeInitialization(result, beanName);
            if (result == null) {
                return result;
            }
        }
        return result;
    }

    @Override
    public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName) throws BeansException {
        Object result = existingBean;
        for (BeanPostProcessor beanProcessor : getBeanPostProcessors()) {
            result = beanProcessor.postProcessAfterInitialization(result, beanName);
            if (result == null) {
                return result;
            }
        }
        return result;
    }

    public List getBeanPostProcessors() {
        return this.beanPostProcessors;
    }

    public void addBeanPostProcessor(AutowiredAnnotationBeanPostProcessor beanPostProcessor) {
        this.beanPostProcessors.remove(beanPostProcessor);
        this.beanPostProcessors.add(beanPostProcessor);
    }

    // ...

}
```
### ClassPathXmlApplicationContext 相关
`refresh` 方法中为当前 beanFactory new 一个 AutowiredAnnotationBeanPostProcessor 并注册：
```java
public class ClassPathXmlApplicationContext {
    
    // ...

    public void refresh() throws BeansException, IllegalStateException {
        registerBeanPostProcessor(this.beanFactory);
        onRefresh();
    }

    private void registerBeanPostProcessor(AutowireCapableBeanFactory beanFactory) {
        beanFactory.addBeanPostProcessor(new AutowiredAnnotationBeanPostProcessor());
    }

    private void onRefresh() {
        this.beanFactory.refresh();
    }

}
```
## 接口隔离：拆分接口
拆分出 `ListableBeanFactory`、`ConfigurableBeanFactory`，新增 `ConfigurableListableBeanFactory`  
`AutowireCapableBeanFactory` 改为接口，增加 `AbstractAutowireCapableBeanFactory` 替代原有实现  
## 扩展：更多支持
- Environment
- ApplicationEvent
- BeansException
- ...
##  集成：应用上下文
ApplicationContext：
```java
public interface ApplicationContext
        extends EnvironmentCapable, ListableBeanFactory, ConfigurableBeanFactory, ApplicationEventPublisher {

    // ...

}
```
抽象类 `AbstractApplicationContext`、具体实现类 `ClassPathXmlApplicationContext`  

至此 IoC 容器完成  
