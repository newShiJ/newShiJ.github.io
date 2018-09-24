---
layout: post
title:  "Spring MVC 控制层参数获取"
date:   2018-08-20 21:29:04 +0800
categories: jekyll update
---
### Spring MVC 如何获取参数值

> 如果使用的反射的模式是不能直接拿到参数名称的，在 java 1.8 以后可以开启保留参数 下面是maven的打包命令 保留参数

```xml
<build>
	<finalName>lakers</finalName>
	<plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-resources-plugin</artifactId>
        </plugin>

		<plugin>
			<groupId>org.apache.maven.plugins</groupId>
			<artifactId>maven-compiler-plugin</artifactId>
			<configuration>
				<source>1.8</source>
				<target>1.8</target>
				<encoding>UTF-8</encoding>
				<compilerArgs>
					<arg>-parameters</arg>
				</compilerArgs>
			</configuration>
		</plugin>
    </plugins>
</build>

```

> 但是Spring 并不是这样操作的，起码在Java 1.8 之前就有Spring MVC 了。接下来就让我们一步步 Debug 代码看看Spring MVC 是如何做到的。


> 因为已经写了一个Debug所以以下内容会比较水一些，主要是记录了之前Debug代码过程中的一些关系步骤

### InvocableHandlerMethod  # invokeForRequest
`InvocableHandlerMethod` 中 `invokeForRequest`方法  

```java
public Object invokeForRequest
    (NativeWebRequest request, ModelAndViewContainer mavContainer,
		Object... providedArgs) throws Exception {
    
    // 关注点获取参数
	Object[] args = getMethodArgumentValues(request, mavContainer, providedArgs);
	if (logger.isTraceEnabled()) {
		logger.trace("Invoking '" + ClassUtils.getQualifiedMethodName(getMethod(), getBeanType()) +
				"' with arguments " + Arrays.toString(args));
	}
	Object returnValue = doInvoke(args);
	if (logger.isTraceEnabled()) {
		logger.trace("Method [" + ClassUtils.getQualifiedMethodName(getMethod(), getBeanType()) +
				"] returned [" + returnValue + "]");
	}
	return returnValue;
}
```

### InvocableHandlerMethod  # getMethodArgumentValues
`InvocableHandlerMethod` 中 `getMethodArgumentValues`

```java
private Object[] getMethodArgumentValues(NativeWebRequest request,ModelAndViewContainer mavContainer,
		Object... providedArgs) throws Exception {
	MethodParameter[] parameters = getMethodParameters();
	Object[] args = new Object[parameters.length];
	for (int i = 0; i < parameters.length; i++) {
		MethodParameter parameter = parameters[i];
		parameter.initParameterNameDiscovery(this.parameterNameDiscoverer);
		args[i] = resolveProvidedArgument(parameter, providedArgs);
		if (args[i] != null) {
			continue;
		}
		if (this.argumentResolvers.supportsParameter(parameter)) {
			try {
			    // 第一次开始拿不到 就进入操作了
				args[i] = this.argumentResolvers.resolveArgument(
						parameter, mavContainer, request, this.dataBinderFactory);
				continue;
			}
			catch (Exception ex) {
				if (logger.isDebugEnabled()) {
					logger.debug(getArgumentResolutionErrorMessage("Failed to resolve", i), ex);
				}
				throw ex;
			}
		}
		if (args[i] == null) {
			throw new IllegalStateException("Could not resolve method parameter at index " +
					parameter.getParameterIndex() + " in " + parameter.getMethod().toGenericString() +
					": " + getArgumentResolutionErrorMessage("No suitable resolver for", i));
		}
	}
	return args;
}

```

### MethodParameter

```java
public String getParameterName() {
	ParameterNameDiscoverer discoverer = this.parameterNameDiscoverer;
	if (discoverer != null) {
		String[] parameterNames = (this.method != null ?
				discoverer.getParameterNames(this.method) : discoverer.getParameterNames(this.constructor));
		if (parameterNames != null) {
			this.parameterName = parameterNames[this.parameterIndex];
		}
		this.parameterNameDiscoverer = null;
	}
	return this.parameterName;
}
```

```java
@Override
public String[] getParameterNames(Method method) {
	for (ParameterNameDiscoverer pnd : this.parameterNameDiscoverers) {
	    //这个是比较核心的地方
		String[] result = pnd.getParameterNames(method);
		if (result != null) {
			return result;
		}
	}
	return null;
}
```
> pnd.getParameterNames(method);
### StandardReflectionParameterNameDiscoverer

`getParameterNames`


```java

/**
 * {@link ParameterNameDiscoverer} implementation which uses JDK 8's reflection facilities
 * for introspecting parameter names (based on the "-parameters" compiler flag).
 *
 * @author Juergen Hoeller
 * @since 4.0   >>  4.0之后才有的东西 
 * @see java.lang.reflect.Method#getParameters()
 * @see java.lang.reflect.Parameter#getName()
 */
@UsesJava8 //java 8 才有的
public class StandardReflectionParameterNameDiscoverer 
        implements ParameterNameDiscoverer {

	@Override
	public String[] getParameterNames(Method method) {
		Parameter[] parameters = method.getParameters();
		String[] parameterNames = new String[parameters.length];
		for (int i = 0; i < parameters.length; i++) {
			Parameter param = parameters[i];
			//应该是 Java 8 才提供的API 
			if (!param.isNamePresent()) {
				return null;
			}
			parameterNames[i] = param.getName();
		}
		return parameterNames;
	}

	@Override
	public String[] getParameterNames(Constructor<?> ctor) {
		Parameter[] parameters = ctor.getParameters();
		String[] parameterNames = new String[parameters.length];
		for (int i = 0; i < parameters.length; i++) {
			Parameter param = parameters[i];
			if (!param.isNamePresent()) {
				return null;
			}
			parameterNames[i] = param.getName();
		}
		return parameterNames;
	}

}
```
> 上面提到了Java 8 的 `Parameter` 的API 那么我们就看看好了

#### Parameter
```java
/**
 * Information about method parameters.
 *
 * A {@code Parameter} provides information about method parameters,
 * including its name and modifiers.  It also provides an alternate
 * means of obtaining attributes for the parameter.
 *
 *  >> 话不多说 1.8 才有的
 * @since 1.8
 */
public final class Parameter implements AnnotatedElement {
    
    ...
    
    /**
     * Returns true if the parameter has a name according to the class
     * file; returns false otherwise. Whether a parameter has a name
     * is determined by the {@literal MethodParameters} attribute of
     * the method which declares the parameter.
     *
     * @return true if and only if the parameter has a name according
     * to the class file.
     */
     
     /**
      *如果参数根据类具有名称，则返回true
      *档案; 否则返回false。 参数是否具有名称
      *由{@literal MethodParameters}属性决定
      *声明参数的方法。
      *
      * @return当且仅当参数具有名称时才返回true
      */ 
    public boolean isNamePresent() {
        return executable.hasRealParameterData() && name != null;
    }
    
    ...
    
    /**
     * Returns the name of the parameter.  If the parameter's name is
     * {@linkplain #isNamePresent() present}, then this method returns
     * the name provided by the class file. Otherwise, this method
     * synthesizes a name of the form argN, where N is the index of
     * the parameter in the descriptor of the method which declares
     * the parameter.
     *
     * @return The name of the parameter, either provided by the class
     *         file or synthesized if the class file does not provide
     *         a name.
     */
     
     **
     *返回参数的名称。 如果参数的名称是
     * {@linkplain #isNamePresent（）present}，然后此方法返回
     *类文件提供的名称。 否则，这个方法
     *合成argN形式的名称，其中N是索引
     *声明方法的描述符中的参数
     *参数。
     *
     * @return参数的名称，由类提供 
     *文件或合成，如果类文件不提供
     * 一个名字。
     */
     
     // 直接就证实了之前 getName 总是拿到 arg0 arg1... 之类的东西
     
    public String getName() {
        // Note: empty strings as paramete names are now outlawed.
        // The .equals("") is for compatibility with current JVM
        // behavior.  It may be removed at some point.
        if(name == null || name.equals(""))
            return "arg" + index;
        else
            return name;
    }
    ...
}

```

```java
@Override
public String[] getParameterNames(Method method) {
	for (ParameterNameDiscoverer pnd : this.parameterNameDiscoverers) {
	    //这个是比较核心的地方
		String[] result = pnd.getParameterNames(method);
		if (result != null) {
			return result;
		}
	}
	return null;
}
```
>`this.parameterNameDiscoverers` 是有两个的 这个应该和Spring的版本有关

> 第一个是 `StandardReflectionParameterNameDiscoverer` 应该走的是Java 1.8以后自带的反射

> 第二个才是核心 `LocalVariableTableParameterNameDiscoverer` spring 2.0 就有的东西 走的是 xxx.class 文件 （极其猥琐）

```java 
// 上文中的 pnd.getParameterNames(method);

@Override
public String[] getParameterNames(Method method) {
	Method originalMethod = BridgeMethodResolver.findBridgedMethod(method);
	Class<?> declaringClass = originalMethod.getDeclaringClass();
	Map<Member, String[]> map = this.parameterNamesCache.get(declaringClass);
	if (map == null) {
	    // 本次源码阅读的终点了 
		map = inspectClass(declaringClass);
		this.parameterNamesCache.put(declaringClass, map);
	}
	if (map != NO_DEBUG_INFO_MAP) {
		return map.get(originalMethod);
	}
	return null;
}
```

```java
private Map<Member, String[]> inspectClass(Class<?> clazz) {
	// ClassUtils.getClassFileName(clazz) 就是直接获取类的文件名 xxx.class
	// 通过Class 以流的形式获取数据 （输入流）
	InputStream is = clazz.getResourceAsStream(ClassUtils.getClassFileName(clazz));
	//输入流为空 出现问题咯
	if (is == null) {
		// We couldn't load the class file, which is not fatal as it
		// simply means this method of discovering parameter names won't work.
		if (logger.isDebugEnabled()) {
			logger.debug("Cannot find '.class' file for class [" + clazz +
					"] - unable to determine constructor/method parameter names");
		}
		return NO_DEBUG_INFO_MAP;
	}
	try {
	    // 新建一个输入流的解析器
		ClassReader classReader = new ClassReader(is);
		Map<Member, String[]> map = new ConcurrentHashMap<Member, String[]>(32);
		// 解析输入流 把数据放进 map 中
		classReader.accept(new ParameterNameDiscoveringVisitor(clazz, map), 0);
		return map;
	}
	catch (IOException ex) {
		if (logger.isDebugEnabled()) {
			logger.debug("Exception thrown while reading '.class' file for class [" + clazz +
					"] - unable to determine constructor/method parameter names", ex);
		}
	}
	catch (IllegalArgumentException ex) {
		if (logger.isDebugEnabled()) {
			logger.debug("ASM ClassReader failed to parse class file [" + clazz +
					"], probably due to a new Java class file version that isn't supported yet " +
					"- unable to determine constructor/method parameter names", ex);
		}
	}
	finally {
		try {
		    //最后关闭输入流
			is.close();
		}
		catch (IOException ex) {
			// 还能直接 ignore？！！  能不能严谨一点 嘲讽一下Spring的工程师
			// ignore
		}
	}
	return NO_DEBUG_INFO_MAP;
}
```

### 总结一波

> 其实在没有Debug阅读源码之前我也是对 Spring 感到很神秘的，对Spring MVC 自动装配参数识别感到惊讶，毕竟之前自己写代码的时候反射获取参数总是 arg0 ，arg1... 今天有幸阅读了代码，发现其实也没有那么神秘，和之前的猜想差不多，既然JDK 的API都不能很好的解决问题，那么很有可能是使用了一些字节码技术。到最后也如愿以偿的看到了Spring 的最终解决方案。最后的时候还顺带发现了一个Spring团队写代码的 ignore 注释，理论上是不是可以使用 Java 7 的 try with 语法。

> 再多废话几句，理论上来说我们可以自己实现一个很简陋的类似于 Spring MVC 的MVC框架体系，有朝一日我觉得我也可以试着去实现一个我的版本的 MVC 虽然不会去使用但是对编码的理解会加深一些。 

