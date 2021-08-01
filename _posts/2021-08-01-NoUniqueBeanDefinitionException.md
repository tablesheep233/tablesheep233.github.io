---
layout:     post
title:      "NoUniqueBeanDefinitionException"
subtitle:   " \"Spring 对于 NoUniqueBeanDefinition的处理\""
date:       2021-08-01 19:10:00
header-style: text
author:     "tablesheep"
catalog: true
tags:
    - Java
    - Spring
---

>  “Spring 对于 NoUniqueBeanDefinition（不唯一的Bean）的处理”



## NoUniqueBeanDefinitionException

`Spring IoC`容器有两种实现依赖控制的方法，一种是依赖注入，一种是依赖查找，在默认情况下，当某个类型的`Bean`存在多个的情况下，都可能会发生`NoUniqueBeanDefinitionException`。

使用错误倒推的方法看看依赖注入以及依赖查找对不唯一`Bean`的处理。

## 依赖注入分析

错误点：在依赖注入处理多个`Bean`时，当一切方案都没用（啥都没配置时），会使用`DependencyDescriptor#resolveNotUnique`抛出异常

```java
public class DependencyDescriptor extends InjectionPoint implements Serializable {
......
	@Nullable
	public Object resolveNotUnique(ResolvableType type, Map<String, Object> matchingBeans) throws BeansException {
		throw new NoUniqueBeanDefinitionException(type, matchingBeans.keySet());
	}

......
}

```

分析入口：`DefaultListableBeanFactory#doResolveDependency`

### doResolveDependency

```java
@Nullable
public Object doResolveDependency(DependencyDescriptor descriptor, @Nullable String beanName,
      @Nullable Set<String> autowiredBeanNames, @Nullable TypeConverter typeConverter) throws BeansException {

   InjectionPoint previousInjectionPoint = ConstructorResolver.setCurrentInjectionPoint(descriptor);
   try {
      ......

          //解析是否集合类型（包括Map）类型的Bean，是返回Bean集合
      Object multipleBeans = resolveMultipleBeans(descriptor, beanName, autowiredBeanNames, typeConverter);
      if (multipleBeans != null) {
         return multipleBeans;
      }

       //获取可以进行依赖注入符合条件的所有Bean
      Map<String, Object> matchingBeans = findAutowireCandidates(beanName, type, descriptor);
      if (matchingBeans.isEmpty()) {
         if (isRequired(descriptor)) {
            raiseNoMatchingBeanFound(type, descriptor.getResolvableType(), descriptor);
         }
         return null;
      }

      String autowiredBeanName;
      Object instanceCandidate;

      if (matchingBeans.size() > 1) {
          //若不止一个Bean，从中决定一个Bean进行注入
         autowiredBeanName = determineAutowireCandidate(matchingBeans, descriptor);
         if (autowiredBeanName == null) {
            if (isRequired(descriptor) || !indicatesMultipleBeans(type)) {
                //决定不了，且不允许为空，抛出异常
               return descriptor.resolveNotUnique(descriptor.getResolvableType(), matchingBeans);
            }
            else {
               // In case of an optional Collection/Map, silently ignore a non-unique case:
               // possibly it was meant to be an empty collection of multiple regular beans
               // (before 4.3 in particular when we didn't even look for collection beans).
               return null;
            }
         }
         instanceCandidate = matchingBeans.get(autowiredBeanName);
      }
      else {
         // We have exactly one match.
         Map.Entry<String, Object> entry = matchingBeans.entrySet().iterator().next();
         autowiredBeanName = entry.getKey();
         instanceCandidate = entry.getValue();
      }

      ......
      return result;
   }
   finally {
      ConstructorResolver.setCurrentInjectionPoint(previousInjectionPoint);
   }
}
```



#### findAutowireCandidates

查找合适的`Bean`，会根据`autowireCandidate`进行判断`Bean`是否能用于自动装配。`isAutowireCandidate`方法进过一系列调用会使用`AutowireCandidateResolver`进行判断，默认使用`ContextAnnotationAutowireCandidateResolver`，包含了`autowireCandidate`判断，以及`@Qualifier`判断。

```java
protected Map<String, Object> findAutowireCandidates(
      @Nullable String beanName, Class<?> requiredType, DependencyDescriptor descriptor) {

    //先把符合类型的Bean都查询出来
   String[] candidateNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
         this, requiredType, true, descriptor.isEager());
   Map<String, Object> result = CollectionUtils.newLinkedHashMap(candidateNames.length);
   ......
       
   for (String candidate : candidateNames) {
       //isAutowireCandidate会先根据autowireCandidate判断，而后会使用@Qualifier判断，若允许则加入result中
      if (!isSelfReference(beanName, candidate) && isAutowireCandidate(candidate, descriptor)) {
         addCandidateEntry(result, candidate, descriptor, requiredType);
      }
   }
   if (result.isEmpty()) {
       //后续保险匹配，可忽略
      boolean multiple = indicatesMultipleBeans(requiredType);
      // Consider fallback matches if the first pass failed to find anything...
      DependencyDescriptor fallbackDescriptor = descriptor.forFallbackMatch();
      for (String candidate : candidateNames) {
         if (!isSelfReference(beanName, candidate) && isAutowireCandidate(candidate, fallbackDescriptor) &&
               (!multiple || getAutowireCandidateResolver().hasQualifier(descriptor))) {
            addCandidateEntry(result, candidate, descriptor, requiredType);
         }
      }
      if (result.isEmpty() && !multiple) {
         // Consider self references as a final pass...
         // but in the case of a dependency collection, not the very same bean itself.
         for (String candidate : candidateNames) {
            if (isSelfReference(beanName, candidate) &&
                  (!(descriptor instanceof MultiElementDescriptor) || !beanName.equals(candidate)) &&
                  isAutowireCandidate(candidate, fallbackDescriptor)) {
               addCandidateEntry(result, candidate, descriptor, requiredType);
            }
         }
      }
   }
   return result;
}
```



##### AutowireCandidateResolver

默认根据`BeanDefinition#autowireCandidate`判断，子类`QualifierAnnotationAutowireCandidateResolver`提供`@Qualifier`处理能力，默认`ContextAnnotationAutowireCandidateResolver`继承`QualifierAnnotationAutowireCandidateResolver`。

```java
public interface AutowireCandidateResolver {

   /**
    * Determine whether the given bean definition qualifies as an
    * autowire candidate for the given dependency.
    * <p>The default implementation checks
    * {@link org.springframework.beans.factory.config.BeanDefinition#isAutowireCandidate()}.
    * @param bdHolder the bean definition including bean name and aliases
    * @param descriptor the descriptor for the target method parameter or field
    * @return whether the bean definition qualifies as autowire candidate
    * @see org.springframework.beans.factory.config.BeanDefinition#isAutowireCandidate()
    */
   default boolean isAutowireCandidate(BeanDefinitionHolder bdHolder, DependencyDescriptor descriptor) {
      return bdHolder.getBeanDefinition().isAutowireCandidate();
   }
}
```



#### determineAutowireCandidate

包含了`Primary`、`@Priority`、名称匹配三种策略

```java
/**
 * Determine the autowire candidate in the given set of beans.
 * <p>Looks for {@code @Primary} and {@code @Priority} (in that order).
 * @param candidates a Map of candidate names and candidate instances
 * that match the required type, as returned by {@link #findAutowireCandidates}
 * @param descriptor the target dependency to match against
 * @return the name of the autowire candidate, or {@code null} if none found
 */
@Nullable
protected String determineAutowireCandidate(Map<String, Object> candidates, DependencyDescriptor descriptor) {
   Class<?> requiredType = descriptor.getDependencyType();
    //是否指定了Primary
   String primaryCandidate = determinePrimaryCandidate(candidates, requiredType);
   if (primaryCandidate != null) {
      return primaryCandidate;
   }
    //是否有使用@Priority指定优先级
   String priorityCandidate = determineHighestPriorityCandidate(candidates, requiredType);
   if (priorityCandidate != null) {
      return priorityCandidate;
   }
   // Fallback
   for (Map.Entry<String, Object> entry : candidates.entrySet()) {
      String candidateName = entry.getKey();
      Object beanInstance = entry.getValue();
       //最后兜底，使用BeanName和注入属性的名称进行匹配
      if ((beanInstance != null && this.resolvableDependencies.containsValue(beanInstance)) ||
            matchesBeanName(candidateName, descriptor.getDependencyName())) {
         return candidateName;
      }
   }
   return null;
}
```



## 依赖查找分析

依赖查找有两种，一种是指定Bean的名称查找，不存在不唯一的情况，另一种便是根据类型去查找，可能会出现`NoUniqueBeanDefinitionException`。

错误点：`resolveNamedBean`抛出异常

分析入口：`DefaultListableBeanFactory#resolveNamedBean`

### resolveNamedBean

根据类型`getBean`，最终会调用`resolveNamedBean`获取`NamedBeanHolder`以获得对应的`Bean`，当存在多个Bean时，此方法依次根据`BeanDefinition#autowireCandidate`、`Primary`以及`Priority`进行处理，在无法决定的情况下会报错。

```java
private <T> NamedBeanHolder<T> resolveNamedBean(
      ResolvableType requiredType, @Nullable Object[] args, boolean nonUniqueAsNull) throws BeansException {

   Assert.notNull(requiredType, "Required type must not be null");
   String[] candidateNames = getBeanNamesForType(requiredType);

   if (candidateNames.length > 1) {
      List<String> autowireCandidates = new ArrayList<>(candidateNames.length);
      for (String beanName : candidateNames) {
          //判断BeanDefinition#autowireCandidate
         if (!containsBeanDefinition(beanName) || getBeanDefinition(beanName).isAutowireCandidate()) {
            autowireCandidates.add(beanName);
         }
      }
      if (!autowireCandidates.isEmpty()) {
         candidateNames = StringUtils.toStringArray(autowireCandidates);
      }
   }

   if (candidateNames.length == 1) {
      return resolveNamedBean(candidateNames[0], requiredType, args);
   }
   else if (candidateNames.length > 1) {
      Map<String, Object> candidates = CollectionUtils.newLinkedHashMap(candidateNames.length);
      for (String beanName : candidateNames) {
         if (containsSingleton(beanName) && args == null) {
            Object beanInstance = getBean(beanName);
            candidates.put(beanName, (beanInstance instanceof NullBean ? null : beanInstance));
         }
         else {
            candidates.put(beanName, getType(beanName));
         }
      }
       //判断Primary
      String candidateName = determinePrimaryCandidate(candidates, requiredType.toClass());
      if (candidateName == null) {
          //根据@Priority筛选优先级最高的
         candidateName = determineHighestPriorityCandidate(candidates, requiredType.toClass());
      }
      if (candidateName != null) {
         Object beanInstance = candidates.get(candidateName);
         if (beanInstance == null) {
            return null;
         }
         if (beanInstance instanceof Class) {
            return resolveNamedBean(candidateName, requiredType, args);
         }
         return new NamedBeanHolder<>(candidateName, (T) beanInstance);
      }
      if (!nonUniqueAsNull) {
         throw new NoUniqueBeanDefinitionException(requiredType, candidates.keySet());
      }
   }

   return null;
}
```



## 总结

对于`NoUniqueBeanDefinitionException`，我们有多种处理方式，按依赖注入&依赖查找来讲

- 依赖注入（按顺序）
  1. `BeanDefinition#autowireCandidate`属性设置，只保留一个为true的
  2. `@Qualifier`注解进行处理
  3. 指定`Primary`的`Bean`
  4. `@Priority`注解指定顺序
  5. 兜底名称匹配
- 依赖查找-按类型（按顺序）
  1. `BeanDefinition#autowireCandidate`属性设置，只保留一个为true的
  2. 指定`Primary`的`Bean`
  3. `@Priority`注解指定顺序