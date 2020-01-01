# Spring AOP 实战

## 一、AOP介绍

~~~shell
	在软件业，AOP为Aspect Oriented Programming的缩写，意为：面向切面编程，通过预编译方式和运行期动态代理实现程序功能的统一维护的一种技术。AOP是OOP的延续，是软件开发中的一个热点，也是Spring框架中的一个重要内容，是函数式编程的一种衍生范型。利用AOP可以对业务逻辑的各个部分进行隔离，从而使得业务逻辑各部分之间的耦合度降低，提高程序的可重用性，同时提高了开发的效率。        --百度百科
~~~

​		 AOP是Spring框架面向切面的编程思想，AOP采用一种称为“横切”的技术，将涉及多业务流程的通用功能抽取并单独封装，形成独立的切面，在合适的时机将这些切面横向切入到业务流程指定的位置中。 

​		 用自己通俗易懂的话来讲：业务流程大致是从request->controller->service->dao->db，一种竖着的结构，而AOP就是横向织入业务流程

## 二、术语定义

- **Aspect（切面）***： Aspect 声明类似于 Java 中的类声明，在 Aspect 中会包含着一些 Pointcut 以及相应的 Advice。

- **Joint point（连接点）**：表示在程序中明确定义的点，典型的包括方法调用，对类成员的访问以及异常处理程序块的执行等等，它自身还可以嵌套其它 joint point。

- **Pointcut（切点）**：表示一组 joint point，这些 joint point 或是通过逻辑关系组合起来，或是通过通配、正则表达式等方式集中起来，它定义了相应的 Advice 将要发生的地方。

- **Advice（增强）**：Advice 定义了在 Pointcut 里面定义的程序点具体要做的操作，它通过 before、after 和 around 来区别是在每个 joint point 之前、之后还是代替执行的代码。

- **Target（目标对象）**：织入 Advice 的目标对象.。

- **Weaving（织入）**：将 Aspect 和其他对象连接起来, 并创建 Adviced object 的过程

​		术语可能不太好理解，用于一个类比关系就是`Aspect`就是定义在项目中的xxxAOP,`Joint point`是项目中所有请求内容，`pointcut`是用于记录的对象，可以是Controller也可以是Annotation



## 三、项目实战

### 1、日志记录

​		AOP最常见的一个功能就是日志记录。常用于出入参信息记录，敏感操作记录。

#### ①出入参记录

​		开发阶段或者生产阶段对于bug的出现，有时候可能是由数据造成的问题而非代码的问题，这时可以利用Aop记录出入参数，观看参数是否符合预期：

~~~java
	@Before("log() ")
    public void exBefore(JoinPoint joinPoint) {
        ServletRequestAttributes requestAttributes = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
        HttpServletRequest request = requestAttributes.getRequest();
        Object[] args = joinPoint.getArgs();
        log.info("请求url:{}", request.getRequestURL());
        log.info("请求type:{}", request.getMethod());
        log.info("请求method:{}", joinPoint.getSignature());
        log.info("请求参数args:{}", Arrays.toString(args));
    }

	@After("log()")
    public void exAfter(JoinPoint joinPoint) {
        log.info("class method:{}.{}执行完毕!", joinPoint.getSignature().getDeclaringTypeName(), joinPoint.getSignature().getName());
    }

    @AfterReturning(returning = "result",pointcut = "log()")
    public void exAfterRunning(Object result){
        log.info("执行返回值:{}",JSON.toJSON(result));
    }
~~~

#### ②操作记录

​		对于一些比较重要的操作，譬如系统管理，这是一个系统里面操作比较少的功能，但每一个都是比较重要的功能，所以对于这些操作需要利用AOP记录下来，然后写入DB数据库，方便查看：

~~~java
	@Pointcut(value = "@annotation(com.njzhxt.spaas.common.annotion.BussinessLog)")
    public void cutService() {
    }

    @Around("cutService()")
    public Object recordSysLog(ProceedingJoinPoint point) throws Throwable {
        //先执行业务
        Object result = point.proceed();
        try {
            handle(point);
        } catch (Exception e) {
            log.error("日志记录出错!", e);
        }
        return result;
    }

    private void handle(ProceedingJoinPoint point) throws Exception {
        //获取拦截的方法名
        Signature sig = point.getSignature();
        MethodSignature msig = null;
        if (!(sig instanceof MethodSignature)) {
            throw new IllegalArgumentException("该注解只能用于方法");
        }
        msig = (MethodSignature) sig;
        Object target = point.getTarget();
        Method currentMethod = target.getClass().getMethod(msig.getName(), msig.getParameterTypes());
        String methodName = currentMethod.getName();
        String className = point.getTarget().getClass().getName();

        //获取操作名称
        BussinessLog annotation = currentMethod.getAnnotation(BussinessLog.class);
        String bussinessName = annotation.value();
        String key = annotation.key();
        Class dictClass = annotation.dict();
        Object arg = point.getArgs()[0];
        String msg;
        if (bussinessName.contains("修改") || bussinessName.contains("编辑")) {
            Object obj1 = LogObjectHolder.me().get();
            msg = Contrast.contrastObj(dictClass, key, obj1, arg);
        } else {
            AbstractDictMap dictMap = (AbstractDictMap) dictClass.newInstance();
            msg = Contrast.parseMutiKey(dictMap, key, arg);
        }

        operationLogService.insertOperationLog(LogType.BUSSINESS,LogSucceed.SUCCESS,bussinessName,className,methodName,msg);
    }
~~~

![image-20191204102820565](http://image.yangyhao.top/image-20191204102820565.png)

### 2、参数校验

​		我们进行业务代码编写的时候经常需要对参数进行校验，看是否符合预期，那么就会写下如下的代码：

![image-20191204103202906](http://image.yangyhao.top/image-20191204103202906.png)

​		写多了，就会感觉很麻烦，那么有没有能够优化体验的呢？

​		肯定有，那就是[JSR303数据校验](https://www.ibm.com/developerworks/cn/java/j-lo-jsr303/index.html)

​		写法如下：

​		1）首先在需要校验的对象上面加上JSR303的注解，比如`@Length`，`@NotBlank`

​		2）在controller上对对象加上@Valid注解，后面加上参数`BindingResult`

​		3）然后在处理业务之前判断校验是否有错误，有错误就抛出异常

~~~java
public ResponseData add(@RequestBody @Valid MenuDTO menuDTO, BindingResult result){
        if(result.hasErrors()){
            throw new RuntimeException();
        }
		...       
}
~~~

​		这样看起来简单了很多，但是每一个业务方法还是需要判断参数是否有错误，看起来还是不够简洁啊，那么AOP可以从中作出哪些优化呢？

​		思路很简单，AOP可以获取到SpringMVC的入参，然后如果有BindingResult的则统一进行校验即可：

~~~java
	@Before("log() ")
    public void exBefore(JoinPoint joinPoint) {
        Object[] args = joinPoint.getArgs();
        for (Object arg :args){
            if(arg instanceof BindingResult){
                BindingResult bindingResult = (BindingResult) arg;
                APIResult result = this.vaildReuqestParams(bindingResult);
                if(!result.isSuccess()){
                    //如果绑定失败则抛出异常
                    throw new BindErrorException(result.getData().toString());
                }
            }
        }
    }

	private APIResult vaildReuqestParams(BindingResult bindingResult){
        if(bindingResult.hasErrors()){
            List<ObjectError> allErrors = bindingResult.getAllErrors();
            List<String> list = new ArrayList<>();
            for (ObjectError error :allErrors){
                FieldError fieldError = (FieldError) error;
                //错误信息拼接
                list.add(fieldError.getField()+error.getDefaultMessage());
            }
            return APIResult.error(BaseEnum.PARAM_HAS_ERROR,list);
        }
        return APIResult.ok();
    }
~~~

​		那这样就可以很愉快的抛弃繁琐的参数校验了

### 3、插入检查

​		对于Insert和Update操作，关键字段的数据重复肯定是无法接受的，所以我们写代码在插入数据之前肯定会有重复性检查： 	

```
List<Menu> exist = sqlManager.query(Menu.class)
                .andEq("name", menuDTO.getName())
                .andEq("code", menuDTO.getCode())
                .select();
```

​		其实这些操作完成可以通过AOP实现：

~~~java
	@Pointcut(value = "@annotation(com.njzhxt.spaas.common.annotion.InsertCheck)")
    private void check() {
    }

    @Before("check()")
    public void insertCheck(JoinPoint joinPoint) throws IllegalAccessException {
        MethodSignature ms = (MethodSignature) joinPoint.getSignature();
        Method method = ms.getMethod();
        //获取注解
        InsertCheck annotation = method.getAnnotation(InsertCheck.class);
        //获取注解内的参数
        String[] eqFields = annotation.eqFields();
        String[] noEqFields = annotation.noEqFields();
        Object[] args = joinPoint.getArgs();
        StringBuilder sb = new StringBuilder("where 1=1 ");
        List<String> typeList = Arrays.asList(types);
        for (Object arg :args){
            //获取类型名称
            String typeName = arg.getClass().getTypeName();
            if(typeList.contains(typeName)){
                //如果是基础类型
                System.out.println("暂不检测基本类型");
            }else {
                if(!(arg instanceof TailBean)){
                    return;
                }
                //如果不是基础类型，是对象则利用反射获取
                Field[] fields = arg.getClass().getDeclaredFields();
                for (String eq :eqFields){
                    for (Field field : fields) {
                        field.setAccessible(true);
                        if(eq.equals(field.getName())){
                            sb.append(" and ").append(StringUtil.camel2Underscore(eq)).append(" = ").append("'").append(field.get(arg)).append("'");
                        }
                    }
                }
                for (String noEq : noEqFields){
                    for (Field field : fields) {
                        field.setAccessible(true);
                        if(noEq.equals(field.getName())){
                            sb.append(" and ").append(StringUtil.camel2Underscore(noEq)).append(" <> ").append("'").append(field.get(arg)).append("'");
                        }
                    }
                }
                List<?> list = sqlManager.query(arg.getClass()).appendSql(sb.toString()).select();
                if(CollectionUtils.isNotEmpty(list)){
                    throw new ServiceException(BizExceptionEnum.CAN_NOT_ADD_AGAIN);
                }
                System.out.println("插入校验检测通过");
                return;
            }
        }

    }
~~~

​			之后完全可以利用注解实现了：

~~~java
	@InsertCheck(eqFields = {"name","code"})
~~~

​			当然这里只是简单的校验，如果涉及复杂的查询判断，比如 and a= xxx or b=xxx 或者（）这些可以去更深层次的研究。

## 四、源码解析

​			首先我们知道AOP是通过动态代理实现的，而动态代理又分为JDK动态代理和CGLIB动态代理.

​			代理模式很简单，接口 + 真实实现类 + 代理类，其中 真实实现类 和 代理类 都要实现接口，实例化的时候要使用代理类。所以，Spring AOP 需要做的是生成这么一个代理类，然后**替换掉**真实实现类来对外提供服务。

​			替换的过程怎么理解呢？在 Spring IOC 容器中非常容易实现，就是在 getBean(…) 的时候返回的实际上是代理类的实例，而这个代理类我们自己没写代码，它是 Spring 采用 JDK Proxy 或 CGLIB 动态生成的。

​			大家都知道mybatis 的mapper就是通过JDK的动态实现的，而对于AOP的规则很相似，对于接口回使用JDK代理，其他的用CGLIB。