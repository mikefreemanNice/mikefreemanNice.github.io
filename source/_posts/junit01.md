---
title: 重写Junit Runner
date: 2019-01-15 16:49:12
tags: [Java]
comments: false
categories: [Java]
description: junit如何实现通用的过滤器过滤某些不想执行的逻辑。
---

# 问题背景
通过jenkins或者其他ci定时执行junit test，会碰到这种情况，比如某些跨时间的单测会导致不同的结果，如何不侵入代码完成这些异常情况的过滤？

# 解决思路
通过重写org.junit.runners.BlockJUnit4ClassRunner的ignore逻辑进行过滤。

# 具体过程
由于测试的项目需要集成Spring，单测需要集成spring的环境配置。
## 构造自己的Runner
代码如下：

```
public class TestBlockingRunner extends SpringJUnit4ClassRunner {

    public TestBlockingRunner(Class<?> klass) throws InitializationError {
        super(klass);
    }

    @Override
    protected void runChild(FrameworkMethod frameworkMethod, RunNotifier notifier) {
        Description description = this.describeChild(frameworkMethod);

        if (isIgnored(frameworkMethod)) {
            notifier.fireTestIgnored(description);
        } else {
            super.runChild(frameworkMethod, notifier);
        }
    }

    @Override
    protected boolean isIgnored(FrameworkMethod child) {
        return isIgnored(child.getMethod()) || super.isIgnored(child);
    }

    private boolean isIgnored(Method method) {
        return isIgnored((RunIf) method.getAnnotation(RunIf.class));
    }

    private boolean isIgnored(RunIf runIf) {
        if (runIf == null) {
            return false;
        } else {
            Class conditionClass = runIf.value();

            try {
                RunIfCondition condition = (RunIfCondition) conditionClass.newInstance();
                return !condition.apply();
            } catch (InstantiationException | IllegalAccessException var3) {
                throw new AssertionError(var3);
            }
        }
    }
}
```
这里重新实现了runChild和isIgnored两个方法，通过查找方法的RunIf注解，来判断当前的过滤条件，过滤条件同意实现RunIfCondition接口。

## RunIf注解
代码如下：

```
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD, ElementType.TYPE})
@Documented
public @interface RunIf {
    Class<? extends RunIfCondition> value();
}
```

## RunIfCondition接口
代码如下：

```
public interface RunIfCondition {
    boolean apply();
}
```

## 过滤条件实现
如果我想在某几个时间点不执行junit test，实现逻辑如下：

```
public class TestCondition implements RunIfCondition {

    // 由于跨天时间执行test有数量变化问题，可以直接过滤这几个时间点
    private static final List<Integer> IGNORE_HOURS = Lists.newArrayList(22, 23, 0, 1);


    // 具体过滤逻辑
    @Override
    public boolean apply() {

        LocalTime localTime = LocalTime.now();

        if (IGNORE_HOURS.contains(localTime.getHour())) {

            return Boolean.FALSE;

        }

        return Boolean.TRUE;
    }
}
```

## 单测用例

```
@RunWith(TestBlockingRunner.class)
@ContextConfiguration(locations = "classpath:applicationContext-dev.xml")
@Rollback
public class Test {

    @Before
    public void setUp() throws TException {
  
    }

    @Test
    @RunIf(TestCondition.class)
    @Transactional
    @Sql("/sql/test.sql")
    public void test() {
    	// 代码省略
    	...
       System.out.println("通过改变时间观察是否有输出。");
    }
}
```