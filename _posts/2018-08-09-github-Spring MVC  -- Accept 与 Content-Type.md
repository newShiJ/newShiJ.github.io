---
layout: post
title:  "Spring MVC  -- Accept 与 Content-Type"
date:   2018-08-09 21:29:04 +0800
categories: [http, spring mvc]
tags:	    [http, spring mvc]
keywords: spring mvc
---

### Rest 请求 

请求方式 | 安全 | 幂等 | 接口说明 
:---:|:---:|:---:|:---:
GET | 安全 | 幂等 | 获取资源
PSOT | 不安全 | 非幂等 | 创建资源
PUT | 不安全 | 幂等| 更新资源
DELETE | 不安全 | 幂等 | 删除资源

幂等/非幂等 依赖于服务端实现，这种方式是一种契约

### HTTP
####  Accept 与 Content-Type  

> Accept代表发送端（客户端）希望接受的数据类型。

```
比如：Accept：text/xml（application/json）; 
代表客户端希望接受的数据类型是xml（json ）类型 
Content-Type代表发送端（客户端|服务器）发送的实体数据的数据类型。

比如：Content-Type：text/html（application/json） ; 
代表发送端发送的数据格式是html（json）。 
二者合起来，

Accept:text/xml； 
Content-Type:text/html

即代表希望接受的数据类型是xml格式，本次请求发送的数据的数据格式是html。
```
```
Accept  application/xml,application/json 
表示的是希望接收数据格式的顺序 最好先是application/xml 不行就走 application/json 
```
### Spring MVC 的实现


```java
package org.springframework.http.converter;


//这个接口为spring MVC 的转换器
public interface  HttpMessageConverter<T>  {

    //判断一个MediaType 是否可读
    boolean canRead(Class<?> clazz, @Nullable MediaType mediaType);
    
    //判断一个MediaType 是否可写
    boolean canWrite(Class<?> clazz, @Nullable MediaType mediaType);
    
    //可以支持哪些 MediaType
    List<MediaType> getSupportedMediaTypes();

    //读取参数 @RequestBody 时使用的 反序列化出一个对象 
    T read(Class<? extends T> clazz, HttpInputMessage inputMessage)
			throws IOException, HttpMessageNotReadableException;

    //将一个对象写入到流中  @ResponseBody 时使用 序列化一个对象
	void write(T t, @Nullable MediaType contentType, HttpOutputMessage outputMessage)
		throws IOException, HttpMessageNotWritableException;

}

这里的 MediaType 就是http请求头中的 Accept/Content-Type 属性

``` 

#### Spring MVC 的基本配置信息
```java

//这个类记录了一些Spring MVC 的基本配置信息
package org.springframework.web.servlet.config.annotation;

public class WebMvcConfigurationSupport implements 
            ApplicationContextAware, ServletContextAware {
    
    //这些就是对应不同的 HTTP 请求头中 Accept/Content-Type 属性转换 Spring MVC 世界中为 MediaType 
    
    /***
     *  默认的spring boot 2.0 是没有 application/xml 的转换器但是可以根据
     *  以下这些默认依赖信息去添加依赖
     */
    
    private static boolean romePresent =
			ClassUtils.isPresent("com.rometools.rome.feed.WireFeed",
					WebMvcConfigurationSupport.class.getClassLoader());

	private static final boolean jaxb2Present =
			ClassUtils.isPresent("javax.xml.bind.Binder",
					WebMvcConfigurationSupport.class.getClassLoader());

	private static final boolean jackson2Present =
			ClassUtils.isPresent("com.fasterxml.jackson.databind.ObjectMapper",
					WebMvcConfigurationSupport.class.getClassLoader()) &&
			ClassUtils.isPresent("com.fasterxml.jackson.core.JsonGenerator",
					WebMvcConfigurationSupport.class.getClassLoader());

	private static final boolean jackson2XmlPresent =
			ClassUtils.isPresent("com.fasterxml.jackson.dataformat.xml.XmlMapper",
					WebMvcConfigurationSupport.class.getClassLoader());

	private static final boolean jackson2SmilePresent =
			ClassUtils.isPresent("com.fasterxml.jackson.dataformat.smile.SmileFactory",
					WebMvcConfigurationSupport.class.getClassLoader());

	private static final boolean jackson2CborPresent =
			ClassUtils.isPresent("com.fasterxml.jackson.dataformat.cbor.CBORFactory",
					WebMvcConfigurationSupport.class.getClassLoader());

	private static final boolean gsonPresent =
			ClassUtils.isPresent("com.google.gson.Gson",
					WebMvcConfigurationSupport.class.getClassLoader());

	private static final boolean jsonbPresent =
			ClassUtils.isPresent("javax.json.bind.Jsonb",
					WebMvcConfigurationSupport.class.getClassLoader());

    ...
    
    // 下面的这个方法就是根据上面一段代码中解析器的依赖添加可以执行处理的MediaType
    //  查询一下哪些 HTTP 中的 Accept/Content-Type  的是可以被处理的添加到Map当中
    
    protected Map<String, MediaType> getDefaultMediaTypes() {
		Map<String, MediaType> map = new HashMap<>(4);
		if (romePresent) {
			map.put("atom", MediaType.APPLICATION_ATOM_XML);
			map.put("rss", MediaType.APPLICATION_RSS_XML);
		}
		if (jaxb2Present || jackson2XmlPresent) {
			map.put("xml", MediaType.APPLICATION_XML);
		}
		if (jackson2Present || gsonPresent || jsonbPresent) {
			map.put("json", MediaType.APPLICATION_JSON);
		}
		if (jackson2SmilePresent) {
			map.put("smile", MediaType.valueOf("application/x-jackson-smile"));
		}
		if (jackson2CborPresent) {
			map.put("cbor", MediaType.valueOf("application/cbor"));
		}
		return map;
	}
    ...
    
    //为了证实上面的陈述，找到了以下的这个方法，看看MVC对请求的映射处理
    // @Bean 可以看出来这个也是单例的
    @Bean
	public RequestMappingHandlerAdapter requestMappingHandlerAdapter() {
		RequestMappingHandlerAdapter adapter = createRequestMappingHandlerAdapter();
		adapter.setContentNegotiationManager(mvcContentNegotiationManager());
		adapter.setMessageConverters(getMessageConverters());
		adapter.setWebBindingInitializer(getConfigurableWebBindingInitializer());
		adapter.setCustomArgumentResolvers(getArgumentResolvers());
		adapter.setCustomReturnValueHandlers(getReturnValueHandlers());


        //这里是重点观察对象 
        //存在jackson2Present （Jackson2的json映射处理器）往往处理器中添加处理实例
		if (jackson2Present) {
			adapter.setRequestBodyAdvice(Collections.singletonList(new JsonViewRequestBodyAdvice()));
			adapter.setResponseBodyAdvice(Collections.singletonList(new JsonViewResponseBodyAdvice()));
		}

		AsyncSupportConfigurer configurer = new AsyncSupportConfigurer();
		configureAsyncSupport(configurer);
		if (configurer.getTaskExecutor() != null) {
			adapter.setTaskExecutor(configurer.getTaskExecutor());
		}
		if (configurer.getTimeout() != null) {
			adapter.setAsyncRequestTimeout(configurer.getTimeout());
		}
		adapter.setCallableInterceptors(configurer.getCallableInterceptors());
		adapter.setDeferredResultInterceptors(configurer.getDeferredResultInterceptors());

		return adapter;
	}
	...
	
    //添加默认的 HttpMessage 处理器集合
	protected final void addDefaultHttpMessageConverters(List<HttpMessageConverter<?>> messageConverters) {
		StringHttpMessageConverter stringHttpMessageConverter = new StringHttpMessageConverter();
		stringHttpMessageConverter.setWriteAcceptCharset(false);  // see SPR-7316

		messageConverters.add(new ByteArrayHttpMessageConverter());
		messageConverters.add(stringHttpMessageConverter);
		messageConverters.add(new ResourceHttpMessageConverter());
		messageConverters.add(new ResourceRegionHttpMessageConverter());
		messageConverters.add(new SourceHttpMessageConverter<>());
		messageConverters.add(new AllEncompassingFormHttpMessageConverter());

        //根据最开始的代码查询是否含有相关依赖王处理集合中添加处理器

		if (romePresent) {
			messageConverters.add(new AtomFeedHttpMessageConverter());
			messageConverters.add(new RssChannelHttpMessageConverter());
		}

		if (jackson2XmlPresent) {
			Jackson2ObjectMapperBuilder builder = Jackson2ObjectMapperBuilder.xml();
			if (this.applicationContext != null) {
				builder.applicationContext(this.applicationContext);
			}
			messageConverters.add(new MappingJackson2XmlHttpMessageConverter(builder.build()));
		}
		else if (jaxb2Present) {
			messageConverters.add(new Jaxb2RootElementHttpMessageConverter());
		}

		if (jackson2Present) {
			Jackson2ObjectMapperBuilder builder = Jackson2ObjectMapperBuilder.json();
			if (this.applicationContext != null) {
				builder.applicationContext(this.applicationContext);
			}
			messageConverters.add(new MappingJackson2HttpMessageConverter(builder.build()));
		}
		else if (gsonPresent) {
			messageConverters.add(new GsonHttpMessageConverter());
		}
		else if (jsonbPresent) {
			messageConverters.add(new JsonbHttpMessageConverter());
		}

		if (jackson2SmilePresent) {
			Jackson2ObjectMapperBuilder builder = Jackson2ObjectMapperBuilder.smile();
			if (this.applicationContext != null) {
				builder.applicationContext(this.applicationContext);
			}
			messageConverters.add(new MappingJackson2SmileHttpMessageConverter(builder.build()));
		}
		if (jackson2CborPresent) {
			Jackson2ObjectMapperBuilder builder = Jackson2ObjectMapperBuilder.cbor();
			if (this.applicationContext != null) {
				builder.applicationContext(this.applicationContext);
			}
			messageConverters.add(new MappingJackson2CborHttpMessageConverter(builder.build()));
		}
	}
	...
}
```

#### Spring MVC 匹配 MediaType
```java
package org.springframework.web.accept;

public class ContentNegotiationManager implements 
        ContentNegotiationStrategy, MediaTypeFileExtensionResolver {

    ...
    
    /**
	 * Resolve the given request to a list of media types. The returned list is
	 * ordered by specificity first and by quality parameter second.
	 * @param webRequest the current request
	 * @return the requested media types, or {@link #MEDIA_TYPE_ALL_LIST} if none
	 * were requested.
	 * @throws HttpMediaTypeNotAcceptableException if the requested media
	 * types cannot be parsed
	 */
	 
    /**
    *将给定请求解析为媒体类型列表。 返回的列表是
    *首先按特异性排序，然后按质量参数排序。
    * @param webRequest当前请求
    * @return请求的媒体类型，或{@link #MEDIA_TYPE_ALL_LIST}（如果没有）
    *被要求。
    * @throws HttpMediaTypeNotAcceptableException如果请求的媒体
    *类型无法解析
    */
    
    @Override
	public List<MediaType> resolveMediaTypes(NativeWebRequest request) throws HttpMediaTypeNotAcceptableException {
		for (ContentNegotiationStrategy strategy : this.strategies) {
			List<MediaType> mediaTypes = strategy.resolveMediaTypes(request);
			if (mediaTypes.equals(MEDIA_TYPE_ALL_LIST)) {
				continue;
			}
			return mediaTypes;
		}
		return MEDIA_TYPE_ALL_LIST;
	}
	
	/**
	* 可以看出是经过for循环去匹配所有可以处理的 MediaType
	* 因此默认的顺序为list中的顺序即上面一个类中 addDefaultHttpMessageConverters 方法的加入顺序
	*/
    ...
       
}
```

#### 除此之外可以看看Spring MVC 对 Request 和 Response 的包装

```java
package org.springframework.http.server;

public class ServletServerHttpRequest implements ServerHttpRequest {
    
    protected static final String FORM_CONTENT_TYPE = "application/x-www-form-urlencoded";

	protected static final Charset FORM_CHARSET = StandardCharsets.UTF_8;


	private final HttpServletRequest servletRequest;
    
    //没错，spring MVC 包装的 Request 其实在内部包装了一个 HttpServletRequest

    ...
    
    
    /**
	 * Construct a new instance of the ServletServerHttpRequest based on the
	 * given {@link HttpServletRequest}.
	 * @param servletRequest the servlet request
	 */
	public ServletServerHttpRequest(HttpServletRequest servletRequest) {
		Assert.notNull(servletRequest, "HttpServletRequest must not be null");
		this.servletRequest = servletRequest;
	}
	//就连构造函数都要一个 HttpServletRequest 
	...
    
}

// 同理Response 应该也是如此

package org.springframework.http.server;

public class ServletServerHttpResponse implements ServerHttpResponse {

    private final HttpServletResponse servletResponse;

	private final HttpHeaders headers;

	private boolean headersWritten = false;

	private boolean bodyUsed = false;


	/**
	 * Construct a new instance of the ServletServerHttpResponse based on the given {@link HttpServletResponse}.
	 * @param servletResponse the servlet response
	 */
	public ServletServerHttpResponse(HttpServletResponse servletResponse) {
		Assert.notNull(servletResponse, "HttpServletResponse must not be null");
		this.servletResponse = servletResponse;
		this.headers = new ServletResponseHttpHeaders();
	}
	
	// 看出来了吧就连构造函数都需要一个 HttpServletResponse 可以认为基本是等价的
	
	...
}

// 综上所述可以看出来 Spring 就是喜欢装*  
```

#### Spring MVC 对于 @RequestBody 参数入参反序列化的处理
```java
package org.springframework.web.servlet.mvc.method.annotation;

public abstract class AbstractMessageConverterMethodArgumentResolver 
        implements HandlerMethodArgumentResolver {

    ...
    
    @SuppressWarnings("unchecked")
	@Nullable
	protected <T> Object readWithMessageConverters(HttpInputMessage inputMessage, MethodParameter parameter,
			Type targetType) throws IOException, HttpMediaTypeNotSupportedException, HttpMessageNotReadableException {

		MediaType contentType;
		boolean noContentType = false;
		try {
			contentType = inputMessage.getHeaders().getContentType();
		}
		catch (InvalidMediaTypeException ex) {
			throw new HttpMediaTypeNotSupportedException(ex.getMessage());
		}
		if (contentType == null) {
			noContentType = true;
			contentType = MediaType.APPLICATION_OCTET_STREAM;
		}

		Class<?> contextClass = parameter.getContainingClass();
		Class<T> targetClass = (targetType instanceof Class ? (Class<T>) targetType : null);
		if (targetClass == null) {
			ResolvableType resolvableType = ResolvableType.forMethodParameter(parameter);
			targetClass = (Class<T>) resolvableType.resolve();
		}

		HttpMethod httpMethod = (inputMessage instanceof HttpRequest ? ((HttpRequest) inputMessage).getMethod() : null);
		Object body = NO_VALUE;

		EmptyBodyCheckingHttpInputMessage message;
		
		
		//重点观察
		try {
			message = new EmptyBodyCheckingHttpInputMessage(inputMessage);

            //进入循环，根据已经初始化的 HttpMessageConverter 集合
			for (HttpMessageConverter<?> converter : this.messageConverters) {
				Class<HttpMessageConverter<?>> converterType = (Class<HttpMessageConverter<?>>) converter.getClass();
				GenericHttpMessageConverter<?> genericConverter =
						(converter instanceof GenericHttpMessageConverter ? (GenericHttpMessageConverter<?>) converter : null);
				if (genericConverter != null ? 
				// 眼睛请看过来
				/**
				* 如果当前处理器（HttpMessageConverter）可以读取（反序列化)
				* 那么就进入了 处理器的处理流程 canRead -> read 
				*
				*/
				genericConverter.canRead(targetType, contextClass, contentType) :
						(targetClass != null && converter.canRead(targetClass, contentType))) {
					if (logger.isDebugEnabled()) {
						logger.debug("Read [" + targetType + "] as \"" + contentType + "\" with [" + converter + "]");
					}
					/**
					* 存在 待反序列化的消息 那么就开始了Spring MVC 的表演
					*/
					if (message.hasBody()) {
						HttpInputMessage msgToUse =
								getAdvice().beforeBodyRead(message, parameter, targetType, converterType);
						// 前置增强处理器处理
						body = (genericConverter != null ? genericConverter.read(targetType, contextClass, msgToUse) :
								((HttpMessageConverter<T>) converter).read(targetClass, msgToUse));
						// 后置增强处理器处理
						body = getAdvice().afterBodyRead(body, msgToUse, parameter, targetType, converterType);
					}
					// 如何http请求中莫得数据就返回给了一个空值的处理
					else {
						body = getAdvice().handleEmptyBody(null, message, parameter, targetType, converterType);
					}
					/***
					* 只要找到了一个可以处理的处理器处理完成之后就再见了
					* 避免了反复处理的问题
					*/
					break;
				}
			}
		}
		catch (IOException ex) {
			throw new HttpMessageNotReadableException("I/O error while reading input message", ex);
		}
        
        // <=======

		if (body == NO_VALUE) {
			if (httpMethod == null || !SUPPORTED_METHODS.contains(httpMethod) ||
					(noContentType && !message.hasBody())) {
				return null;
			}
			throw new HttpMediaTypeNotSupportedException(contentType, this.allSupportedMediaTypes);
		}

		return body;
	}

    ...
}
```

#### Spring MVC 对于 @ResponseBody 的处理方式
```java
package org.springframework.web.servlet.mvc.method.annotation;

public abstract class AbstractMessageConverterMethodArgumentResolver 
        implements HandlerMethodArgumentResolver {
        
    ...
    
    @SuppressWarnings({"rawtypes", "unchecked"})
	protected <T> void writeWithMessageConverters(@Nullable T value, MethodParameter returnType,
			ServletServerHttpRequest inputMessage, ServletServerHttpResponse outputMessage)
			throws IOException, HttpMediaTypeNotAcceptableException, HttpMessageNotWritableException {

		Object outputValue;
		Class<?> valueType;
		Type declaredType;

		if (value instanceof CharSequence) {
			outputValue = value.toString();
			valueType = String.class;
			declaredType = String.class;
		}
		else {
			outputValue = value;
			valueType = getReturnValueType(outputValue, returnType);
			declaredType = getGenericType(returnType);
		}
		
		if (isResourceType(value, returnType)) {
			outputMessage.getHeaders().set(HttpHeaders.ACCEPT_RANGES, "bytes");
			if (value != null && inputMessage.getHeaders().getFirst(HttpHeaders.RANGE) != null) {
				Resource resource = (Resource) value;
				try {
					List<HttpRange> httpRanges = inputMessage.getHeaders().getRange();
					outputMessage.getServletResponse().setStatus(HttpStatus.PARTIAL_CONTENT.value());
					outputValue = HttpRange.toResourceRegions(httpRanges, resource);
					valueType = outputValue.getClass();
					declaredType = RESOURCE_REGION_LIST_TYPE;
				}
				catch (IllegalArgumentException ex) {
					outputMessage.getHeaders().set(HttpHeaders.CONTENT_RANGE, "bytes */" + resource.contentLength());
					outputMessage.getServletResponse().setStatus(HttpStatus.REQUESTED_RANGE_NOT_SATISFIABLE.value());
				}
			}
		}


		List<MediaType> mediaTypesToUse;

		MediaType contentType = outputMessage.getHeaders().getContentType();
		if (contentType != null && contentType.isConcrete()) {
			mediaTypesToUse = Collections.singletonList(contentType);
		}
		else {
			HttpServletRequest request = inputMessage.getServletRequest();
			List<MediaType> requestedMediaTypes = getAcceptableMediaTypes(request);
			List<MediaType> producibleMediaTypes = getProducibleMediaTypes(request, valueType, declaredType);

			if (outputValue != null && producibleMediaTypes.isEmpty()) {
				throw new HttpMessageNotWritableException(
						"No converter found for return value of type: " + valueType);
			}
			mediaTypesToUse = new ArrayList<>();
			for (MediaType requestedType : requestedMediaTypes) {
				for (MediaType producibleType : producibleMediaTypes) {
					if (requestedType.isCompatibleWith(producibleType)) {
						mediaTypesToUse.add(getMostSpecificMediaType(requestedType, producibleType));
					}
				}
			}
			if (mediaTypesToUse.isEmpty()) {
				if (outputValue != null) {
					throw new HttpMediaTypeNotAcceptableException(producibleMediaTypes);
				}
				return;
			}
			MediaType.sortBySpecificityAndQuality(mediaTypesToUse);
		}

		MediaType selectedMediaType = null;
		for (MediaType mediaType : mediaTypesToUse) {
			if (mediaType.isConcrete()) {
				selectedMediaType = mediaType;
				break;
			}
			else if (mediaType.equals(MediaType.ALL) || mediaType.equals(MEDIA_TYPE_APPLICATION)) {
				selectedMediaType = MediaType.APPLICATION_OCTET_STREAM;
				break;
			}
		}
        
        // 眼睛看过来咯 <=======
		/***
		* 如果客户端发送的 Accept 不为空开始 Spring MVC 的操作咯
		*/
		if (selectedMediaType != null) {
			selectedMediaType = selectedMediaType.removeQualityValue();
			/***
			* 和上面处理请求是一个套路 for 循环遍历（它们二者是同一个类的不同方法）
			*/
			for (HttpMessageConverter<?> converter : this.messageConverters) {
				GenericHttpMessageConverter genericConverter =
						(converter instanceof GenericHttpMessageConverter ? (GenericHttpMessageConverter<?>) converter : null);
				if (genericConverter != null ?
						/**
						* 又开始咯 能不能写 能写就继续表演（写处理）
						* 不能写就下一个咯
						*/
						((GenericHttpMessageConverter) converter).canWrite(declaredType, valueType, selectedMediaType) :
						converter.canWrite(valueType, selectedMediaType)) {
					outputValue = (T) 
					//能写就开始了 这里是前置处理器
					getAdvice().beforeBodyWrite(outputValue, returnType, selectedMediaType,
							(Class<? extends HttpMessageConverter<?>>) converter.getClass(),
							inputMessage, outputMessage);
					if (outputValue != null) {
						addContentDispositionHeader(inputMessage, outputMessage);
						if (genericConverter != null) {
							genericConverter.write(outputValue, declaredType, selectedMediaType, outputMessage);
						}
						else {
							((HttpMessageConverter) converter).write(outputValue, selectedMediaType, outputMessage);
						}
						if (logger.isDebugEnabled()) {
							logger.debug("Written [" + outputValue + "] as \"" + selectedMediaType +
									"\" using [" + converter + "]");
						}
					}
					//处理完就直接 return 再见 不跟你嘻嘻哈哈
					return;
				}
			}
		}
		//  <=======
        
        // for循环一圈咯 还是不行 实在是没有能支持处理这个MediaType的处理器
		if (outputValue != null) {
			// 抛个异常  对不起臣妾做不到啊
			throw new HttpMediaTypeNotAcceptableException(this.allSupportedMediaTypes);
		}
	}
    ...
            
}
    
```

#### 往项目中添加xml的转换器

```xml
pom.xml 添加如下依赖 因为是Spring Boot所以可以不写版本号

<!-- jackson xml 的转换器 -->
<dependency>
    <groupId>com.fasterxml.jackson.dataformat</groupId>
    <artifactId>jackson-dataformat-xml</artifactId>
</dependency>

```
下图为添加 xml 转换器依赖的效果

![image](https://ws1.sinaimg.cn/large/005ZRPkkly1fu3fzbi7l0j312n0odn02.jpg)

![image](https://ws1.sinaimg.cn/large/005ZRPkkly1fu3g5jd1oqj31hc0u0agv.jpg)

 接下来开始放大招了 写一个自定义的转换器
### 自定义 MVC 转换器

> 先编写一个自定义的转换器 

```java
/**
 * @author chenmingming
 * @date 2018/8/9
 */
public class PersonPropertiesMessageConverter extends
        AbstractHttpMessageConverter<Person> {

   /***
    * 注意看这里的构造方法，描述了 MediaType 类型 以及 编码格式
    */
    public PersonPropertiesMessageConverter(){
        super(MediaType.valueOf("application/properties+person"));
        setDefaultCharset(Charset.forName("UTF-8"));
    }


    @Override
    protected boolean supports(Class clazz) {
        return clazz.isAssignableFrom(Person.class);
    }

    
    @Override
    protected Person readInternal(Class clazz, HttpInputMessage inputMessage) throws IOException, HttpMessageNotReadableException {
        Properties properties = new Properties();
        properties.load(new ReaderUTF8(inputMessage.getBody()));
        Person person = new Person();
        person.setName(properties.getProperty("person.name"));
        person.setId(Long.valueOf(properties.getProperty("person.id")));
        return person;
    }


    @Override
    protected void writeInternal(Person o, HttpOutputMessage outputMessage) throws IOException, HttpMessageNotWritableException {
        Properties properties = new Properties();
        properties.setProperty("person.name",o.getName());
        properties.setProperty("person.id",String.valueOf(o.getId()));
        OutputStream body = outputMessage.getBody();
        Charset charset = Charset.forName("UTF-8");
        properties.store(new OutputStreamWriter(outputMessage.getBody(),charset),"write By Chen");
    }
}

```

> 接下来就添加配置将我们上面那个转转换器添加到 Spring MVC 转换器列表中

```java
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {
    
    /***
     *  这里只是扩展了其中的一个方法
     *  在 WebMvcConfigurer 的方法都是 default 有了默认的实现 
     */
    @Override
    public void extendMessageConverters(
            List<HttpMessageConverter<?>> converters) {
        converters.add(new PersonPropertiesMessageConverter());
    }

}
```

效果如图所示
![image](https://ws1.sinaimg.cn/large/005ZRPkkgy1fu3o2yk6dlj312n0odjuc.jpg)
![image](https://ws1.sinaimg.cn/large/005ZRPkkgy1fu3o3qi5zpj312n0odgok.jpg)