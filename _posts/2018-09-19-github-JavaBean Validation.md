---
layout: post
title:  "Javabean Validation"
date:   2018-09-19 21:29:04 +0800
categories: jekyll update
---
### JSR 303

 > &nbsp;&nbsp;&nbsp;&nbsp;参数校验是我们程序开发中必不可少的过程。用户在前端页面上填写表单时，前端js程序会校验参数的合法性，当数据到了后端，为了防止恶意操作，保持程序的健壮性，后端同样需要对数据进行校验。后端参数校验最简单的做法是直接在业务方法里面进行判断，当判断成功之后再继续往下执行。但这样带给我们的是代码的耦合，冗余。当我们多个地方需要校验时，我们就需要在每一个地方调用校验程序,导致代码很冗余，且不美观。那么如何优雅的对参数进行校验呢？JSR303就是为了解决这个问题出现的.
 
 JSR303 是一套JavaBean参数校验的标准，它定义了很多常用的校验注解，我们可以直接将这些注解加在我们JavaBean的属性上面，就可以在需要校验的时候进行校验了。注解如下
 
> @NotNull                 注解元素必须是非空   
 @Null                      注解元素必须是空   
 @Digits                    验证数字构成是否合法   
 @Future                   验证是否在当前系统时间之后     
 @Past                       验证是否在当前系统时间之前    
 @Max                      验证值是否小于等于最大指定整数值   
 @Min                       验证值是否大于等于最小指定整数值    
 @Pattern                  验证字符串是否匹配指定的正则表达式    
 @Size                      验证元素大小是否在指定范围内   
 @DecimalMax   验证值是否小于等于最大指定小数值   
 @DecimalMin    验证值是否大于等于最小指定小数值   
 @AssertTrue             被注释的元素必须为true   
 @AssertFalse     被注释的元素必须为false   
Hibernate validator 在JSR303的基础上对校验注解进行了扩展，扩展注解如下：   
 @Email                    被注释的元素必须是电子邮箱地址   
 @Length                   被注释的字符串的大小必须在指定的范围内   
 @NotEmpty              被注释的字符串的必须非空   
 @Range                    被注释的元素必须在合适的范围内


### Spring 中使用代码示例

+ 下面一段代码就是一个基本的JavaBean对象的校验实例代码

```java
//JavaBean 对象

@Data
public class User {
    @Max(10000L)
    private long id;

    @NotNull
    private String name;

    @CardNumerValidation
    private String cardNumber;
}
```
```
//控制层代码
@RestController
public class UserController {

    @PostMapping("/user")
    public User save(@Validated @RequestBody User user){
        return user;
    }
}
```
+ 演示情况

<a href="https://ibb.co/chK3yK"><img src="https://preview.ibb.co/h6OVdK/1534597994891.jpg" alt="1534597994891" border="0"></a>

+ 错误代码

```json
{
    "timestamp": "2018-08-18T13:15:29.503+0000",
    "path": "/user",
    "status": 400,
    "error": "Bad Request",
    "message": "Validation failed for argument at index 0 in method: public cn.hyperchain.validation.User cn.hyperchain.validation.UserController.save(cn.hyperchain.validation.User), with 1 error(s): [Field error in object 'user' on field 'id': rejected value [1000000]; codes [Max.user.id,Max.id,Max.long,Max]; arguments [org.springframework.context.support.DefaultMessageSourceResolvable: codes [user.id,id]; arguments []; default message [id],10000]; default message [最大不能超过10000]] ",
    "errors": [
        {
            "codes": [
                "Max.user.id",
                "Max.id",
                "Max.long",
                "Max"
            ],
            "arguments": [
                {
                    "codes": [
                        "user.id",
                        "id"
                    ],
                    "arguments": null,
                    "defaultMessage": "id",
                    "code": "id"
                },
                10000
            ],
            "defaultMessage": "最大不能超过10000",
            "objectName": "user",
            "field": "id",
            "rejectedValue": 1000000,
            "bindingFailure": false,
            "code": "Max"
        }
    ]
}
```

### 扩展  Validation

+ 目标 ->  自定义一个校验规则处理  
   + 自定义一个注解 
   + 自定义一个注解的处理
   + 测试验证
   + 如果可以看看源码


**1.自定义注解**

+ 首先先观察已有的注解以 `@NotNull` 为例说明

```
package javax.validation.constraints;

@Target({ METHOD, FIELD, ANNOTATION_TYPE, CONSTRUCTOR, PARAMETER, TYPE_USE })
@Retention(RUNTIME)
@Repeatable(List.class)  //暂时没有使用到
@Documented
@Constraint(validatedBy = { })  //该注解的处理器类数组
public @interface NotNull {

    //该注解进行校验如果校验失败返回的提示信息
    //这里Javax 默认使用的是国际化的处理方式
	String message() default "{javax.validation.constraints.NotNull.message}";

    //分组处理 暂时不用
	Class<?>[] groups() default { };
    //不太懂 有待提高了解
	Class<? extends Payload>[] payload() default { };

	/**
	 * 这个内部类就更加不懂咯
	 * Defines several {@link NotNull} annotations on the same element.
	 *
	 * @see javax.validation.constraints.NotNull
	 */
	@Target({ METHOD, FIELD, ANNOTATION_TYPE, CONSTRUCTOR, PARAMETER, TYPE_USE })
	@Retention(RUNTIME)
	@Documented
	@interface List {

		NotNull[] value();
	}
}

```
+ 查看处理器定义


```java
package javax.validation;

public interface ConstraintValidator<A extends Annotation, T> {

    /**
     * 初始化方法
     * 不过看样子好像也没有什么必要去重写，所以定义的时候直接就给了默认实现方法
     **/
	default void initialize(A constraintAnnotation) {
	}

    /**
     * 核心的校验方法  成功 true  失败 false
     * 
     * T 定义的值类型
     * validation 上下文环境
     **/
	boolean isValid(T value, ConstraintValidatorContext context);
}

```


+ 开始搞事

```
/**
 * 自定义注解
 * @author chenmingming
 * @date 2018/8/18
 */
@Target({ FIELD})
@Retention(RUNTIME)
@Documented
@Constraint(validatedBy = {CardNumberValidator.class})
public @interface CardNumerValidation {
    String message() default "{自定义校验字段不通过}";

    Class<?>[] groups() default { };

    Class<? extends Payload>[] payload() default { };

}


/**
 * 自定义注解处理器
 * @author chenmingming
 * @date 2018/8/18
 */
public class CardNumberValidator implements ConstraintValidator<CardNumerValidation,String> {

    @Override
    public void initialize(CardNumerValidation constraintAnnotation) {

    }


    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        String[] split = StringUtils.split(value, "-");
        
        //不开心就搞事 往默认的提示信息里面加点料
        ConstraintValidatorContext.ConstraintViolationBuilder builder =
                context.buildConstraintViolationWithTemplate("没想到吧哈哈哈");

        ConstraintValidatorContext constraintValidatorContext = builder.addConstraintViolation();

        if(ArrayUtils.getLength(split) < 2){
            return false;
        }
        boolean prefix = split[0].equals("CMM");
        boolean suffix = split[split.length-1].endsWith("END");
        return prefix && suffix;
    }
}

```

+ Postman 测试结果 Response 的响应体

```
{
    "timestamp": "2018-08-18T13:50:02.905+0000",
    "path": "/user",
    "status": 400,
    "error": "Bad Request",
    "message": "Validation failed for argument at index 0 in method: public cn.hyperchain.validation.User cn.hyperchain.validation.UserController.save(cn.hyperchain.validation.User), with 2 error(s): [Field error in object 'user' on field 'cardNumber': rejected value [CMM-END111]; codes [CardNumerValidation.user.cardNumber,CardNumerValidation.cardNumber,CardNumerValidation.java.lang.String,CardNumerValidation]; arguments [org.springframework.context.support.DefaultMessageSourceResolvable: codes [user.cardNumber,cardNumber]; arguments []; default message [cardNumber]]; default message [没想到吧哈哈哈]] [Field error in object 'user' on field 'cardNumber': rejected value [CMM-END111]; codes [CardNumerValidation.user.cardNumber,CardNumerValidation.cardNumber,CardNumerValidation.java.lang.String,CardNumerValidation]; arguments [org.springframework.context.support.DefaultMessageSourceResolvable: codes [user.cardNumber,cardNumber]; arguments []; default message [cardNumber]]; default message [{自定义校验字段不通过}]] ",
    "errors": [
        {
            "codes": [
                "CardNumerValidation.user.cardNumber",
                "CardNumerValidation.cardNumber",
                "CardNumerValidation.java.lang.String",
                "CardNumerValidation"
            ],
            "arguments": [
                {
                    "codes": [
                        "user.cardNumber",
                        "cardNumber"
                    ],
                    "arguments": null,
                    "defaultMessage": "cardNumber",
                    "code": "cardNumber"
                }
            ],
            "defaultMessage": "没想到吧哈哈哈",  
            "objectName": "user",
            "field": "cardNumber",
            "rejectedValue": "CMM-END111",
            "bindingFailure": false,
            "code": "CardNumerValidation"
        },
        {
            "codes": [
                "CardNumerValidation.user.cardNumber",
                "CardNumerValidation.cardNumber",
                "CardNumerValidation.java.lang.String",
                "CardNumerValidation"
            ],
            "arguments": [
                {
                    "codes": [
                        "user.cardNumber",
                        "cardNumber"
                    ],
                    "arguments": null,
                    "defaultMessage": "cardNumber",
                    "code": "cardNumber"
                }
            ],
            "defaultMessage": "{自定义校验字段不通过}",
            "objectName": "user",
            "field": "cardNumber",
            "rejectedValue": "CMM-END111",
            "bindingFailure": false,
            "code": "CardNumerValidation"
        }
    ]
}
```

+ 读读源码
 
1. 经过 HttpMessageConvertor 处理反序列化成一个JavaBean
```java
package org.springframework.validation.beanvalidation;

SpringValidatorAdapter

@Override
public void validate(@Nullable Object target, Errors errors) {
	if (this.targetValidator != null) {
		processConstraintViolations(this.targetValidator.validate(target), errors);
	}
}

```
2. 进入上面代码中的`this.targetValidator.validate`里面去看看（这里是hibernate）

+ 包名称  `org.hibernate.validator.internal.engine` 类名称`ValidatorImpl` 类定义如下
```
public class ValidatorImpl implements Validator, ExecutableValidator {
    ...
}
```
+ 进入`validate`方法

```java
    @Override
	public final <T> Set<ConstraintViolation<T>> validate(T object, Class<?>... groups) {
		Contracts.assertNotNull( object, MESSAGES.validatedObjectMustNotBeNull() );
		sanityCheckGroups( groups );
        
        //获取验证的上下文环境
		ValidationContext<T> validationContext = getValidationContextBuilder().forValidate( object );

		if ( !validationContext.getRootBeanMetaData().hasConstraints() ) {
			return Collections.emptySet();
		}
    
		ValidationOrder validationOrder = determineGroupValidationOrder( groups );
		ValueContext<?, Object> valueContext = ValueContext.getLocalExecutionContext(
				validatorScopedContext.getParameterNameProvider(),
				object,
				validationContext.getRootBeanMetaData(),
				PathImpl.createRootPath()
		);

		return validateInContext( validationContext, valueContext, validationOrder );
	}
```
+ 进入`validateInContext`方法

```
	private <T, U> Set<ConstraintViolation<T>> validateInContext(ValidationContext<T> validationContext, ValueContext<U, Object> valueContext,
			ValidationOrder validationOrder) {
		if ( valueContext.getCurrentBean() == null ) {
			return Collections.emptySet();
		}

		BeanMetaData<U> beanMetaData = valueContext.getCurrentBeanMetaData();
		if ( beanMetaData.defaultGroupSequenceIsRedefined() ) {
			validationOrder.assertDefaultGroupSequenceIsExpandable( beanMetaData.getDefaultGroupSequence( valueContext.getCurrentBean() ) );
		}

		// process first single groups. For these we can optimise object traversal by first running all validations on the current bean
		// before traversing the object.
		//分组处理
		Iterator<Group> groupIterator = validationOrder.getGroupIterator();
		while ( groupIterator.hasNext() ) {
			Group group = groupIterator.next();
			valueContext.setCurrentGroup( group.getDefiningClass() );
			//我们的核心处理代码就是在这一步调用处理的
			validateConstraintsForCurrentGroup( validationContext, valueContext );
			if ( shouldFailFast( validationContext ) ) {
				return validationContext.getFailingConstraints();
			}
		}
		groupIterator = validationOrder.getGroupIterator();
		while ( groupIterator.hasNext() ) {
			Group group = groupIterator.next();
			valueContext.setCurrentGroup( group.getDefiningClass() );
			validateCascadedConstraints( validationContext, valueContext );
			if ( shouldFailFast( validationContext ) ) {
				return validationContext.getFailingConstraints();
			}
		}

		// now we process sequences. For sequences I have to traverse the object graph since I have to stop processing when an error occurs.
		Iterator<Sequence> sequenceIterator = validationOrder.getSequenceIterator();
		while ( sequenceIterator.hasNext() ) {
			Sequence sequence = sequenceIterator.next();
			for ( GroupWithInheritance groupOfGroups : sequence ) {
				int numberOfViolations = validationContext.getFailingConstraints().size();

				for ( Group group : groupOfGroups ) {
					valueContext.setCurrentGroup( group.getDefiningClass() );

					validateConstraintsForCurrentGroup( validationContext, valueContext );
					if ( shouldFailFast( validationContext ) ) {
						return validationContext.getFailingConstraints();
					}

					validateCascadedConstraints( validationContext, valueContext );
					if ( shouldFailFast( validationContext ) ) {
						return validationContext.getFailingConstraints();
					}
				}
				if ( validationContext.getFailingConstraints().size() > numberOfViolations ) {
					break;
				}
			}
		}
		return validationContext.getFailingConstraints();
	}
```

+ 验证后续处理  
包名称`org.springframework.web.reactive.result.method.annotation`   
类名称`AbstractMessageReaderArgumentResolver`  
方法名`validate`   

```java
    private void validate(Object target, Object[] validationHints, MethodParameter param, BindingContext binding, ServerWebExchange exchange) {
        String name = Conventions.getVariableNameForParameter(param);
        WebExchangeDataBinder binder = binding.createDataBinder(exchange, target, name);
        binder.validate(validationHints);
        //要是有出现错误就抛出异常
        if (binder.getBindingResult().hasErrors()) {
            throw new WebExchangeBindException(param, binder.getBindingResult());
        }
    }
    
    //后续就只能交由 Spring MVC 或 WebFlux 处理   
    //经过本人调试 MVC 与 WebFlux 貌似这一步之后的处理逻辑不同 
```










  


