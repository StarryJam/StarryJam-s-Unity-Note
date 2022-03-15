# Unity通用有限状态机的从零搭建手册（七）：新的挑战

### 复合状态机

来试想一个问题：如果一个游戏角色有待机、移动、攻击、死亡四个状态，他们之间的转化关系应该是什么样的？

一般来说待机、移动、攻击这三者是可以互相转化的，而死亡状态则可以由这三种状态单项转化而来：

![image-20220307164354496](https://raw.githubusercontent.com/StarryJam/PicDock/main/imgimage-20220307164354496.png)

如果我们继续丰富角色的行为，比如施法、跳跃、下蹲，无论这些类的转化关系如何，无一例外地它们都会有一条指向死亡的箭头，这样的状态转化关系图势必变得十分繁琐，但如果我们把除了死亡之外的全部状态都包含于叫做存活的父状态中呢？

![image-20220307164939347](https://raw.githubusercontent.com/StarryJam/PicDock/main/imgimage-20220307164939347.png)

转化关系一下就被简化了，一条由存活指向死亡的箭头代替了所有子状态指向死亡的箭头，今后不管角色新增什么状态，只要它也属于存活状态的子状态，那他就不需要强调他和死亡状态的关系，而只需专注于与其他子状态的相互转化。这一章我们要做的，就是让我们的状态机支持这样的复合关系。

### 接口抽象

以存活状态为例，复合状态内部也如同一个状态机一样运作，这意味着它既有状态的行为（可以被状态机所使用）又有状态机的行为（可以对其配置状态），显然类的多继承在面向对象编程中是不被允许的（不了解的同学可以查一下菱形继承），那自然我们就要考虑用接口抽象来完成这一需求了。

我们将状态和状态机抽象成接口，然后原本的状态类和状态机类分别实现这两个接口，而我们的复合状态类同时实现两个接口：

![image-20220308180707710](https://raw.githubusercontent.com/StarryJam/PicDock/main/imgimage-20220308180707710.png)

那首先我们要做的就是将这两个接口抽象出来。

```c#
/// <summary>
/// 状态接口
/// </summary>
/// <typeparam name="T">主体类</typeparam>
public interface IState<T>
{
    /// <summary>
    /// 状态ID
    /// </summary>
    Enum StateID { get; }
    
    /// <summary>
    /// 状态主体
    /// </summary>
    T Subject { get; }
    
    /// <summary>
    /// 从属的状态机
    /// </summary>
    IStateMachine<T> StateMachine { get; }

    /// <summary>
    /// 切入动作
    /// </summary>
    void Enter();

    /// <summary>
    /// 持续动作
    /// </summary>
    void Update();

    /// <summary>
    /// 切出动作
    /// </summary>
    void Leave();

    /// <summary>
    /// 添加转换条件
    /// </summary>
    /// <param name="conditionID">条件ID</param>
    /// <param name="stateID">目标状态ID</param>
    /// <returns>this</returns>
    IState<T> AddCondition(Condition<T> conditionID, Enum stateID);

    /// <summary>
    /// 转换条件检查
    /// </summary>
    /// <returns>第一个满足条件的目标状态ID，没有则为null</returns>
    Enum TransitionCheck();
}
```

```c#
/// <summary>
/// 状态机接口
/// </summary>
/// <typeparam name="T">主体类</typeparam>
public interface IStateMachine<T>
{
    /// <summary>
    /// 状态主体
    /// </summary>
    T Subject { get; }

    /// <summary>
    /// 默认状态
    /// </summary>
    Enum DefaultStateID { get; set; }

    /// <summary>
    /// 添加状态（建议直接使用添加转换关系方法AddTransition()）
    /// </summary>
    /// <param name="state"></param>
    /// <returns></returns>
    IStateMachine<T> AddState(Enum state);

    /// <summary>
    /// 添加状态转换关系
    /// </summary>
    /// <param name="fromState">来源状态ID</param>
    /// <param name="toState">目标状态ID</param>
    /// <param name="condition">条件ID</param>
    /// <returns>this</returns>
    IStateMachine<T> AddTransition(Enum fromState, Enum toState, Enum condition);

    /// <summary>
    /// 变更状态
    /// </summary>
    /// <param name="targetState">目标状态ID</param>
    void ChangeState(Enum targetState);
}
```

基本上就是将原先两个类的共有方法和属性放到接口中，将（包括私有成员）之前所有的具体类依赖的参数和返回值改为接口依赖，；然后继承一下接口：

```c#
public class FiniteStateMachine<T> : IStateMachine<T>
{
	//...
}
```

```c#
public abstract class State<T> : IState<T>
{
	//...
}
```

完事之后创建一个新的类CompositeState类，先不要急着继承两个接口，我们来考虑一件事：复合状态类究竟是一个状态，还是一个状态机，或是两者都不是而是单纯有两者的行为？其实从名字就不难看出复合状态其实是一个**(is a)**状态，所有状态的方法和普通状态无二，但是它同时也有类似**(like a)**状态机的行为，最重要的是，它会以状态的身份从工厂类被生产出来；所以我们只需要把他定义为状态的子类，直接继承普通状态的方法，同时实现状态机接口，而状态机的行为我们可以通过给复合状态内置一个状态机来实现。所以最终的类图其实是这样的：

![image-20220310123206868](https://raw.githubusercontent.com/StarryJam/PicDock/main/imgimage-20220310123206868.png)



### 代码实现

思路理清之后代码部分没有什么难度，复合新状态本身允许和普通状态一样有自己的行为，子状态的行为则用内置状态机来管理，最大程度地复用代码：

```c#
/// <summary>
/// 复合状态类
/// </summary>
/// <typeparam name="T"></typeparam>
public abstract  class CompositeState<T> : State<T>, IStateMachine<T>
{
    protected CompositeState(IStateMachine<T> stateMachine, Enum stateID) : base(stateMachine, stateID)
    {
        _innerStateMachine = new FiniteStateMachine<T>(Subject);
    }
    
    //内置状态机
    private readonly FiniteStateMachine<T> _innerStateMachine;

    public Enum DefaultStateID
    {
        get => _innerStateMachine.DefaultStateID;
        set => _innerStateMachine.DefaultStateID = value;
    }

    public override void Awake()
    {
        base.Awake();
        _innerStateMachine.Awake();
    }

    public override void Enter()
    {
        base.Enter();
        _innerStateMachine.Start();
    }

    public override void Update()
    {
        base.Update();
        _innerStateMachine.Update();
    }

    public override void Leave()
    {
        base.Leave();
        _innerStateMachine.ChangeState(null);
    }

    public IStateMachine<T> AddState(Enum state)
    {
        _innerStateMachine.AddState(state);

        return this;
    }

    public IStateMachine<T> AddTransition(Enum fromState, Enum toState, Enum condition)
    {
        _innerStateMachine.AddTransition(fromState, toState, condition);

        return this;
    }

    public void ChangeState(Enum targetState)
    {
        _innerStateMachine.ChangeState(targetState);
    }
}
```

考虑到加入复合状态之后需要在代码中配置嵌套的状态机，我们给状态机接口一个Open方法像打开文件夹一样打开嵌套的子状态机：

```c#
/// <summary>
/// 状态机接口
/// </summary>
/// <typeparam name="T">主体类</typeparam>
public interface IStateMachine<T>
{
    /*
    ...
    */
    
    /// <summary>
    /// 打开复合状态的子状态机
    /// </summary>
    /// <param name="stateID">复合状态ID</param>
    /// <returns>子状态机</returns>
    IStateMachine<T> Open(Enum stateID);
    
    /*
    ...
    */
}
```

```c#
public class FiniteStateMachine<T> : IStateMachine<T>
{
    /*
    ...
	*/
    
    public IStateMachine<T> Open(Enum stateID)
    {
        var state = _GetState(stateID);
        try
        {
            var compState = (CompositeState<T>)state;

            return compState;
        }
        catch (Exception e)
        {
            Debug.LogError($"获取复合状态错误，ID：{stateID}");
            Debug.LogError($"错误类型：{e}");
            throw;
        }
    }

    /*
    ...
	*/
}
```

```c#
public class CompositeState<T> : State<T>, IStateMachine<T>
{
    /*
    ...
	*/
    
    public IStateMachine<T> Open(Enum stateID)
    {
        return _innerStateMachine.Open(stateID);
    }

    /*
    ...
	*/
}
```

完成复合状态之后我们的整个通用有限状态机的搭建就基本完工了，复合状态机的使用我们直接放到最后一章的综合案例中展现，作为整篇文章的结尾。下一章见！