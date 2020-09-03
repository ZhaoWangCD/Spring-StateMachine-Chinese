# Spring StateMachine Reference 中文版

写在前面：此部分和Spring StateMachine Reference官方文档基本对应，基于Spring StateMachine状态机开发请先看官方文档，在官方文档后面给出了几个状态机开发的实际用例（用Gradle构建的，可以拉下来项目练习，对了解状态机比较有用，对开发实际项目用处不太大）

spring系列的官方文档一直是简单有效的，主要还是以开发应用为主





## 0.状态机基本知识

先从状态机的定义入手，StateMachine<States, Events>，其中：

- StateMachine：状态机模型

- state：S-状态，一般定义为一个枚举类，如创建、待风控审核、待支付等状态

- event：E-事件，同样定义成一个枚举类，如订单创建、订单审核、支付等，代表一个动作。
  一个状态机的定义就由这两个主要的元素组成，状态及对对应的事件（动作）。
  
  

**简单状态：不含子状态的状态，即只有一个含义的状态**

**复合状态：具有子状态（嵌套状态）的状态。**

> 举例：
>
> 待还款状态包括一分没还和分期还了前几期两种状态

子状态可以是**顺序的（不相交）**或**并发的（正交）**（根据有几条通路判断）。

UML 2.4将复合状态定义为包含一个或多个[区域](https://www.uml-diagrams.org/state-machine-diagrams.html#region)的状态 。顺序一个区域，正交多个区域

（请注意，**区域被定义为复合状态或状态机的正交部分。**）状态不允许同时具有区域和子机。



#### Spring StateMachine状态机中相关概念含义

- Transition: （状态迁移表的）节点，是组成状态机引擎的核心

- source：节点的当前状态

- target：节点的目标状态

- event：触发节点从当前状态到目标状态的动作

- guard：起校验功能，**一般用于校验是否可以执行后续action**

- action：用于实现当前节点对应的业务逻辑处理

  

## 1.StateContext--参数传递时的状态快照

表示在状态机执行的各个阶段中使用的当前上下文。 例如，这些命令包括“转换”，“操作”和“警卫”命令，以访问事件标头和ExtendedState。
上下文实际上不是状态机的当前状态，而更像是**将上下文传递给各种方法时状态机所在位置的快照**。

## 2.StatemachineConfigurer状态机配置类

### 2.1头部注解使用

### *@EnableStateMachine*和*@EnableStateMachineFactory*。

我们使用熟悉的spring *enabler*注释来简化配置。存在两个注释，

***@EnableStateMachine*和*@EnableStateMachineFactory*。**

如果将这些注释放在*@Configuration*类中，则将启用状态机所需的一些基本功能。

*@EnableStateMachine*在配置要创建 *StateMachine*的实例时使用。通常， *@* Configuration类扩展了适配器 `EnumStateMachineConfigurerAdapter`或`StateMachineConfigurerAdapter`允许用户覆盖配置回调方法。我们会自动检测用户是否正在使用这些适配器类，并修改运行时配置逻辑。

*@EnableStateMachineFactory*在配置要创建 *StateMachineFactory*的实例时使用。

## 2.2配置状态



### 2.2.1配置**初态、终态、状态集合、状态动作**

```java
//配置初态--lock状态
//加载所有状态到状态机
//可以同时配置状态进入、退出时的动作--api 11.7
//state()如果状态机在初始状态和非初始状态之间来回转换，则执行用定义的动作。
@Override
public void configure(StateMachineStateConfigurer<CreditState, CreditEvent> states)
        throws Exception {
    states
            .withStates()
            // 初始状态：合作方推标
            .initial(CreditState.CREDIT_PARTNER_INPUT)
            //结束状态。。。。。。。。。。。。。。。。。。。。。。。。。。。。
            .end(CreditState.XXX)
            .states(EnumSet.allOf(CreditState.class));
}
```

### 2.2.2多状态机嵌套--配置分层状态、正交状态

流程嵌套

### 2.2.3配置为状态--暂时不用

初始化状态

历史状态

选择状态

加入状态

。。。。





## 2.3配置转换节点--包含action与guard配置

外部转换--状态之间的迁移.withExternal()

内部转换--单一源状态+事件.withInternal()

.withChoice() 当执行一个动作，可能导致多种结果时，可以选择使用choice+guard来跳转

```java
            .and()  // 使用and串联
            .withChoice() // 使用choice来做选择
            .source(WYD_INITIAL_JUMP) // 当前状态
             // 第一个分支
            .first(WAIT_REAL_NAME_AUTH, needNameAuthGurad(), needNameAuthAction)  
            .last(WAIT_BORROW, waitBorrowAction) // 第二个分支
```

- 注意，withChoice没有对应的event，所以依赖上一个节点的event，只要source是WYD_INITIAL_JUMP，就会自动执行当前choice transition
- withChoice没有对应的target，只有几个选择分支
- first/then/last 类似于 if--else if--else ，then可以不设置，但是last一定要有，否则会报错
- needNameAuthGurad() 用于判断是否走当前分支，返回true时选择当前分支，返回false走下面分支
- last中其实是省略了一个guard的实现，默认为true，意味着如果不走first分支，就会走last分支。
- withChoice其实是个PseudoState，也就是伪状态或瞬时状态，会马上跳转到下面的分支流程中。

> 本地转换--？？.withLocal()	



```java
@Configuration
@EnableStateMachine
public class Config3
		extends EnumStateMachineConfigurerAdapter<States, Events> {

	@Override
	public void configure(StateMachineStateConfigurer<States, Events> states)
			throws Exception {
		states
			.withStates()
				.initial(States.S1)
				.states(EnumSet.allOf(States.class));
	}

	@Override
	public void configure(StateMachineTransitionConfigurer<States, Events> transitions)
			throws Exception {
		transitions
			.withExternal()
				.source(States.S1).target(States.S2)
				.event(Events.E1)
				.and()
			.withInternal()
				.source(States.S2)
				.event(Events.E2)
				.and()
			.withLocal()
				.source(States.S2).target(States.S3)
				.event(Events.E3);
        
        
        .and()
                .withInternal()
                .source(States.S1)
			//.event(TurnstileEvents.COIN)
            //配置了event后不会执行States.S1的外部转换
            //不配置event相当于States.S1的状态监听
                .action(unlockInternal())
	}

}
```





### 2.3.1配置防护（守卫条件）

**保护状态转换--相当于屏蔽了event，action、transition不会继续执行了**

您可以使用防护来保护状态转换。您可以使用该`Guard`接口进行评估，其中方法可以访问`StateContext`。或者直接配置守卫条件是否

写法：和action相同或者直接配置.guardExpression("true");

```java
@Bean
public Guard<States, Events> guard1() {
	return new Guard<States, Events>() {

		@Override
		public boolean evaluate(StateContext<States, Events> context) {
			return true;
		}
	};
}

@Bean
public BaseGuard guard2() {
	return new BaseGuard();
}

public class BaseGuard implements Guard<States, Events> {

	@Override
	public boolean evaluate(StateContext<States, Events> context) {
		return false;
	}
}
```



### 2.3.2配置动作

您可以定义要通过转换和状态执行的动作。

##### transtion中配置动作：Transition Action

**动作总是作为源自触发器的转换的结果而运行的,完成transtion才执行**。

其中action可以多个插入，也就是有多少单独的业务需要在这里面处理都行

```java
.withExternal()

                    					     .source(ComplexFormStates.FULL_FORM).target(ComplexFormStates.CHECK_CHOICE)
                    .event(ComplexFormEvents.CHECK)
                    .action(new ComplexFormChoiceAction(),new ComplexFormChoiceAction())
                    .guard(new ComplexFormCheckChoiceGuard())
                    .and()
```



##### states中配置动作：

1、状态初始化动作

> 使用`initial()`功能定义动作仅在启动状态机或子状态时才运行特定动作。此操作是<u>仅运行一次的初始化操作。</u>`state()`如果状态机在初始状态和非初始状态之间来回转换，则运行用定义的操作。

2、状态进入/退出动作---含有两个参数

3、State Actions---含有一个参数---可以设置取消策略和超时取消时间（状态完成先于动作执行完成）

```java
//在StateMachineStateConfigurer中
.initial(States.S1, initialAction())
.state(States.S1, action(), null)
.state(States.S2, null, action())
.state(States.S3, action(), action());


.state(States.S2, action())
```

4.action写法---

匿名内部类

实现action接口的子类

spel表达式

```java
@Bean
public Action<States, Events> action1() {
	return new Action<States, Events>() {

		@Override
		public void execute(StateContext<States, Events> context) {
		}
	};
}

@Bean
public BaseAction action2() {
	return new BaseAction();
}

@Bean
public SpelAction action3() {
	ExpressionParser parser = new SpelExpressionParser();
	return new SpelAction(
			parser.parseExpression(
					"stateMachine.sendEvent(T(org.springframework.statemachine.docs.Events).E1)"));
}

public class BaseAction implements Action<States, Events> {

	@Override
	public void execute(StateContext<States, Events> context) {
	}
}

public class SpelAction extends SpelExpressionAction<States, Events> {

	public SpelAction(Expression expression) {
		super(expression);
	}
}
```





```java
//配置转换--状态迁移表：流式调用中包含action与guard
//转换方式-->原状态--目标状态--事件--动作--守卫条件
//unlock--push-->locked
//lock--coin-->unlocked

@Override
public void configure(StateMachineTransitionConfigurer<CreditState, CreditEvent> transitions)throws Exception {
  
        transitions
          .withExternal()         
             		.source(CreditState.CREDIT_PARTNER_INPUT)
          			.target(CreditState.CREDIT_CHECK_REBUILD)
               		.event(CreditEvent.REBUILD_CREDIT_SUCC)
          			.action(creditRebuildAction)
          			//守卫条件--执行方法
          			//直接配置结果--->.guardExpression("true");
          			.guard(guardTest())
                .and()
          
          .withExternal()
                .source(CreditState.CREDIT_CHECK_REBUILD)
          			.target(CreditState.CREDIT_AUDIT)
                .event(CreditEvent.AUDIT_CREDIT_SUCC).action(creditAuditAction)
        ;
    }
```



## 2.4配置状态机模型---一般不要去操作

**尽管有可能，定义定制模型通常不是人们想要的。但是，这是允许从外部访问此配置模型的中心概念。**

`StateMachineModelFactory`是一个钩子，可让您无需手动配置即可配置状态机模型。本质上，它是集成到配置模型中的第三方集成。您可以`StateMachineModelFactory`使用来加入配置模型`StateMachineModelConfigurer`。

## 2.5配置注意事项--spring bean机制对配置结果的影响

#### 普通方法直接返回结果

#### spring bean是单例，先影响结果，后面才会回调

When defining actions, guards, or any other references from a configuration, it pays to remember how Spring Framework works with beans. In the next example, we have defined a normal configuration with states `S1` and `S2` and four transitions between those. All transitions are guarded by either `guard1` or `guard2`. You must ensure that `guard1` is created as a real bean because it is annotated with `@Bean`, while `guard2` is not.

This means that event `E3` would get the `guard2` condition as `TRUE`, and `E4` would get the `guard2` condition as `FALSE`, because those are coming from plain method calls to those functions.

However, because `guard1` is defined as a `@Bean`, it is proxied by the Spring Framework. Thus, additional calls to its method result in only one instantiation of that instance. Event `E1` would first get the proxied instance with condition `TRUE`, while event `E2` would get the same instance with `TRUE` condition when the method call was defined with `FALSE`. This is not a Spring State Machine-specific behavior. Rather, it is how Spring Framework works with beans. The following example shows how this arrangement works:

```java
@Configuration
@EnableStateMachine
public class Config1
		extends StateMachineConfigurerAdapter<String, String> {

	@Override
	public void configure(StateMachineStateConfigurer<String, String> states)
			throws Exception {
		states
			.withStates()
				.initial("S1")
				.state("S2");
	}

	@Override
	public void configure(StateMachineTransitionConfigurer<String, String> transitions)
			throws Exception {
		transitions
			.withExternal()
				.source("S1").target("S2").event("E1").guard(guard1(true))
				.and()
			.withExternal()
				.source("S1").target("S2").event("E2").guard(guard1(false))
				.and()
			.withExternal()
				.source("S1").target("S2").event("E3").guard(guard2(true))
				.and()
			.withExternal()
				.source("S1").target("S2").event("E4").guard(guard2(false));
	}

	@Bean
	public Guard<String, String> guard1(final boolean value) {
		return new Guard<String, String>() {
			@Override
			public boolean evaluate(StateContext<String, String> context) {
				return value;
			}
		};
	}

	public Guard<String, String> guard2(final boolean value) {
		return new Guard<String, String>() {
			@Override
			public boolean evaluate(StateContext<String, String> context) {
				return value;
			}
		};
	}
}
```



## 3状态机ID--machineId--区分不同种状态机实例

***在运行时，`machineId`将不同流程状态机彼此区分开，不是单一流程的不同实例***（builder.beanFactory形式）

**`machineId`将不同状态机实例彼此区分开，含单一流程的不同实例**（EnableStateMachineFactory形式）

其他功能实际上没有任何重要的作用-例如，在跟踪日志或进行更深入的调试时。如果没有简单的方法来识别这些实例，那么拥有许多不同的机器实例将很快使开发人员迷失在翻译中。结果，我们添加了设置的选项 `machineId`。

用`StateMachine.getId()`获取

## 4.状态机工厂适配器

在配置类上面加上@EnableStateMachineFactory注解，

> 声明状态机注解是在编译时创建静态状态机
>
> 状态机工厂--运行时动态创建状态机
>
> 框架内部时动态创建的，声明状态机注解是将接口暴露给用户
>
> 配置类内容不变，仅加上状态机工厂注解即可
>
> 使用方式，注解注入即可



```java
public class Bean3 {

	@Autowired
	StateMachineFactory<States, Events> factory;

	void method() {
		StateMachine<States,Events> stateMachine = factory.getStateMachine();
		stateMachine.start();
	}
}
```



缺陷：与一个状态机相关的所有的动作和守卫条件共享一个工厂实例

不同的状态机调用同一个状态机工厂时注意处理

```java
//使用时用@Qualifier("secondStateMachineFactory")
@EnableStateMachineFactory(name = "secondStateMachineFactory")
```

状态机工厂适配器依赖spring上下文，限制了编译时的配置，对象是在spring容器加载后才有的

所以按官方注入写法编译会出现红线



## 5状态机构建者

灵活度更高，不依赖spring，同样可以使用适配器

```
StateMachine<String, String> buildMachine1() throws Exception {
	Builder<String, String> builder = StateMachineBuilder.builder();
	builder.configureStates()
		.withStates()
			.initial("S1")
			.end("SF")
			.states(new HashSet<String>(Arrays.asList("S1","S2","S3","S4")));
	return builder.build();
}
```



```
StateMachine<String, String> buildMachine2() throws Exception {
	Builder<String, String> builder = StateMachineBuilder.builder();
	builder.configureConfiguration()
		.withConfiguration()
			.autoStartup(false)
			.beanFactory(null)
			.taskExecutor(null)
			.taskScheduler(null)
			.listener(null);
	return builder.build();
}
```

## 6.延迟事件



## 7.使用范围---状态机仅在session域中作用



## 8.使用扩展状态-ExtendedState

`StateMachine`有一种称为的方法`getExtendedState()`。

它返回一个名为的接口`ExtendedState`，该接口可以访问扩展状态变量。

您可以通过状态机直接访问这些变量，也可以`StateContext`在操作或转换的回调期间访问这些变量 。

#### 应用场景：

用于记录流程流转过程中的变量用于进一步处理

#### 监听扩展状态中的变量变化

1.使用`StateMachineListener`中的`extendedStateChanged(key, value)`方法侦听回调

```java
public class ExtendedStateVariableListener
		extends StateMachineListenerAdapter<String, String> {

	@Override
	public void extendedStateChanged(Object key, Object value) {
		// do something with changed variable
	}
}
```

2.实现一个Spring Application上下文 `OnExtendedStateChanged`侦听器

```java
public class ExtendedStateVariableEventListener
		implements ApplicationListener<OnExtendedStateChanged> {

	@Override
	public void onApplicationEvent(OnExtendedStateChanged event) {
		// do something with changed variable
	}
}
```





## 9.StateContext--存储状态机当前状态的快照

#### 里面有许多方法与回调来给出状态机当前的状态

​				---->状态模式中context存储状态机当前状态

#### `StateContext`被传递给各种组件，例如 `Action`和`Guard`

可以用StateContext获取如下：

- The current `Message` or `Event` (or their `MessageHeaders`, if known).

- The state machine’s `Extended State`.

- The `StateMachine` itself.

- To possible state machine errors.

- To the current `Transition`, if applicable.

- The source state of the state machine.

- The target state of the state machine.

- The current `Stage`, as described in [Stages](https://docs.spring.io/spring-statemachine/docs/2.1.3.RELEASE/reference/#sm-statecontext-stage).

  当前事件

  扩展状态

  状态机自身

  状态机异常

  当前状态转换

  ​		----源状态

  ​		----目标状态

  当前状态

## 10.阶段--状态机生命周期中的一个点或者时间段

#### **监听器中就是监听这些时间点或者时间段**

`EVENT_NOT_ACCEPTED`, `EXTENDED_STATE_CHANGED`, `STATE_CHANGED`, `STATE_ENTRY`, `STATE_EXIT`, `STATEMACHINE_ERROR`, `STATEMACHINE_START`, `STATEMACHINE_STOP`, `TRANSITION`, `TRANSITION_START`, and `TRANSITION_END`.

## 11.转换触发--驱动状态机转换-send（message/event）

### 11.1`EventTrigger`事件触发器

两种不同的方式发送事件。

1.使用称为的状态机API方法发送类型安全事件 `sendEvent(E event)`。

2.通过使用`sendEvent(Message<E> message)` 带有自定义事件头的API方法发送包装在Spring消息`Message`中的事件。这使我们可以向事件添加任意的额外信息，**然后这些信息对于`StateContext`是可见的，在`StateContext`可以用get方法获取**。

**注意--消息头会一直存在，直到事件在状态机导致的状态流转彻底走完**

在spring statemachine里面，我们**把事件event塞到message的payload**里面，然后**把需要传递的业务数据（例子里面就是order对象）塞到header里面--本质是为了存储orderId订单号**。创建message用的是messagebuilder，看它的名字就知道是专门创建message的。

```java
@Autowired
StateMachine<States, Events> stateMachine;

void signalMachine() {
    //方式1--不带事件消息头
	stateMachine.sendEvent(Events.E1);

    //方式2--带事件消息头
	Message<Events> message = MessageBuilder
			.withPayload(Events.E2)
        	//可以多个setHeader连用,传递多个对象
        	//.setHeader("foo", order).setHeader("foo", order)
			.setHeader("order", order)
			.build();
	stateMachine.sendEvent(message);
}
```



### 11.2`TimerTrigger`--定时触发器

`TimerTrigger`需要**在没有任何用户交互的情况下自动触发某件事时**，此功能很有用。



**类似前端的定时器与延时器**---几种转换类型见前面的描述

配置定时器的状态**动作与监听**会来回执行

**两者均是在源状态为活动状态时才会触发**

`timer()`只要源状态不改变，一直执行，状态改变后停止

`timerOnce()`在**实际进入源状态时延迟后才触发，仅一次**

```java
	@Override
	public void configure(StateMachineTransitionConfigurer<String, String> transitions)
			throws Exception {
		transitions
			.withExternal()
				.source("S1").target("S2").event("E1")
				.and()
			.withExternal()
				.source("S1").target("S3").event("E2")
				.and()
			.withInternal()
				.source("S2")
				.action(timerAction())
				.timer(1000)//定时器，每(1000ms/1s)执行一次，总多次.
				.and()
			.withInternal()
				.source("S3")
				.action(timerAction())
				.timerOnce(1000);//延时器，(1000ms/1s)后执行一次，仅一次
	}
```

## 12.监听状态机事件

监听状态机正在发生什么，**对某些情况做出反应或获取日志详细信息**以进行调试。Spring Statemachine提供了用于添加侦听器的接口。然后，这些侦听器提供一个选项，以在发生各种状态更改，动作等时获取回调。

两种方案：

### listen to Spring application context events. 监听spring应用程序上下文事件

Application Context Events应用程序上下文事件

实现ApplicationListener<StateMachineEvent>接口

监听应用程序上下文事件类`OnTransitionStartEvent`，`OnTransitionEvent`，

`OnTransitionEndEvent`，`OnStateExitEvent`， `OnStateEntryEvent`，`OnStateChangedEvent`，

`OnStateMachineStart`，`OnStateMachineStop`，和其他扩展基本`StateMachineEvent`事件类 （把状态转换、变换均作为spring事件）。这些可以与Spring一起使用`ApplicationListener`。

#### 配置类中@bean单独配置

`StateMachine`通过发送上下文事件`StateMachineEventPublisher`。如果使用`@Configuration` 注释类，则会自动创建默认实现`@EnableStateMachine`。以下示例`StateMachineApplicationEventListener` 从`@Configuration`类中定义的bean中获取一个：

```java
public class StateMachineApplicationEventListener
		implements ApplicationListener<StateMachineEvent> {

	@Override
	public void onApplicationEvent(StateMachineEvent event) {
	}
}

@Configuration
public class ListenerConfig {

	@Bean
	public StateMachineApplicationEventListener contextListener() {
		return new StateMachineApplicationEventListener();
	}
}
```



#### builder配置方式

上下文事件也可以使用`@EnableStateMachine`来启用，和`StateMachine`状态机一起用构建器构建并注册为bean，

```java
@Configuration
@EnableStateMachine
public class ManualBuilderConfig {

	@Bean
	public StateMachine<String, String> stateMachine() throws Exception {

		Builder<String, String> builder = StateMachineBuilder.builder();
		builder.configureStates()
			.withStates()
				.initial("S1")
				.state("S2");
		builder.configureTransitions()
			.withExternal()
				.source("S1")
				.target("S2")
				.event("E1");
		return builder.build();
	}
}
```



### directly attach a listener to a state machine. 直接将侦听器附加到状态机

使用状态机监听器

继承并实现其中方法StateMachineListener/StateMachineListenerAdapter

其中`stateContext`监听方法可以访问各种 `StateContext`在不同阶段的变化。

```java
public class StateMachineEventListener
		extends StateMachineListenerAdapter<States, Events> {

	@Override
	public void stateChanged(State<States, Events> from, State<States, Events> to) {
	}

	@Override
	public void stateEntered(State<States, Events> state) {
	}

	@Override
	public void stateExited(State<States, Events> state) {
	}

	@Override
	public void transition(Transition<States, Events> transition) {
	}

	@Override
	public void transitionStarted(Transition<States, Events> transition) {
	}

	@Override
	public void transitionEnded(Transition<States, Events> transition) {
	}

	@Override
	public void stateMachineStarted(StateMachine<States, Events> stateMachine) {
	}

	@Override
	public void stateMachineStopped(StateMachine<States, Events> stateMachine) {
	}

	@Override
	public void eventNotAccepted(Message<Events> event) {
	}

	@Override
	public void extendedStateChanged(Object key, Object value) {
	}

	@Override
	public void stateMachineError(StateMachine<States, Events> stateMachine, Exception exception) {
	}

	@Override
	public void stateContext(StateContext<States, Events> stateContext) {
	}
}
```

定义自己的侦听器后，您可以使用`addStateListener`方法在状态机中注册它。

```java
public class Config7 {

	@Autowired
	StateMachine<States, Events> stateMachine;

	@Bean
	public StateMachineEventListener stateMachineEventListener() {
		StateMachineEventListener listener = new StateMachineEventListener();
		stateMachine.addStateListener(listener);
		return listener;
	}

}
```



### 局限性和问题

**Spring应用程序上下文并不是目前最快的事件总线**，因此建议您考虑一下状态机发送事件的速率。

**为了获得更好的性能，最好使用该 `StateMachineListener`接口。**

**可以将`contextEvents`标志与`@EnableStateMachine`和`@EnableStateMachineFactory`一起使用， 以禁用Spring应用程序上下文事件，采用使用状态机监听器StateMachineListener/StateMachineListenerAdapter的方式**

```java
@Configuration
@EnableStateMachine(contextEvents = false)
public class Config8
		extends EnumStateMachineConfigurerAdapter<States, Events> {
}

@Configuration
@EnableStateMachineFactory(contextEvents = false)
public class Config9
		extends EnumStateMachineConfigurerAdapter<States, Events> {
}
```



## 13.上下文整合集成

类似上面的形式，通过监听状态机的事件或使用具有状态和转换的动作来与状态机进行交互有点受限制，不便于与业务应用（响应或log记录）整合

先使用`@WithStateMachine`注释将状态机与现有**bean关联**。再将支持的注释添加到该**bean的方法**中。

还可以使用注释`name`字段从应用程序上下文附加任何其他状态机。

使用`id属性`能够更好的区分多状态机实例

```java
@WithStateMachine(name = "myMachineBeanName")
//@WithStateMachine(id = "myMachineId")
public class Bean2 {

	@OnTransition
	public void anyTransition() {
	}
}
```

将@WithStateMachine作为元注解进一步封装自定义注解

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@WithStateMachine(name = "myMachineBeanName")
public @interface WithMyBean {//注解特定name
}
```



### 启用整合集成

通过@EnableWithStateMachine注解开启@WithStateMachine注解的全部功能，

@EnableWithStateMachine将需要的配置已经导入spring上下文中

@EnableStateMachine` and `@EnableStateMachineFactory注解中@EnableWithStateMachine是元注解，所以无需再配置@EnableWithStateMachine或@WithStateMachine注解，但是如果是构建者方式配置状态机，需要再配置@EnableWithStateMachine或@WithStateMachine注解，如下

**如果未将机器创建为Bean，则需要设置 `BeanFactory`机器，如下例所示。否则，tge机器不会意识到调用您的`@WithStateMachine`方法的处理程序。**

```java
public static StateMachine<String, String> buildMachine(BeanFactory beanFactory) throws Exception {
	Builder<String, String> builder = StateMachineBuilder.builder();

	builder.configureConfiguration()
		.withConfiguration()
			.machineId("myMachineId")
			.beanFactory(beanFactory);

	builder.configureStates()
		.withStates()
			.initial("S1")
			.state("S2");

	builder.configureTransitions()
		.withExternal()
			.source("S1")
			.target("S2")
			.event("E1");

	return builder.build();
}

@WithStateMachine(id = "myMachineId")
static class Bean17 {

	@OnStateChanged
	public void onStateChanged() {
	}
}
```





### 监听器中的方法参数

参数根对象是`StateContext`。我们还在内部进行了一些调整，以便可以`StateContext`直接访问方法而无需通过上下文句柄，其中参数的数量和顺序无关紧要。

```java
@WithStateMachine
public class Bean3 {

  //直接使用StateContext对象
	@OnTransition
	public void anyTransition(StateContext<String, String> stateContext) {
	}
  
  //注解列举StateContext其中参数
  @OnTransition
	public void anyTransition(
			@EventHeaders Map<String, Object> headers,//全部事件头
			@EventHeader("myheader1") Object myheader1,//单个事件头
			@EventHeader(name = "myheader2", required = false) String myheader2,
			ExtendedState extendedState,
			StateMachine<String, String> stateMachine,
			Message<String> message,
			Exception e) {
	}
}
```



### 监听转换的注解

**注解：`@OnTransition`，`@OnTransitionStart`和`@OnTransitionEnd`**，用法相同



在此类注解中，可以使用属性`source`并`target`限定过渡。如果 `source`和`target`保留为空，则匹配任何过渡。

```java
@WithStateMachine
public class Bean5 {

	@OnTransition(source = "S1", target = "S2")
	public void fromS1ToS2() {
	}

	@OnTransition
	public void anyTransition() {
	}
}
```



**默认情况下，`@OnTransition`由于Java语言的限制，不能将注解与创建的状态枚举和事件枚举一起使用。因此，您需要使用字符串表示形式。**

**但是，如果要使用类型安全的注解，则可以创建一个新的注解并将其`@OnTransition`用作元注解。该用户级注释可以引用实际状态和事件枚举，并且框架尝试以相同方式匹配它们。**

```java
//枚举增强注解
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@OnTransition
public @interface StatesOnTransition {

	States[] source() default {};

	States[] target() default {};
}
```

用法与元注解形式相同，用枚举代替状态字符串

```java
@WithStateMachine
public class Bean7 {

	@StatesOnTransition(source = States.S1, target = States.S2)
	public void fromS1ToS2() {
	}
}
```





### 状态注解

`@OnStateChanged`，`@OnStateEntry`，和 `@OnStateExit`。三者配置方法相同

```java
@WithStateMachine
public class Bean8 {

	@OnStateChanged
	public void anyStateChange() {
	}
  
  @OnStateChanged(source = "S1", target = "S2")
	public void stateChangeFromS1toS2() {
	}
  
  @OnStateEntry
	public void anyStateEntry() {
	}

	@OnStateExit
	public void anyStateExit() {
	}
}

```



同样，为了类型安全，需要通过使用`@OnStateChanged`元注释为枚举创建新的注解。

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@OnStateChanged
public @interface StatesOnStates {

	States[] source() default {};

	States[] target() default {};
}
```



```java
@WithStateMachine
public class Bean10 {

	@StatesOnStates(source = States.S1, target = States.S2)
	public void fromS1ToS2() {
	}
}
```





### 事件注解

有一个与事件相关的注解。它被命名为`@OnEventNotAccepted`。如果指定`event`属性，则可以侦听未被接受的特定事件。如果您未指定事件，则可以列出任何未被接受的事件。

```java
@WithStateMachine
public class Bean12 {

	@OnEventNotAccepted
	public void anyEventNotAccepted() {
	}

	@OnEventNotAccepted(event = "E1")
	public void e1EventNotAccepted() {
	}
}
```



### 状态机注解

：`@OnStateMachineStart`， `@OnStateMachineStop`，和`@OnStateMachineError`。

```java
@WithStateMachine
public class Bean13 {

	@OnStateMachineStart
	public void onStateMachineStart() {
	}

	@OnStateMachineStop
	public void onStateMachineStop() {
	}
  
  @OnStateMachineError
	public void onStateMachineError() {
	}
}
```



### 扩展状态注解

有一个与状态相关的扩展注解。它被命名为 `@OnExtendedStateChanged`。您还可以监听特定`key`对应的扩展状态更改。

```java
@WithStateMachine
public class Bean15 {

	@OnExtendedStateChanged
	public void anyStateChange() {
	}

	@OnExtendedStateChanged(key = "key1")
	public void key1Changed() {
	}
}
```





## 14.运用 `StateMachineAccessor`状态机存取器

`StateMachine`是与一个状态机进行通信的主要接口。有时，需要获取动态地可编程地访问状态机及其嵌套的状态机和区域的内部结构。`StateMachine`提供了`StateMachineAccessor`接口，对单独的状态机和区域实例进行访问

`StateMachineFunction`是一个简单的功能接口，可让您将`StateMachineAccess接口应用于状态机。

以下第一种是jdk7写法，第二种是jdk8的lambda表达式写法

该`doWithAllRegions`方法可以访问`Region`状态机中的所有实例。

```java
stateMachine.getStateMachineAccessor().doWithAllRegions(new StateMachineFunction<StateMachineAccess<String,String>>() {

	@Override
	public void apply(StateMachineAccess<String, String> function) {
		function.setRelay(stateMachine);
	}
});

stateMachine.getStateMachineAccessor()
	.doWithAllRegions(access -> access.setRelay(stateMachine));
```

该`doWithRegion`方法可以访问`Region`状态机中的单个实例。

```java
stateMachine.getStateMachineAccessor().doWithRegion(new StateMachineFunction<StateMachineAccess<String,String>>() {

	@Override
	public void apply(StateMachineAccess<String, String> function) {
		function.setRelay(stateMachine);
	}
});

stateMachine.getStateMachineAccessor()
	.doWithRegion(access -> access.setRelay(stateMachine));
```

该`withAllRegions`方法可以访问`Region`状态机中的所有实例。

```java
for (StateMachineAccess<String, String> access : stateMachine.getStateMachineAccessor().withAllRegions()) {
	access.setRelay(stateMachine);
}

stateMachine.getStateMachineAccessor().withAllRegions()
	.stream().forEach(access -> access.setRelay(stateMachine));
```

该`withRegion`方法可以访问`Region`状态机中的单个实例。

```java
stateMachine.getStateMachineAccessor()
	.withRegion().setRelay(stateMachine);
```



## 15.运用 `StateMachineInterceptor`状态机拦截器

`StateMachineInterceptor`接口与`StateMachineListener`接口都有状态监听作用，

用法功能上有区别：**可以使用拦截器来拦截和停止当前状态更改或更改其转换逻辑。**

使用一个`StateMachineInterceptorAdapter`适配器类来覆盖默认的no-op方法，而不是实现完整的接口。



您可以通过`StateMachineAccessor`来注册拦截器**（`StateMachineAccessor`提供的是对状态机和状态机内部区域的访问，而拦截器正是状态机内部接口）**。拦截器的概念是一个相对较深的内部特征，因此不会直接通过`StateMachine`接口公开。

```java
stateMachine.getStateMachineAccessor()
	.withRegion().addStateMachineInterceptor(new StateMachineInterceptor<String, String>() {

		@Override
		public Message<String> preEvent(Message<String> message, StateMachine<String, String> stateMachine) {
			return message;
		}

		@Override
		public StateContext<String, String> preTransition(StateContext<String, String> stateContext) {
			return stateContext;
		}

		@Override
		public void preStateChange(State<String, String> state, Message<String> message,
				Transition<String, String> transition, StateMachine<String, String> stateMachine) {
		}

		@Override
		public void preStateChange(State<String, String> state, Message<String> message,
				Transition<String, String> transition, StateMachine<String, String> stateMachine,
				StateMachine<String, String> rootStateMachine) {
		}

		@Override
		public StateContext<String, String> postTransition(StateContext<String, String> stateContext) {
			return stateContext;
		}

		@Override
		public void postStateChange(State<String, String> state, Message<String> message,
				Transition<String, String> transition, StateMachine<String, String> stateMachine) {
		}

		@Override
		public void postStateChange(State<String, String> state, Message<String> message,
				Transition<String, String> transition, StateMachine<String, String> stateMachine,
				StateMachine<String, String> rootStateMachine) {
		}

		@Override
		public Exception stateMachineError(StateMachine<String, String> stateMachine,
				Exception exception) {
			return exception;
		}
	});
```





## 16.状态机安全--权限控制

安全功能是在Spring Security](https://projects.spring.io/spring-security)的功能之上构建的 。当需要保护状态机的一部分执行和交互时，安全功能可以非常方便做到。

> 我们希望您对Spring Security非常熟悉，这意味着我们不会详细介绍整个安全框架的工作原理。有关此信息，您应该阅读Spring Security参考文档（可[在此处获得](https://spring.io/projects/spring-security#learn)）。



安全性的第一级防御是自然地保护事件，这些事件实际上驱动着状态机中将要发生的事情。然后，您可以为过渡和操作定义更细粒度的安全设置。这与授予员工访问建筑物的权限，然后访问建筑物中特定房间的权限，甚至允许打开和关闭特定房间中的灯光的能力平行。如果您信任用户，那么事件安全性可能就是您所需要的。如果不是，则需要应用更详细的安全性。

举例：根据角色定义状态变换的权限，是否可以流转到某些状态

### 略



## 17.状态机错误处理

如果状态机在状态转换逻辑期间检测到内部错误，则可能引发异常。在内部处理此异常之前，有机会进行拦截。

用`StateMachineInterceptor`来拦截错误，

```java
StateMachine<String, String> stateMachine;

void addInterceptor() {
	stateMachine.getStateMachineAccessor()
		.doWithRegion(new StateMachineFunction<StateMachineAccess<String, String>>() {

		@Override
		public void apply(StateMachineAccess<String, String> function) {
			function.addStateMachineInterceptor(
					new StateMachineInterceptorAdapter<String, String>() {
				@Override
				public Exception stateMachineError(StateMachine<String, String> stateMachine,
						Exception exception) {
					// return null indicating handled error
					return exception;
				}
			});
		}
	});

}
```

当检测到错误时，正常事件通知机制将执行。这使您可以使用`StateMachineListener`或Spring Application上下文事件侦听器。和监听一节内容完全相同

**状态机监听适配器**

```java
public class ErrorStateMachineListener
		extends StateMachineListenerAdapter<String, String> {

	@Override
	public void stateMachineError(StateMachine<String, String> stateMachine, Exception exception) {
		// do something with error
	}
}
```

**spring 应用监听上下文**

```java
public class GenericApplicationEventListener
		implements ApplicationListener<StateMachineEvent> {

  //方法方式一
	@Override
	public void onApplicationEvent(StateMachineEvent event) {
		if (event instanceof OnStateMachineError) {
			// do something with error
		}
	}
  
  //方法方式二
	@Override
	public void onApplicationEvent(OnStateMachineError event) {
		// do something with error
	}

}
```















## 18.运用 `StateMachineService`

StateMachineService是更高级别的实现，旨在提供更多用户级别的功能以简化正常的运行时操作，`StateMachineService`是一个接口，用于处理正在运行的状态机。

并具有用于**“获取”和“释放”状态机**的简单方法。

它有一个默认的实现，名为`DefaultStateMachineService`。



## 19.持久化状态机

以订单为例，把订单状态存到订单表里面，其他的业务信息也都有表保存，而状态机的主要作用其实是规范整个订单业务流程的状态和事件，所以状态机要不要保存真的不重要，我们只需要从订单表里面把状态取出来，知道当前是什么状态，然后伴随着业务继续流浪到下一个状态节点即可

**持久化的意义**----持久化不是本意，让状态机能够随时抓换到任意状态节点才是目的。在实际的企业开发中，不可能所有情况都是从头到尾的按状态流程来，会有很多意外，比如历史数据，故障重启后的遗留流程......，所以这种可以任意调节状态的才是我们需要的状态机。





```java
//见项目持久化配置类中
* 使用方法
* 存储状态机---stateMachineRuntimePersister.persist(stateMachine,order1.getId());
* 恢复状态机---stateMachineRuntimePersister.restore(stateMachine,order1.getId());
* 
* 恢复状态机即获取一个新的状态机实例，获取之前保存的状态机的状态，把新状态机的状态置为保存的状态
```



### 持久化位置

不选StateMachineListener中：**当侦听器通知状态更改时，状态更改已经发生。如果侦听器中的自定义持久方法无法更新外部存储库中的序列化状态，则状态机中的状态和外部存储库中的状态将处于不一致状态。**

选择[`StateMachineInterceptor`]：**使用状态机拦截器来尝试在状态机内状态更改期间将序列化状态保存到外部存储中。如果此拦截器回调失败，则可以停止状态更改尝试，然后可以手动处理此错误，而不是结束不一致的状态。**



### 持久化内容`StateMachineContext`非StateMachine

**存`StateMachineContext`获取StateMachine！！！只是持久化当前状态快照，恢复状态，不要求同一状态机！！！**

`StateMachine`中包含的对象太多并且包含对其他Spring上下文类的过多依赖。

`StateMachineContext` 是状态机的运行时表示形式（存储状态机当前状态的快照，可以从中获取`StateMachine`），可用`StateMachineContext`对象将状态机还原到特定的状态 。



持久化代码：https://segmentfault.com/a/1190000019578055



## 20.Spring Boot支持



## 21.监视状态机--StateMachineMonitor

`StateMachineMonitor`用来获取有关执行过渡和执行操作需要多长时间的更多信息。

```java
public class TestStateMachineMonitor extends AbstractStateMachineMonitor<String, String> {

	@Override
	public void transition(StateMachine<String, String> stateMachine, Transition<String, String> transition, long duration) {
	}

	@Override
	public void action(StateMachine<String, String> stateMachine, Action<String, String> action, long duration) {
	}
}
```

配置`StateMachineMonitor`，将其添加到状态机

```java
@Configuration
@EnableStateMachine
public class Config1 extends StateMachineConfigurerAdapter<String, String> {

	@Bean
	public StateMachineMonitor<String, String> stateMachineMonitor() {
		return new TestStateMachineMonitor();
	}
}
```


