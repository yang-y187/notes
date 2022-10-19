## 3、参数处理原理

接收参数处理，进入 doServlet()->doDispatch()

方法中：

##### 3.1，首先，在许多的HandlerMapping中找到合适的Handler

```Java
mappedHandler = this.getHandler(processedRequest);
```

- 

```Java
@Nullable
    protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
        if (this.handlerMappings != null) {
            Iterator var2 = this.handlerMappings.iterator();

            while(var2.hasNext()) {
                HandlerMapping mapping = (HandlerMapping)var2.next();
                HandlerExecutionChain handler = mapping.getHandler(request);
                if (handler != null) {
                    return handler;
                }
            }
        }
```

##### 3.2，为当前的Handler找到合适的适配器HandlerAdapter

```Java
HandlerAdapter ha = this.getHandlerAdapter(mappedHandler.getHandler());
```

![image-20220111090834908](D:\我的坚果云\springBootImg\Image5.png)

**HandlerAdapter**：

​	0-对应的是自己编写的controller  即有@RestController的类，最终是**支持@RequestMapping 修饰的方法**

​	1-函数编程的适配器

​	2,3为系统自带的适配器



3.3，最终执行在handle

```
mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
```

handle()->handleInternal()->invokeHandlerMethod()真正执行的方法

invokeHandlerMethod()中：

​	现在各种参数赋值，以及返回值类型，最终调用 invokeAndHandle执行



##### 3.3 **this.argumentResolvers():参数解析器的集合**，即包含标注各种注解的参数，RequestParam，PathVariable....共27种

​	Controller中方法能写多少种参数类型。取决于参数解析器。

![image-20220111092446889](D:\我的坚果云\springBootImg\image-20220111092446889.png)

##### 3.4 **this.returnValueHandlers():返回值处理器**，即方法可以返回的返回值类型，Model ，view，或ModelAndView共15种

![image-20220111092905515](D:\我的坚果云\springBootImg\image-20220111092905515.png)

##### 3.5 执行目标方法，获取目法的参数值

**invokeAndHandle（）调用invokeForRequest**，真正执行目标方法，即RequestMapping修饰的方法

- invokeForRequest方法中：

​	Object[] args = this.getMethodArgumentValues(request, mavContainer, providedArgs);//获取相对应的参数

```Java
  protected Object[] getMethodArgumentValues(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer, Object... providedArgs) throws Exception {
        MethodParameter[] parameters = this.getMethodParameters();
        if (ObjectUtils.isEmpty(parameters)) {
            return EMPTY_ARGS;
        } else {
            Object[] args = new Object[parameters.length];

            for(int i = 0; i < parameters.length; ++i) {
                MethodParameter parameter = parameters[i];
                parameter.initParameterNameDiscovery(this.parameterNameDiscoverer);
                args[i] = findProvidedArgument(parameter, providedArgs);
                if (args[i] == null) {
                    if (!this.resolvers.supportsParameter(parameter)) {
                        throw new IllegalStateException(formatArgumentError(parameter, "No suitable resolver"));
                    }

                    try {
                        args[i] = this.resolvers.resolveArgument(parameter, mavContainer, request, this.dataBinderFactory);
                    } catch (Exception var10) {
                        if (logger.isDebugEnabled()) {
                            String exMsg = var10.getMessage();
                            if (exMsg != null && !exMsg.contains(parameter.getExecutable().toGenericString())) {
                                logger.debug(formatArgumentError(parameter, exMsg));
                            }
                        }

                        throw var10;
                    }
                }
            }

            return args;
        }
    }
```

在上述方法中

- 遍历所有参数解析器，判断是否可以解析该参数

```Java
//this.resolvers.supportsParameter(parameter) ，判定参数解析器是否支持该参数的解析，会调用getArgumentResolver，获取对应的参数解析器
@Nullable
	private HandlerMethodArgumentResolver getArgumentResolver(MethodParameter parameter) {
		HandlerMethodArgumentResolver result = this.argumentResolverCache.get(parameter);
		if (result == null) {
			for (HandlerMethodArgumentResolver resolver : this.argumentResolvers) {
				//supportsParameter，仅仅是判断该参数的修饰注解是否是当前参数解析器的注解，是 true，不是 false
                if (resolver.supportsParameter(parameter)) {
					result = resolver;
                    //加入缓存，方便解析参数的时候直接调用该参数解析器
					this.argumentResolverCache.put(parameter, result);
					break;
				}
			}
		}
		return result;
	}
```

最终会找到对应的参数解析器

- **参数解析**

调用resolveArgument（）->resolver.resolveArgument()->

```Java
	@Override
	@Nullable
	public final Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
			NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {

		NamedValueInfo namedValueInfo = getNamedValueInfo(parameter);
		MethodParameter nestedParameter = parameter.nestedIfOptional();

		Object resolvedName = resolveEmbeddedValuesAndExpressions(namedValueInfo.name);
		if (resolvedName == null) {
			throw new IllegalArgumentException(
					"Specified name must not resolve to null: [" + namedValueInfo.name + "]");
		}
		//实际获取值的方法
		Object arg = resolveName(resolvedName.toString(), nestedParameter, webRequest);
		if (arg == null) {
			if (namedValueInfo.defaultValue != null) {
				arg = resolveEmbeddedValuesAndExpressions(namedValueInfo.defaultValue);
			}
			else if (namedValueInfo.required && !nestedParameter.isOptional()) {
				handleMissingValue(namedValueInfo.name, nestedParameter, webRequest);
			}
			arg = handleNullValue(namedValueInfo.name, arg, nestedParameter.getNestedParameterType());
		}
		else if ("".equals(arg) && namedValueInfo.defaultValue != null) {
			arg = resolveEmbeddedValuesAndExpressions(namedValueInfo.defaultValue);
		}

		if (binderFactory != null) {
			WebDataBinder binder = binderFactory.createBinder(webRequest, null, namedValueInfo.name);
			try {
				arg = binder.convertIfNecessary(arg, parameter.getParameterType(), parameter);
			}
			catch (ConversionNotSupportedException ex) {
				throw new MethodArgumentConversionNotSupportedException(arg, ex.getRequiredType(),
						namedValueInfo.name, parameter, ex.getCause());
			}
			catch (TypeMismatchException ex) {
				throw new MethodArgumentTypeMismatchException(arg, ex.getRequiredType(),
						namedValueInfo.name, parameter, ex.getCause());
			}
			// Check for null value after conversion of incoming argument value
			if (arg == null && namedValueInfo.defaultValue == null &&
					namedValueInfo.required && !nestedParameter.isOptional()) {
				handleMissingValueAfterConversion(namedValueInfo.name, nestedParameter, webRequest);
			}
		}

		handleResolvedValue(arg, namedValueInfo.name, parameter, mavContainer, webRequest);

		return arg;
	}
```


Object arg = resolveName(resolvedName.toString(), nestedParameter, webRequest);获取参数值

```Java
	@Override
	@Nullable
	protected Object resolveName(String name, MethodParameter parameter, NativeWebRequest request) throws Exception {
		HttpServletRequest servletRequest = request.getNativeRequest(HttpServletRequest.class);

		if (servletRequest != null) {
			Object mpArg = MultipartResolutionDelegate.resolveMultipartArgument(name, parameter, servletRequest);
			if (mpArg != MultipartResolutionDelegate.UNRESOLVABLE) {
				return mpArg;
			}
		}

		Object arg = null;
		MultipartRequest multipartRequest = request.getNativeRequest(MultipartRequest.class);
		if (multipartRequest != null) {
			List<MultipartFile> files = multipartRequest.getFiles(name);
			if (!files.isEmpty()) {
				arg = (files.size() == 1 ? files.get(0) : files);
			}
		}
		if (arg == null) {
			String[] paramValues = request.getParameterValues(name);
			if (paramValues != null) {
				arg = (paramValues.length == 1 ? paramValues[0] : paramValues);
			}
		}
		return arg;
	}
```

#### Servlet API：解析参数Servlet参数

WebRequest、ServletRequest、MultipartRequest、 HttpSession、javax.servlet.http.PushBuilder、Principal、InputStream、Reader、HttpMethod、Locale、TimeZone、ZoneId

**ServletRequestMethodArgumentResolver  解析以上的部分参数**



我们当然要彼此依赖，来面对以后的问题。是啊，现在生活问题叠着问题。如果不想向上走，现在肯定很快乐，我们都是学生，没有工作的压力，没有生活的琐碎。可我记得对你说过，我要对你负责。对你负责，不止是在此时此刻，也是在以后的每一分一秒。

我确实挂断了电话，洗澡也可以晚一点去洗。可我并不随意，我已经说明要去洗澡，也让你有时间缓冲，你总会在聊天的最后，抛出一个世纪难题。我知道这不是你真正想得到的。你想看到的是我为你停留下脚步来证明我多在乎你。一定是我对你的爱不明显，让你的一再的以这种方式来证明。我会好好反思的，我也希望你能得到些什么。而且我们应该对彼此更多的理解，而不是彼此限制步伐，我们是一个整体，我束缚了你，也是束缚我自己，因为以后是我们一起生活。

我从未把你想的如何不堪。你在我心里依然是一个可爱有时还有点任性的女孩。可你有时的任性让我不知所措，使得我对你的情感得不到回应，安全感不是一个人的特权。我知道你对我很好，所以我也想加倍对你好。我也知道你想要的不多，你想要的是一个稳定的家庭。这也是我想要的。可稳定的不止是感情。我们还有面对生活的烦恼，比如说，钱。至于礼物，我当然希望送你好的礼物（贵的那种），可是我拿什么来送你，拿我的花呗吗。我确实想要现实一点，可我对你一点也现实不起来。我无法想象，以后你还为了一块半而发愁，那我现在所做的一切都没有意义。我十分害怕以后会这样。

我原本想着你开学再说的，答应我好吗，我们共同努力，我们一起有一个美好的未来





# 阿里面试

- 介绍一下项目，遇到了什么难点，怎么解决的

  答：我完成注册的时候需要一个发送邮件进行激活，不知道怎么弄，查了查官方文档。

  另外，敏感词过滤使用了前缀树，真正用到了数据结构的方式来解决问题

- 讲一下登录功能的实现

  - 请求，页面接收响应，对验证码，用户名，密码（加上随机字符串再md5加密后）对比，redis对比。是否准确

- 你的这个项目，使用的http还是https

  - 我回答使用的http，确实https更安全一点

- 追问，那你请求的用户名密码是怎么加密的，明文发送吗，你怎么保证它的安全性

  - 我说我用的post请求发送，并不是明文，所有内容在请求实体中

- 这样安全性依然不能保证，我仍然可以截获你的请求，获取你的用户名和密码，你怎么解决这个问题的

  - 答：不了解
  - 这个问题就是考HTTPS的保密性。**你要用https作为协议，不能用http。我应该说，在测试时用http，部署到服务器上使用HTTPS，来避免传输的安全性问题**

- 问rentenetlock请求的那个时刻，发生了什么，讲一下。也可以说A，B两个线程同时请求一个锁对象，那一时刻发生了什么

  - **rentenetlock原理：AQS步骤，已经挂起和唤醒巴拉巴拉**

- 你刚才说道了volatail，讲一下volatile叭

  - 能满足可见性，有序性，不能满足原子性，具体原因巴拉巴拉

- 设想一个场景：有一个变量i被volatile修饰，10个线程，每个线程循环10次，i++。能加到100吗

  - 我以为是jvm，我说i++没有在操作栈中执行，i++不能，但是i = i+1可以
  - 实际上都不可以，**因为并发操作一定要加锁，来保证原子性，volatile只保证了可见性和有序性**
  - **volatile与锁是共存的关系，不能只用volatile来解决并发问题，这句话已经看到好多遍，直到遇到才会发现原来自己并不懂**

- 讲一下Unsafe类的原理叭

  - Unsafe类适用了CAS进行赋值比较

- 追问：CAS底层怎么实现的。

  - CAS有三个值，内存值，旧值，新值，进行修改操作时，先判断内存值是否与旧值相等，等，则赋予新值，不等，则循环或者执行结束不赋值

- 追问，底层，

  - 我说Unsafe类的方法通过C/C++编写，与操作系统进行交互，获取到指定对象的地址，根据地址与旧值进行比较

  - 问：与操作系统怎么执行的，什么指令

  - 我不了解，**我现在严重怀疑是不是我说错了，可我就是这么学的呀，**，可能是这句话说错了：与操作系统进行交互

- 5个哲学家问题

  - 有五个哲学家，他们的生活方式是交替地进行思考和进餐。他们共用一张圆桌，分别坐在五张椅子上。

    在圆桌上有五个碗和五支筷子，平时一个哲学家进行思考，饥饿时便试图取用其左、右最靠近他的筷子，只有在他拿到两支筷子时才能进餐。进餐完毕，放下筷子又继续思考。

  - 你打算怎么写代码（并不是将思路层面，又是代码层面的）

  - 没答出来

- 你说你用到了springboot，那你说一下怎么使用事务，代码层面，也就是说你怎么写代码

  - 我说spring有声明式事务和编程式事务，一般选择声明式事务，写注解@Transactional

- 咱们问一点数据库的东西，数据库的隔离级别

  - 巴拉巴拉

- 设想场景，A向B转账，怎么保证它的线程安全

  - 我说加事务加锁，让其单线程执行

- 追问：要是分布式呢，即A向B转账，A向C转账，保证安全，面试官解释了多次，

  - 然后我才明白要用数据库加锁，我并不懂分布式
  - 我说数据库加行锁，

- 追问：怎么加，代码层面，你怎么写

  - 答：我忘了

  - **应该是SQL语句上加锁**

    - **1、表中创建索引， select ... where 字段（必须是索引） 不然行锁就无效。**

      **2、必须要有事务，这样才是 行锁（排他锁）**

      **3、在select 语句后面 加 上 FOR UPDATE；**

- 比如说我们支付过程，由于网路卡顿，或者颠簸，用户点了好几次，你怎么解决这个问题

  - 我说，我确实遇到了这个问题，我选择执行完一个业务后，进行页面跳转，（即使是再次刷新，也是刷新的跳转页面，不再执行前面的业务）
  - 只有这一点是不够的，还需要加入随机码，在每次请求该支付页面时都有一个随机码。

- 面试官：我大概其明白你的意思了

- 问一道算法题，镜像树，你怎么做，说思路

  - 递归判断

- 使用过线程池吗？

  - 没用过，但是我了解一点线程池参数（我有点后悔了，就应该说不了解）

- 那我先说一下：最大线程数据，救急线程数目，核心线程数据，阻塞队列，这三者的关系。

  - 我答：线程阻塞时就使用救急线程
  - 我现在想想，我肯定错了，一定是阻塞队列满了或者超过一定阈值了才用救急线程来执行，

- 你说你写了几篇专利，讲一下专利叭，你做了那部分

  - 我说偏向硬件，巴拉巴拉

- 我看你的专业都是以硬件为主，（已经开始婉拒我了），你是什么专业的，你的计算机基础知识啥的

  - 我说我都是自学的

- 你专业都和硬件相关那你的就业前景是怎么考虑的

  - 我说我打算找后端的工作。更喜欢干这个

- 反问：

  - 我当时已经知道我肯定是过不了了。我就问：volatile并发线程那个问题，怎么说才是对的
  - 面试官回答：说涉及到因为Java会转换成汇编语言，也并不是一个线程执行，这种并发性没有解决，所以达不到100
  - 我又问：我确实答得不好，咱们部门面试都是全部都是高并发吗？
  - 面试官说，这些场景确实能够遇到，有很多问题打不出来，确实需要几年开发经验才能答出来。也没有关系。你的算法题和一些基础知识都答对了
  - 我说：您没必要这样说，这次面试的结果我已经知道了
  - 面试官说：我们面试也看你们所有人的成绩，看你具体能到哪个档。（意思就是说，要是我们招的这批人都垃的话，你就能来我们这）

- 感慨:

  - 与我美团面试差别很大，我以为一面是问八股文，顶多阿里是问的更深一点。但并不是。而是直接问高并发问题，而且从头问到尾。会给你出一个场景，你怎么写代码，而不是简单的理论说一下，具体代码怎么写，会问到这种程度
  - 我开始以为volatile涉及到汇编，CAS涉及到操作系统，post请求涉及到http请求如何避免明文传输，这些完全超出了Java层面的认知。现在我才意识到，可能是我开始答案说错了，所以才会追问，才会这样问。
  - 以10个并发线程那个例子，我虽然知道volatile不能满足原子性，但是遇到真实问题，我依然想不到，而只想到了volatile的可见性，既然可见，那么操作就没问题。（现在想想好搞笑），虽然可见性能满足，但依然是多线程，进行了覆盖操作，达不到100。一个问题让我原形毕露。
  - 反思：这次面试暴露出：我只是一个背了一点八股文的非科班学生，对一些东西并不懂（10个并发线程），而且代码层面也能力不足。



# 华为面试

一面

- jdk与jre的区别
  - jre：Java运行环境，但是遇到jsp页面，不能运行，需要用到jdk
  - jdk，可以编译和创建程序，包含jre。
- 各个容器的区别，list，set，map
  - 共同点是都保存object对象
  - list：ArrayList的底层主要是数组，或者Linkedlist双向链表，
  - set的底层是使用的map，value为null
  - map的底层使用的是数组加链表+红黑树  ，linkedHashMap使用数组加链表+红黑树，外加一个链表，维持顺序
- 追问：再说一下list和set的区别
  - 无序性，list插入数据是有序的，set是无序的，因为使用hash值作为插入顺序。
  - **好像还有时间复杂度相关的东西没说，现在才想起来**
- 面试官还在等待我的回答
  - priorityQueue也实现了list接口，虽然说输出有有序的，但仅仅是维护了一个最小堆，里面仍然是无序的
- 你刚才说到了这类型，那说一下==和equal
- 比如说i=1，b=1，相等吗：等
- Integer i=1，b=1，相等吗，等
  - 常量池的相关东西
- 比如说，student类，两个名称都叫张三，相等吗
  - 不等

- 项目遇到了什么难点
  - 我的邮件发送注册不懂，后来了解到spring-mall，瞬间显得高大上
- 你学的数据库是啥，
  - 数据库的相关东西，MySQL和redis
- 存储过程说一下？ 
  - 这是啥？，是redo log ---记录过程吗，还是查询过程，涉及的索引？
- 那你说一下索引叭，
  - 索引是一种数据结构，主要是提高查询速度，但是需要保存索引，需要空间，维护索引需要时间，对内存要求较高
  - 比如说聚簇索引，和非聚簇索引，需要回表操作，
  - 我应该再说一下，也有特殊情况，比如说，索引下推，和索引覆盖
- 说一下索引的设计规则
  - 巴拉巴拉
- 说一下SQL查询比较慢的情况
  - 我说是查询比较慢哈，（不是等待时间长，）
  - 问：这么说吧，我两个表查询，比较慢，怎么办
    - 那就是explain+SQL，进行分析。
      - explain，查看，先查询哪个表，后查询哪个表，能用到的索引，实际用到的索引，查询的物理行数
    - 以此来看看是不是索引失效了，是不是表结构有问题，如果都没问题，就用MVCC解决

- 你学过jvm叭，问一点jvm相关的东西，堆和方法区的位置，

  - **现在想想，应该是想问，方法区在哪，我应该说在云空间，在直接内存中，堆在jvm内存中**
  - 问：这么问叭，他们里面盛放的是啥
  - 堆存放的对象实例，方法区存放的是类的相关信息，常量，静态变量，方法的一些信息，**（还有元数据信息忘说了）**

- 对象实例什么时候创建呢？

  - 啊哈？不是new 对象的时候创建吗，您是想问对象的创建过程吗，还是类的加载过程？
  - 面试官也笑了，下一个

- gc的特点

  - gc过于宽泛了，但都有停顿时间，进行可达性分析标记，不同的垃圾回收器有不同的特点，主要是分为，吞吐量和响应时间
  - 垃圾回收的算法不同，也会有所不同，标记-整理，清除，复制，巴拉巴拉
  - （**太过于泛泛，不知道从哪回答**）

- gc可以自己调用吗

  - 可以使用system.gc(),是全局的gc，对老年代，新生代，幸存区这些都会进行垃圾回收。不建议自己使用，垃圾回收器会自己进行判断

- 用过runtime.gc吗，

  - 不了解

- system.gc()不是立即执行。

  - **我好像是说错了**

- 算法是：二分法查找起始和终止位置

- 反问：

  - 你们部门主要是处理什么业务

  - 他们说不同部门处理的业务不同，可能一个部门处理的不尽相同，甚至语言也不大一样，

    - 这尼玛是，我面的Java，进去用c?

  - 我想了解一下咱们使用的技术栈

    - 比较主流的框架都使用，数据库方面使用自己的，与mysql类似。
    - 比较新的框架不会使用，等待它稳定以后会进行引入
    - （比较担心，华为的技术不共享，用的都是老旧的组件啥的，看来也不全是）

    

- 总结

  - 回答的一些，还是感觉不是那么在点上，可能是第一场面试的第一个人，所以在回答上，不那么流利
  - 而且回答的东西，不全，总是在缺东西，比如，list和set的问题，需要提醒才能想到无序性的问题
  - 算法方面，
    - 给忘了怎么进行，查找起始位置和终止位置，使用二分法，而是使用最简单的二分法查找到一个target，进行前后遍历，
    - 面试官可能仅仅是想考验我一下二分法，并没有想到，要找起始位置和终止位置，找到一个就行
    - 算法二分法我给忘了，确实是一个很大的问题，让我属实是慌了 



# 简历

A multi-featured time-frequency neural network system for classifying sEMG

https://ieeexplore.ieee.org/document/9790853

2022/6/8

潍柴动力励学金

2021/11/6



130181199711198759

18722373155

wyangyang187@foxmail.com



河北省石家庄市辛集市



电气自动化与信息工程学院

控制科学与工程



人工智能与数据科学学院

新能源科学与工程



暑期实习生

北京三快科技有限公司(美团)

美团/优选事业部/研发部/优选研发部/业务研发组/销售研发组/销售系统组

CRM系统中，作战平台系统的后端开发与物料商城的性能维护

1. SQL慢查询优化，查询请求由176ms，优化到33ms，性能提升81.25%，已上线；
2. 文件导出请求超时优化，请求1.5w条数据的时间由29.97s缩减到6.75s，性能提升77.48%，已上线；
3. 销售组全平台orgId切换，独立完成需求作战平台由业务orgId切换为基础侧orgId，开发完成。





宿舍跳蚤市场交易平台

项目描述：

该交易平台是一个分布式项目，包括：首页门户、商品搜索、商品展示、购物车、订单、会员中心、秒杀活动等微服务；

核心功能实现：

1. 通过压力测试定位系统吞吐量瓶颈并进行系统、DB和静态资源的优化,TPS由1.3k提升至2.3k；
2. 完成下订单购买完整流程，并通过延时队列实现订单过期取消，保证可靠消息的最终一致性；
3. 对接口进行幂等性处理，提交订单避免重复提交以及获取验证码实现接口防刷；
4. demo网址：https://gitee.com/wyy1446514241/mall；



后台开发



学院师生交流论坛

项目描述：

实现了用户注册登录、支持上传头像、发布帖子、发送私信与敏感词过滤，点赞关注功能。
核心功能实现：

1. 使用Redis作为本地缓存保存点赞关注信息，提高系统性能，经测试，TPS可以达到1.7k;
2. 使用Kafka作为消息队列，在被点赞、评论、关注后、放入异步队列，通知该用户;
3. 服务器部署网址：http://39.103.176.33/index



https://github.com/yang-y187/notes



- 熟练掌握Java基础，集合等相关知识。熟悉JVM的垃圾回收机制、类加载机制及Java的内存区域；
- 熟悉Java并发编程，熟悉volatile、synchronize关键字，熟悉多线程，线程池，Java内存模型；
- 熟练使用 MySQL，熟悉MySQL 索引、事务、存储引擎、锁机制、MVCC、日志等；
- 熟悉Redis数据类型使用场景、持久化和过期淘汰策略、高并发缓存场景、分布式锁、集群等；
- 熟悉OSI 和 TCP/IP 网络分层模型结构，掌握常见网络协议，如HTTP/HTTPS 、TCP等；
- 熟悉操作系统的进程通信、死锁、内存管理等知识；
- 熟练使用Spring Boot、Spring、Mybatis等常用框架，熟悉Spring IOC 、自动配置原理；
- GitHub分享：https://github.com/yang-y187/notes；



# 自我介绍

面试官您好，我叫王洋阳，今天我来应聘的是Java后端的开发岗。我现在是一名在读的研究生。本科毕业于坐落于天津的一所211高校河北工业大学。现在研究生阶段，就读于天津大学。我在研究生阶段，写了四篇专利和一篇SCI论文。为了准备后端的工作，我自学了计算机基础知识，包括计算机网络，操作系统，和常见的数据结构。我学的语言是Java，还学习了JVM和高并发的相关知识。数据库方面，学习了MySQL和redis，对于数据库的高级部分都有认真学习。在框架方面，学习了Spring和springboot，了解了一些源码方面知识。学习了一些前端页面的相关知识。

7月份结束了在美团的暑期实习，主要工作是美团优选CRM平台的后端开发，在实习期间，完成了SQL慢查询优化，查询时间由原来的170多ms，降到30ms。任务文件导出请求超时的优化，由30s的请求处理时间优化到6s，并且独立完成了orgId切换业务需求。

在实习期间，锻炼了解决线上问题的能力，和熟悉了美团生产环境的开发，上线流程。

【可说可不说】对开发有了更深的认识，无论是代码方面还是软素质方面。



我搭建了一个宿舍跳蚤市场交易平台，这是一个分布式项目，其中包括首页门户、商品搜索、商品展示、购物车、订单流程、会员中心、秒杀活动等微服务。我的工作主要集中在：性能优化，接口幂等性的处理，完善下订单的整个流程保证最终一致性和秒杀活动。

此外，我搭建了一个师生论坛系统。实现了用户注册登录、支持上传头像、发布帖子、发送私信与敏感词过滤，实现了点赞关注功能。



我的性格比较开朗，能够并且愿意主动和他人沟通，之前的项目中都和周围人有良好的合作关系，

 

自我评价：

有美团后台开发暑期实习经历，天津大学硕士，211本科。学习成绩优秀，计算机基础扎实。技术的应用实战非常过关，能够快速学习各类技术并应用，性格阳光开朗。





交通银行问题81044909822.

1. 我在美团实习期间，独立负责开发一个业务需求，orgId切换。管理层需要将orgId进行切换方便后续加入更大的平台系统

   背景

   我们整个优选销售系统有将近20个微服务 ，我负责作战平台的orgId切换。

​		问题：
​		我们平台内部的orgId是从基础侧获取的，现在要从人员侧获取orgId。我们销售系统中微服务之间会相互调用。并且请求携带这个orgId或者返回携带这个orgId。问题在于：但每个微服务切换时间不固定，我们并不清楚其他平台有没有进行orgId切换，在微服务之间相互调用时，不知道携带新的orgId还是旧的orgId。并且我们已经把旧orgId保存到数据库。我们作战平台内部的业务逻辑凡是涉及到进行切换。
​       另外，由于orgID是整个销售系统的切换，如果切换上线后，假设一个微服务出现了线上问题，需要将orgid切换为旧orgId，我们能不能立即做到将新orgId切换为旧orgId，不影响线上生产。

2. 控制频率越高，能保证控制效果的下限。
   		在事前或事中考虑的，我认为要看我们要实现的目标对已知风险容忍度。
   1. 控制风险发生的频率：就是要进行精准的预测，从而避免风险的发生。比如说：我在美团实习期间，开发时，请求其他微服务，获取到相应的数据后，不能直接进行数据处理，我们需要进行相应的检测，判断是我们想要的数据后，才能处理，不然会出现异常，最终影响线上生产。
   2. 控制风险产生的后果：要控制风险产生的后果就要果断的采取有效措施。我们在进行新功能开发时，需要考虑，如果新功能有问题，我们需要能立即关闭新功能，不使其线上问题影响之前的功能。所以开发时，需要配置开关。 
3. 在美团实习期间，独立负责orgId需求的开发，这个任务是对我来说是很困难的。上面说到的orgId切换 的问题，我先开始写技术文档，在技术文档考虑到所有的问题，并写出所有的解决办法后，才 开始进行开发写代码。利用配置中心中配置流量出口，流量入口的开关，以及使用定时任务进行数据刷新。
4. 首先有的人 拒绝新鲜事物，有的人 追求新鲜事物，是很正常的。首先说，尝试新事物是有风险的，有的人喜欢冒险，有的人不喜欢。
   至于为什么有人不喜欢冒险，每个人的价值观不同，知识量不同，对生命的理解也不同。当然还是就事而论，打比方吃蚕蛹，有的人不了解蚕蛹的营养价值，而且天生就对虫子有一种厌恶感，那TA就不愿意去冒这个险，TA会不自觉地幻想虫子在它肚子里作怪，因而选择不去尝试。

​		每个人都有适合自己的生活方式。我尊重他们每一种人，而不是试图去改变它

5. 有的人喜欢做自己比较擅长或者熟悉的部分，有人会挑战自己。与我来说，会去争取，当然是先总结关键点，然后抓主要矛盾。

   1. 了解任务详情，明确目标及要达到的效果

      2.明确重难点，列计划，自己研究、请教。

      3.随时跟领导沟通，反复修改。

6. 遇到问题首先是解决问题，而不是逃避，。我在实习期间遇到了一个问题。请求orgId切换，原本应该请求到一个数值，但是请求不是我们想要的内容，我晓得这是提供接口的人的问题，所以我就联系上了他，向他反馈一下，这个问题。过了几天后，但问题并没有解决。我又找了他一次，让他解决一下这个 问题。仍然没有效果。然后我就把他拉到我们十几个人的开发群，@他让他修改。此时他才开始解决这个问题。没多久就解决了这个问题。











