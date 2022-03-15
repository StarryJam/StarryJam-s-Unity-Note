# Unity通用有限状态机的从零搭建手册（二）：框架搭建

### 三大基础类创建

上一章末尾梳理了一下几个最重要的类的职责，这一章我们将以此为基础构建一个框架

首先在Unity中创建FiniteStateMachine、State、Condition三个类

![image-20220227173502340](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20220227173502340.png)

因为我们要搭建的是通用的状态机，保证任何的类都可以适用而不需要针对不同的类重新编写，所以我们首先要做的就是给这三个类定义为模板类，并且去掉MonoBehaviour的继承，然后备注一下将要实现的功能：

```c#
//状态机类
public class FiniteStateMachine<T>
{
    //属性：
    	//现态
    	//状态列表
    
    //方法：
    	//现态的更新
    	//添加状态
    	//删除状态
    	//状态转移
}
```

```c#
//状态类
public abstract class State<T>
{
    //属性：
    	//转换条件
    
    //方法：
    	//状态行为
    	//转换条件判断
}
```

因为状态类是具体状态的抽象，所以加上抽象类关键字，条件类同。

```c#
//条件类
public abstract class Condition<T>
{
    //方法：
    	//条件判断
}
```



### 方法定义

大纲列好之后，我们逐个类分析完成方法和属性的定义，从最核心的状态机类开始。

**状态机类：**

状态机需要维护一个状态的列表，实现状态的添加、删除和更新，最简单的做法就是将状态类实例作为参数传进方法中完成对状态的操作，尽管这种方法在真正使用时可能会碰到诸多不便，但是我们大可不必在刚刚起步时就顾虑太多，更好的做法可以在功能完成之后再分析优化的方向。

```c#
public class FiniteStateMachine<T>
{
    public FiniteStateMachine() { }

    //状态列表
    private List<State<T>> _states = new List<State<T>>();

    //当前状态
    private State<T> _currentState;

    //逻辑帧
    public void Update()
    {
       //TODO 状态转移判断
        
       //TODO 更新当前状态（调用现态的持续动作）
    }

    /// <summary>
    /// 添加状态
    /// </summary>
    public void AddState(State<T> state)
    {
        //TODO
    }

    //切出当前状态，切入目标状态
    public bool ChangeState(State<T> targetState)
    {
        //TODO
    }
}
```

具体的方法内容由于其他的类还没有完成，暂时先留白。

**状态类：**

我们做状态机的目的是将对象的不同状态抽象出来，用具体的状态类来封装每个状态的行为、转换条件以及条件达成后的目标状态，所以在状态类中我们需要维护一个映射表，提供方法配置条件和次态，并且完成切入、切出和持续动作的方法：

```c#
public abstract class State<T>
{
    //状态主体
    public T Subject { get; set; }

    /// <summary>
    /// 转换条件和对应状态的映射表
    /// </summary>
    private Dictionary<Condition<T>, State<T>> conditionMap = new Dictionary<Condition<T>, State<T>>();

    //切入时动作
    public virtual void Enter() { }
    
    //持续行为
    public virtual void Update() { }
    
    //切出时动作
    public virtual void Leave() { }
    
    /// <summary>
    /// 添加转换条件
    /// </summary>
    public bool AddCondition(Condition<T> condition, State<T> state)
    {
        //TODO
    }

    /// <summary>
    /// 检查是否满足某个转换条件，返回对应目标状态，没有则返回空
    /// </summary>
    public State<T> TransitionCheck()
    {
        //TODO
    }
}
```

因为并不是所有状态都一定在切入、切出和持续都有特定的行为，所以三个行为方法声明为虚方法供子类选择性重写就可以了

**条件类：**

条件类的职责很简单，就是负责判断条件是否达成，返回一个布尔值。

```c#
public abstract class Condition<T>
{
	//条件判断
    public abstract bool ConditionCheck();
}
```



#### 引入状态主体

上面在定义三个类的时候其实忽略了一个问题，状态机的存在其实是为了帮助目标对象分离各个状态，无法脱离对象而单独存在的，所以我们应该要在状态机系统中引入目标对象，我称之为状态主体（Subject），我们在最开始定义三个类的时候留出了一个模板参数T，其实就是我们的状态主体类。引入这个概念之后，再结合上面已经声明好的一些方法，我们进一步完善一下这三个类：

**状态机类：**

```c#
 /// 状态机类
    /// </summary>
    /// <typeparam name="T">主体</typeparam>
    public class FiniteStateMachine<T>
    {
        public FiniteStateMachine() { }

        //状态机的主体
        public T Subject { get; set; }    
    
        //状态列表
        private List<State<T>> _states = new List<State<T>>();

        //当前状态
        private State<T> _currentState;

        /// <summary>
        /// 逻辑帧（状态更新、转移）
        /// </summary>
        public void Update()
        {
            var targetState = _currentState.TransitionCheck(Subject);
            if (targetState != null)
            {
                ChangeState(targetState);
            }
            
            _currentState.Update();
        }
      
        /// <summary>
        /// 添加状态
        /// </summary>
        public void AddState(State<T> state)
        {
            if (_states.Contains(state))
            {
                Debug.LogWarning($"尝试重复添加状态{state}");
                return;
            }
            
            _states.Add(state);
        }

        //切出当前状态，切入目标状态
        public void ChangeState(State<T> targetState)
        {
            //触发现态切出动作
            _currentState.Leave();
            //触发次态切入动作
            targetState.Enter();
            //将次态设置为现态
            _currentState = targetState;
        }
    }
```



**状态类：**

```c#
/// <summary>
/// 状态类
/// </summary>
/// <typeparam name="T">主体</typeparam>
public abstract class State<T>
{
    //状态主体
    public T Subject { get; set; }

    /// <summary>
    /// 转换条件和对应状态的映射表
    /// </summary>
    private Dictionary<Condition<T>, State<T>> conditionMap = new Dictionary<Condition<T>, State<T>>();

    //切入时动作
    public virtual void Enter() { }
    
    //持续行为
    public virtual void Update() { }
    
    //切出时动作
    public virtual void Leave() { }
    
    /// <summary>
    /// 添加转换条件
    /// </summary>
    public bool AddCondition(Condition<T> condition, State<T> state)
    {
        if (conditionMap.ContainsKey(condition))
        {
            Debug.LogWarning($"条件{condition}被重复添加，一个条件只能对应一种次态！");
            return false;
        }
        
        conditionMap.Add(condition, state);
        return true;
    }

    /// <summary>
    /// 检查是否满足转换条件
    /// </summary>
    public State<T> TransitionCheck()
    {
        foreach (var map in conditionMap)
        {
            if (map.Key.ConditionCheck(Subject))
            {
                return map.Value;
            }
        }

        return null;
    }
}
```

这边要说明的是由于状态是一个对象自身的属性，状态类理应可以对主体拥有绝对的控制权，因此我们的具体状态子类在三个动作中只要直接控制主体的行为即可例如Subject.Run()，为了方便调用在状态类里可以留一个主体的引用。



**条件类：**

同上，条件类也应该可以直接通过读取主体的属性来判断是否达成，因为条件类的职责非常简单所以我们就使用传参的方式完成这个步骤省去初始化的麻烦，并且这样做有利于未来只需要创建一个条件实例就可以完成所有同类型主体的条件判断优化内存。

```c#
/// <summary>
/// 条件类
/// </summary>
/// <typeparam name="T">主体</typeparam>
public abstract class Condition<T>
{
    public abstract bool ConditionCheck(T subject);
}
```



写到这里状态机的框架基本就完成了，下一章我们将在此基础上完成完整的基础功能并尝试简单地应用在Unity当中。
