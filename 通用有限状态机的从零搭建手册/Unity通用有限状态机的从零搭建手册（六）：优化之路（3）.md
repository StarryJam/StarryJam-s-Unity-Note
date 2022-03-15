# Unity通用有限状态机的从零搭建手册（六）：优化之路（3）

经过了上两章的努力，状态机的结构已经有不错的优化，我们加入了工厂模式，大幅精简了繁琐的配置过程结，然后又使用了反射机制，进一步给工厂做了一次“工业革命”，完成了状态的自动化生产。而这一章我们将针对用法上再进行一次优化。

### 获得主体的完全控制权

之前的小方块Demo中我们的状态机看似已经比较完善，但是其实忽略了实际在投入使用过程中会遇到的一些问题，我们来看Cube类中的成员变量和属性：

![image-20220306150217589](https://raw.githubusercontent.com/StarryJam/PicDock/main/imgimage-20220306150217589.png)

为了让状态类可以控制主体，条件类可以读取主体的一些信息，我们将他们的访问权限设成了public，而真正的项目中，一个类很多的成员是不应该暴露给外部的，假如说我不希望Cube类的这些成员被外部所访问和控制，将权限设为private或protected，势必导致状态类和条件类同样无法访问这些成员，而C#也没有友元类，这时候我们就需要用到新的技巧：内部类。

内部类即定义在一个类内部的类，他可以访问外部类的私有成员（包括静态），而对于外部类来说对于内部类的访问权限和普通类没有太大区别，私有成员依旧是被保护的；内部类也可以设置访问等级，使得除了外部类以外其他的类无法直接访问它。下图是内部类和外部类的一些访问权限的关系：

![image-20220306234958015](https://raw.githubusercontent.com/StarryJam/PicDock/main/imgimage-20220306234958015.png)

内部类他的作用之一便是将外部类的某一状态下的行为进行封装，是不是和我们的状态类职责不谋而合？只要我们将状态和条件定义为主体的内部类，就可以获得主体的完全控制权，同时也保证了主体成员对其他类的封装性。当然直接写在同一个文件里会显得非常累赘，我们只需要用分布类的方式做处理就可以保证代码的整洁美观：

```c#
//分布类关键词partial
//Cube.cs
public partial class Cube : MonoBehaviour
{
    private FiniteStateMachine<Cube> _stateMachine;

    //访问权限改为私有
    //获取材质
    private Material material => GetComponent<MeshRenderer>().material;
    //鼠标是否悬浮
    private bool mouseOver = false;
    //鼠标是否按下
    private bool mouseDown = false;
    
    /*
    ...
    */
}
```

```c#
//分布类
//Cube_State_Default.cs
public partial class Cube
{
    //默认状态
    private class Cube_State_Default : State<Cube>
    {
        public Cube_State_Default(FiniteStateMachine<Cube> stateMachine, Enum stateID) : base(stateMachine, stateID) { }
    
        public override void Enter()
        {
            base.Enter();
            //外部类的私有成员依然可以访问
            Subject.material.color = Color.white;
        }
    }
}
```

用相同的方法修改其他的状态类和条件类即可。

### 新的规则与工厂类更新

将状态和条件都改成内部类运行之后，却发出了报错：

![image-20220307101303287](https://raw.githubusercontent.com/StarryJam/PicDock/main/imgimage-20220307101303287.png)

可以推测是因为改为内部类之后类名在元数据中有了变化，我们输出一下内部类的名字看一下发生了什么变化：

![image-20220307102904367](https://raw.githubusercontent.com/StarryJam/PicDock/main/imgimage-20220307102904367.png)

原本的类名被加上了外部类名和一个加号，我们只要更新一下枚举转类型的工厂类，将枚举类型名去掉后缀作为外部类名连同加号拼接到目标类名前：

```c#
//枚举转类型工厂
    public abstract class Enum2TypeFactory
    {
        //根据所给枚举返回完整类型名
        public static Type GetType(Enum typeEnum)
        {
            var enumName = typeEnum.GetType().Name;

            //去掉后缀得到外部类名
            var outerClassName = "";
            var words = enumName.Split('_');
            outerClassName += words[0];
            for (int i = 1; i < words.Length - 1; i++)
            {
                outerClassName += '_' + words[i];
            }
            
            //拼接
            string targetClassName = $"{outerClassName}+{enumName}_{Enum.GetName(typeEnum.GetType(), typeEnum)}";
            Type type = Type.GetType(targetClassName);

            if (type == null)
                Debug.LogError($"枚举类型[{typeEnum}]没有找到对应的类，请检查");

            return type;
        }
    }
```

当然这么做之后也意味着我们今后的状态类和条件类必须定义为主体的内部类，否则无法正确得到类名，当然这样的作法本身也符合状态机的设计初衷。



### 文件结构和调试小优化

一般来说一个主体状态和条件类数量都不会太少，凌乱的放置肯定不便于管理和查找文件，我的做法是在主体类的文件夹里建一个名为FSM的文件夹和名为State和Condition的子文件夹分别存放这个主体的状态和条件类。

![image-20220307160847601](https://raw.githubusercontent.com/StarryJam/PicDock/main/imgimage-20220307160847601.png)

大家也可以用自己习惯的方式存放这些文件，但是尽量让文件结构清晰易懂，方便日后项目的推进。



最后是针对调试的一个优化，为了方便日后团队输出日志或者断点查看，我们最好可以保证每一个枚举值的整数值都是唯一的，我在项目中给每一个主体预留了一千个枚举值，例如Cube的状态从1000开始到1999结束，1000一般习惯作为None标记没有任何作用，后面每多一个主体类型编号千位就+1。为了尽可能自动化排序和方便查看对应关系，我创建了一个FSMIDRuleConfig类来配置ID排序的规则：

```c#
public class FSMIDRuleConfig
{
    //单个主体的状态、条件的预留数量
    public const int IDLimit = 1000;
    
    public enum SubjectType
    {
        None,
        Cube,
        AnotherType,
    }
}
```

响应地StateIDConfig和ConditionIDConfig也需要做一点点改动：

```c#
public enum Cube_State
{
    None = FSMIDRuleConfig.SubjectType.Cube * FSMIDRuleConfig.IDLimit,

    /// <summary>
    /// 默认状态
    /// </summary>
    Default,
    /// <summary>
    /// 鼠标悬浮状态
    /// </summary>
    MouseOver,
    /// <summary>
    /// 鼠标按下状态
    /// </summary>
    MouseDown
}

public enum AnotherType_State
{
    None = FSMIDRuleConfig.SubjectType.AnotherType * FSMIDRuleConfig.IDLimit,

    State1,
    State2,
}
```



OK经过三章的努力我们这一阶段的优化任务就全部完成啦！但这还不是终点，相反，重头戏才刚刚开始。