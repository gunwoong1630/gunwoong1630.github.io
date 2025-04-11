---
layout: post
title: Troble shoting - org.springframework.beans.NotReadablePropertyException 
description: >
  Howdy! This is an example blog post that shows several types of HTML content supported in this theme.
sitemap: false
category: [project,velog-export]
hide_last_modified: true
---

# 상황 및 에러 로그

상황 : 프로젝트 진행 과정에서 컨트롤러에서 model을 통해 특정 객체를 프론트에 접근하는 과정, 데이터 바인딩 과정

```
Caused by: org.springframework.beans.NotReadablePropertyException: Invalid property 'isReplaceImgUrl' of bean class [com.velogexport.velogexport.domain.VelogDetail]: Bean property 'isReplaceImgUrl' is not readable or has an invalid getter method: Does the return type of the getter match the parameter type of the setter?
	at org.springframework.beans.AbstractNestablePropertyAccessor.getPropertyValue(AbstractNestablePropertyAccessor.java:627)
	at org.springframework.beans.AbstractNestablePropertyAccessor.getPropertyValue(AbstractNestablePropertyAccessor.java:617)
	at org.springframework.validation.AbstractPropertyBindingResult.getActualFieldValue(AbstractPropertyBindingResult.java:105)
	at org.springframework.validation.AbstractBindingResult.getFieldValue(AbstractBindingResult.java:225)
	at org.springframework.web.servlet.support.BindStatus.<init>(BindStatus.java:129)
	at org.springframework.web.servlet.support.RequestContext.getBindStatus(RequestContext.java:928)
	at org.thymeleaf.spring6.context.webmvc.SpringWebMvcThymeleafRequestContext.getBindStatus(SpringWebMvcThymeleafRequestContext.java:232)
	at org.thymeleaf.spring6.util.FieldUtils.getBindStatusFromParsedExpression(FieldUtils.java:306)
	at org.thymeleaf.spring6.util.FieldUtils.getBindStatus(FieldUtils.java:253)
	at org.thymeleaf.spring6.util.FieldUtils.getBindStatus(FieldUtils.java:227)
	at org.thymeleaf.spring6.processor.AbstractSpringFieldTagProcessor.doProcess(AbstractSpringFieldTagProcessor.java:174)
	at org.thymeleaf.processor.element.AbstractAttributeTagProcessor.doProcess(AbstractAttributeTagProcessor.java:74)
	... 63 more
```

# 기본 지식

``` java
public class NotReadablePropertyExceptionTest {
    @Test
    void contextLoads() {
        Temp temp = new Temp();
        temp.getValueA();
        temp.isValueB();
        temp.isValueC();

        temp.setValueA("");
        temp.setValueB(false);
        temp.setValueC(false);

    }

    @Data
    class Temp {
        private String valueA;
        private boolean valueB;
        private boolean isValueC;
    }
}

```

예시 코드와 같이 boolean인것과 boolean이 아닌 것들이 @Getter와 @Setter가 아래의 표와 같이 다르게 메서드가 생성된다.

|      | Getter   | Setter |
|:---------|:----------|:----------|
| boolean    |        is~ |        set~ |
| not boolean      |         get~ |         set~ |

# 원인 분석 및 디버그

원인 객체는 아래와 같습니다.

``` java
@Data
public class VelogDetail {
    private String id;
    private boolean isReplaceImgUrl;
}
```

이제 에러 로그 발자국을 따라가보겠습니다.

``` java
    @Nullable
    protected Object getPropertyValue(PropertyTokenHolder tokens) throws BeansException {
        String propertyName = tokens.canonicalName;
        String actualName = tokens.actualName;
        PropertyHandler ph = this.getLocalPropertyHandler(actualName);
        if (ph != null && ph.isReadable()) {
            // 생략
        } else {
            // ph가 null이라 아래의 Exception 발생
            throw new NotReadablePropertyException(this.getRootClass(), this.nestedPath + propertyName);
        }
    }
```

우리가 객체 바인딩 과정에서 에러가 발생했었다. 그래서 해당 코드는 프로퍼티를 가져오는 과정에서 에러가 발생함을 알수 있고 , 코드에서는 ph 가 null이라서 NotReadablePropertyException을 던지고 있었습니다.

그렇다면 왜 ph, 즉 PropertyHeader가 null이 됐을까?

getLocalPropertyHandler 메서드를 한번 찾아보자

``` java
    // 생략

    @Nullable
    protected BeanPropertyHandler getLocalPropertyHandler(String propertyName) {
        PropertyDescriptor pd = this.getCachedIntrospectionResults().getPropertyDescriptor(propertyName);
        return pd != null ? new BeanPropertyHandler((GenericTypeAwarePropertyDescriptor)pd) : null;
    }

    // 생략
```

```java
    // 생략

    private final Map<String, PropertyDescriptor> propertyDescriptors;

    @Nullable
    PropertyDescriptor getPropertyDescriptor(String name) {
        // this.propertyDescriptors의 실제값은 아래 이미지로 삽입
        PropertyDescriptor pd = (PropertyDescriptor)this.propertyDescriptors.get(name);
        if (pd == null && StringUtils.hasLength(name)) {
            pd = (PropertyDescriptor)this.propertyDescriptors.get(StringUtils.uncapitalize(name));
            if (pd == null) {
                pd = (PropertyDescriptor)this.propertyDescriptors.get(StringUtils.capitalize(name));
            }
        }

        return pd;
    }

    // 생략
```

getLocalPropertyHandler메서드에서 getPropertyDescriptor 메서드를 사용하고 있고 getPropertyDescriptor메서드를 찾아보면 this.propertyDescriptors를 get하고 있음을 볼수 있다.

디버그로 찍어서 this.propertyDescriptors의 실제 값을 살펴보면 아래와 같다.

![exp1](/assets/img/blog/project/velog-export/troubleShooting-NotReadablePropertyException1.png)

그랬더니 이상하게도 내가 분명 객체의 변수의 이름은 id와 isReplaceImgUrl인데, id만 정상적으로 있고 isReplaceImgUrl가 replaceImgUrl로 변경되었을까?

왜 this.propertyDescriptors에 이렇게 저장돼었는가를 보기위해 this.propertyDescriptors의 생성 과정을 디버깅해보았다.


``` java
    private static BeanInfo getBeanInfo(Class<?> beanClass) throws IntrospectionException {
        Iterator var1 = beanInfoFactories.iterator();

        BeanInfo beanInfo;
        do {
            if (!var1.hasNext()) {
                return simpleBeanInfoFactory.getBeanInfo(beanClass);
            }

            BeanInfoFactory beanInfoFactory = (BeanInfoFactory)var1.next();
            beanInfo = beanInfoFactory.getBeanInfo(beanClass);
        } while(beanInfo == null);

        return beanInfo;
    }

    private CachedIntrospectionResults(Class<?> beanClass) throws BeansException {
        try {
            // logging 생략

            this.propertyDescriptors = new LinkedHashMap();
            Set<String> readMethodNames = new HashSet();
            PropertyDescriptor[] pds = this.beanInfo.getPropertyDescriptors();
            PropertyDescriptor[] var4 = pds;
            int var5 = pds.length;

            for(int var6 = 0; var6 < var5; ++var6) {
                PropertyDescriptor pd = var4[var6];
                if ((Class.class != beanClass || "name".equals(pd.getName()) || pd.getName().endsWith("Name") && String.class == pd.getPropertyType()) && (URL.class != beanClass || !"content".equals(pd.getName())) && (pd.getWriteMethod() != null || !this.isInvalidReadOnlyPropertyType(pd.getPropertyType(), beanClass))) {
                    // logging 생략
                    pd = this.buildGenericTypeAwarePropertyDescriptor(beanClass, pd);
                    this.propertyDescriptors.put(pd.getName(), pd);
                    Method readMethod = pd.getReadMethod();
                    if (readMethod != null) {
                        readMethodNames.add(readMethod.getName());
                    }
                }
            }

            for(Class<?> currClass = beanClass; currClass != null && currClass != Object.class; currClass = currClass.getSuperclass()) {
                this.introspectInterfaces(beanClass, currClass, readMethodNames);
            }

            this.introspectPlainAccessors(beanClass, readMethodNames);
        } catch (IntrospectionException var9) {
            throw new FatalBeanException("Failed to obtain BeanInfo for class [" + beanClass.getName() + "]", var9);
        }
    }
```

뭔가 코드가 장황한데 포인트는 `simpleBeanInfoFactory.getBeanInfo(beanClass)`부분이다.

this.propertyDescriptors에 프로퍼티 데이터를 넣기 위해 simpleBeanInfoFactory의 getBeanInfo 메서드를 통해 생성된 것을 전처리한 것이다.

그렇다면 `simpleBeanInfoFactory.getBeanInfo(beanClass)`를 보자

``` java
class SimpleBeanInfoFactory implements BeanInfoFactory, Ordered {
    SimpleBeanInfoFactory() {
    }

    @NonNull
    public BeanInfo getBeanInfo(final Class<?> beanClass) throws IntrospectionException {
        final Collection<? extends PropertyDescriptor> pds = PropertyDescriptorUtils.determineBasicProperties(beanClass);
        return new SimpleBeanInfo() {
            public BeanDescriptor getBeanDescriptor() {
                return new BeanDescriptor(beanClass);
            }

            public PropertyDescriptor[] getPropertyDescriptors() {
                return (PropertyDescriptor[])pds.toArray(PropertyDescriptorUtils.EMPTY_PROPERTY_DESCRIPTOR_ARRAY);
            }
        };
    }

    public int getOrder() {
        return 2147483646;
    }
}
```

``` java
    public static Collection<? extends PropertyDescriptor> determineBasicProperties(Class<?> beanClass) throws IntrospectionException {
        Map<String, BasicPropertyDescriptor> pdMap = new TreeMap();
        Method[] var2 = beanClass.getMethods();
        int var3 = var2.length;

        for(int var4 = 0; var4 < var3; ++var4) {
            Method method = var2[var4];
            String methodName = method.getName();
            boolean setter;
            byte nameIndex;
            if (methodName.startsWith("set") && method.getParameterCount() == 1) {
                setter = true;
                nameIndex = 3;
            } else if (methodName.startsWith("get") && method.getParameterCount() == 0 && method.getReturnType() != Void.TYPE) {
                setter = false;
                nameIndex = 3;
            } else {
                if (!methodName.startsWith("is") || method.getParameterCount() != 0 || method.getReturnType() != Boolean.TYPE) {
                    continue;
                }

                setter = false;
                nameIndex = 2;
            }

            String propertyName = StringUtils.uncapitalizeAsProperty(methodName.substring(nameIndex));
            if (!propertyName.isEmpty()) {
                BasicPropertyDescriptor pd = (BasicPropertyDescriptor)pdMap.get(propertyName);
                if (pd != null) {
                    Method readMethod;
                    if (setter) {
                        readMethod = pd.getWriteMethod();
                        if (readMethod != null && !readMethod.getParameterTypes()[0].isAssignableFrom(method.getParameterTypes()[0])) {
                            pd.addWriteMethod(method);
                        } else {
                            pd.setWriteMethod(method);
                        }
                    } else {
                        readMethod = pd.getReadMethod();
                        if (readMethod == null || readMethod.getReturnType() == method.getReturnType() && method.getName().startsWith("is")) {
                            pd.setReadMethod(method);
                        }
                    }
                } else {
                    pd = new BasicPropertyDescriptor(propertyName, !setter ? method : null, setter ? method : null);
                    pdMap.put(propertyName, pd);
                }
            }
        }

        return pdMap.values();
    }
```

`simpleBeanInfoFactory.getBeanInfo(beanClass)`는 `PropertyDescriptorUtils.determineBasicProperties(beanClass)`를 사용하고 드디어 어떻게 저장되어있는지 직접적으로 보여주는 코드를 발견했다.

객체의 메서드가 "set"이나 "get"으로 들어왔을 경우에 `BasicPropertyDescriptor`로 저장한다. 즉, "set"과 "get", "is"을 통해 객체 바인딩이 진행됨을 알게되었다.

하지만 이 것은 원인이 아니다. (뜻밖의 수확)

**문제의 부분은 바로 @Data였다. 정확히 얘기하자만 @Getter 혹은 @Data를 사용하는 환경에서 boolean 자료형의 변수 이름이 is로 시작할 경우가 문제였다.**

위 코드를 보면 메서드들을 전체 조회하는 과정에서 "get"이나 "set", "is" 일때는 앞의 글자수를 세서 substring을 했다. 

그리고 @Getter는 boolean 자료형에 대해서 변수 이름이 is~일때 is~() 메서드로 get을 구현한다. 이부분이 문제였는데 위 코드에서 보면 앞에 is일때 nameIndex를 2로 지정하여 substring을 하기 때문에 기존의 자료형 이름이 is~였다면 프로퍼티에 얼렁뚱땅하게 is가 지워진 변수 이름으로 저장되어 에러가 발생했다. 아래의 예시를 보자

``` java
@Data
public class VelogDetail {
    private String id;
    private boolean isReplaceImgUrl;
}
```

위 코드처럼 `isReplaceImgUrl`라는 boolean 변수가 있을 때 Getter는 `isReplaceImgUrl()`로 실행된다. 그래서 `PropertyDescriptorUtils.determineBasicProperties(beanClass)`를 통해서 원래 `isReplaceImgUrl`변수 이름으로 저장되어야 하는데 앞에 is가 붙어있어서 전처리되어서 `replaceImgUrl`로 저장된다. 그럼으로 이후에 변수 이름을 기준으로 프로퍼티를 조회할때 당연히 이름이 안맞기 때문에 에러가 발생했엇다!!


# 해결

해결책은 아래와 같이 여러가지이다.

1. boolean 자료형을 Boolean으로 변경
2. Getter 메서드를 임의로 생성

본인은 1번 방법을 사용하여 문제를 해결하였다.