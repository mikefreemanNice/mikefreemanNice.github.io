---
title: 你真的了解@Resource和@Autowired吗
date: 2019-03-08 13:52:19
tags: [Spring]
comments: false
categories: [Spring]
description: 详细介绍@Resource和@Autowired工作原理和使用方式
---

# 引言
每当我们被问到@Resource和@Autowired的区别，通常会这么回答：@Resource是通过名字注入，@Autowired是通过类型注入。 事实真的如此吗？？？

# 问题引入
项目中采用spring+mybatis框架,同时引入了zebra[https://github.com/Meituan-Dianping/Zebra](https://github.com/Meituan-Dianping/Zebra)，mybatis采用基于2.0方式。

简略代码如下

```
public interface DemoDao<T> {
	void insert(T t);
}

@Repository
public class DemoDaoImpl implements DemoDao{

	// 这里T我们不具体写某个实体
	@Override
	public void insert(T t){
		return getSqlSession().insert("demo.insert", t);
	}
}

@Service
public class DemoService{
	@Resource
	private DemoDao demoDao;
	
	public void insert(T t){
		demoDao.insert(t);
	}
}

```

mapper如下

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="demo.test">
	<insert id="insert" >
      ...
   </insert>
</mapper>
```
如上所示，只是service调用dao，那么结果会调用成功吗？(我们确认mapper配置都没有问题)

结果如下

```
org.apache.ibatis.binding.BindingException: Invalid bound statement (not found): com.test.dao.DemoDao.insert
```

如果采用@Autowired代替@Resource，结果会一样吗？ 请看下文。
# 问题分析
## 结果初探
我们根据结果首先想到的一个问题是，报错信息是找不到对应的statement：com.test.dao.DemoDao.insert，namespace我们命名配置的是demo.test，那么找statement应该是demo.test.insert，这里却发生了变化。
这里还有一个疑问，我们交给spring容器管理的bean name应该是demoDaoImpl而不是demoDao，那么demoDao这个bean从何而来？

## zebra分析
第一感觉是org.mybatis.spring.SqlSessionFactoryBean这个bean会将接口装载为bean吗？但很遗憾，通过代码查看，它只是对mybatis的configuration进行组装，并不涉及bean的组装。

这时候会返现com.dianping.zebra.dao.mybatis.ZebraMapperScannerConfigurer的引入，这个scanner的作用是扫描package，对一些sql和其它配置做一些封装。
代码如下：

```
public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
		if (this.processPropertyPlaceHolders) {
			processPropertyPlaceHolders();
		}
		String[] beanDefinitionNames = registry.getBeanDefinitionNames();
		for (String beanDefinitionName : beanDefinitionNames) {
			BeanDefinition beanDefinition = registry.getBeanDefinition(beanDefinitionName);
			String beanClassName = beanDefinition.getBeanClassName();
			if(SqlSessionFactoryBean.class.getName().equals(beanClassName)){
				beanDefinition.setBeanClassName(FixedSqlSessionFactoryBean.class.getName());
			}
		}

		ZebraClassPathMapperScanner scanner = new ZebraClassPathMapperScanner(registry);
		scanner.setAddToConfig(this.addToConfig);
		scanner.setAnnotationClass(this.annotationClass);
		scanner.setMarkerInterface(this.markerInterface);
		scanner.setSqlSessionFactory(this.sqlSessionFactory);
		scanner.setSqlSessionTemplate(this.sqlSessionTemplate);
		scanner.setSqlSessionFactoryBeanName(this.sqlSessionFactoryBeanName);
		scanner.setSqlSessionTemplateBeanName(this.sqlSessionTemplateBeanName);
		scanner.setResourceLoader(this.applicationContext);
		scanner.setBeanNameGenerator(this.nameGenerator);
		scanner.registerFilters();
		scanner.scan(StringUtils.tokenizeToStringArray(this.basePackage,
				ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS));
	}
```
注意一点scanner.registerFilters();这是配置扫描的过滤策略，来跟下代码

```
public void registerFilters() {
		boolean acceptAllInterfaces = true;
		...
		if (acceptAllInterfaces) {
			// default include filter that accepts all classes
			addIncludeFilter(new TypeFilter() {
				public boolean match(MetadataReader metadataReader, MetadataReaderFactory metadataReaderFactory)
				      throws IOException {
					return true;
				}
			});
		}
		...
	}
```
有兴趣的读者可以继续查看com.dianping.zebra.dao.mybatis.ZebraClassPathMapperScanner#doScan的代码，其实到这里已经知道了问题，这个scanner在扫包的时候会将满足条件的接口装载为bean，bean名称为接口名称。
到这里我们明确了一个问题，我们spring启动时，针对于DemoDao会生成两个实现类，一个是DemoDao,一个是DemoDaoImpl。 （这个问题如果我们采用mybatis3.0的方式其实可以避免）

## @Resource分析
既然有两个实现类，@Resource为什么会注入zebra生成的，而不是用我们自定义的DemoDaoImpl？这个问题我们要回归到spring，spring对@Resource的解析在这个类中：
org.springframework.context.annotation.CommonAnnotationBeanPostProcessor。
代码如下：

```
private InjectionMetadata buildResourceMetadata(final Class<?> clazz) {
		LinkedList<InjectionMetadata.InjectedElement> elements = new LinkedList<InjectionMetadata.InjectedElement>();
		Class<?> targetClass = clazz;

		do {
			final LinkedList<InjectionMetadata.InjectedElement> currElements =
					new LinkedList<InjectionMetadata.InjectedElement>();

			ReflectionUtils.doWithLocalFields(targetClass, new ReflectionUtils.FieldCallback() {
				@Override
				public void doWith(Field field) throws IllegalArgumentException, IllegalAccessException {
					if (webServiceRefClass != null && field.isAnnotationPresent(webServiceRefClass)) {
						if (Modifier.isStatic(field.getModifiers())) {
							throw new IllegalStateException("@WebServiceRef annotation is not supported on static fields");
						}
						currElements.add(new WebServiceRefElement(field, field, null));
					}
					else if (ejbRefClass != null && field.isAnnotationPresent(ejbRefClass)) {
						if (Modifier.isStatic(field.getModifiers())) {
							throw new IllegalStateException("@EJB annotation is not supported on static fields");
						}
						currElements.add(new EjbRefElement(field, field, null));
					}
					else if (field.isAnnotationPresent(Resource.class)) {
						if (Modifier.isStatic(field.getModifiers())) {
							throw new IllegalStateException("@Resource annotation is not supported on static fields");
						}
						if (!ignoredResourceTypes.contains(field.getType().getName())) {
							currElements.add(new ResourceElement(field, field, null));
						}
					}
				}
			});
			...
}
```
```
private class ResourceElement extends LookupElement {

		private final boolean lazyLookup;

		public ResourceElement(Member member, AnnotatedElement ae, PropertyDescriptor pd) {
			super(member, pd);
			Resource resource = ae.getAnnotation(Resource.class);
			String resourceName = resource.name();
			Class<?> resourceType = resource.type();
			this.isDefaultName = !StringUtils.hasLength(resourceName);
			if (this.isDefaultName) {
				resourceName = this.member.getName();
				if (this.member instanceof Method && resourceName.startsWith("set") && resourceName.length() > 3) {
					resourceName = Introspector.decapitalize(resourceName.substring(3));
				}
			}
			else if (embeddedValueResolver != null) {
				resourceName = embeddedValueResolver.resolveStringValue(resourceName);
			}
			if (resourceType != null && Object.class != resourceType) {
				checkResourceType(resourceType);
			}
			else {
				// No resource type specified... check field/method.
				resourceType = getResourceType();
			}
			this.name = resourceName;
			this.lookupType = resourceType;
			String lookupValue = (lookupAttribute != null ?
					(String) ReflectionUtils.invokeMethod(lookupAttribute, resource) : null);
			this.mappedName = (StringUtils.hasLength(lookupValue) ? lookupValue : resource.mappedName());
			Lazy lazy = ae.getAnnotation(Lazy.class);
			this.lazyLookup = (lazy != null && lazy.value());
		}

		@Override
		protected Object getResourceToInject(Object target, String requestingBeanName) {
			return (this.lazyLookup ? buildLazyResourceProxy(this, requestingBeanName) :
					getResource(this, requestingBeanName));
		}
	}
```
大概解释下：@Resource注解会先判断注解是否指定name，如果没有，则会取属性的名称，即@Resource DemoDao demoDao 中的demoDao，如果这个名字找不到bean，则会通过类型来判断。回到最开始的问题，zebra恰好为我们生成了demoDao，所以我们在service中注入了demoDao，而找不到mapper对应的statement。

解决方案： 

- 指定@Resource的name，@Resource(name="demoDaoImpl")
- 将demoDao改名，让其按类型匹配，@Resource private DemoDao test;

## @Autowired分析
我们回归到最开始的问题，当我们代码改为如下，结果会怎么样？

```
@Service
public class DemoService{
	@Autowired
	private DemoDao demoDao;
	
	public void insert(T t){
		demoDao.insert(t);
	}
}
```
执行结果：

```
org.apache.ibatis.binding.BindingException: Invalid bound statement (not found): com.test.dao.DemoDao.insert
```

依旧会有这个问题，为什么？

我们这里有两个bean，对于Autowried来说，当它发现有两个实现类时会报“org.springframework.beans.factory.NoUniqueBeanDefinitionException: No qualifying bean of type XXX”，但是在这之前它会优先去找属性字段名称的bean，而在这个例子中它恰好可以找到demoDao的bean，而不是我们自己定义的DemoDaoImpl，因此报错。

解决方案
- 在DemoDaoImpl上加入注解@Primary，让Autowired优先找这个bean

# 总结
1 看似简单的问题其实很复杂。
2 有幸的读者可以看看spring对Autowired的解析：
org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor。