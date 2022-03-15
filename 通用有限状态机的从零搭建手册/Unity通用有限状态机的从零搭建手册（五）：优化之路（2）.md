# Unity通用有限状态机的从零搭建手册（五）：优化之路（2）

### 引入C#反射机制

前面我们已经实现了用ID来代替新建实例的配置过程，而这一点的前提是我们可以通过ID在一个状态机中唯一确定一个状态（或条件）；而上次优化并没有完全解决矛盾而是将其转移到了工厂当中。那么我们是不是可以用相似的思路来彻底解决这一问题呢？我们可以发现状态类和状态ID其实也是一一对应的关系，那么一定有一种方式可以让程序自动化地生成实例，而不需要每次新增类型的时候人为地添加映射关系，我们遵循”只要是让人觉得麻烦的地方就一定有偷懒余地“的原则，将最麻烦的事情交给计算机来做。不过难道我们在写代码的时候连具体的类型都不需要知道就可以新建出一个实例来吗？利用C#的反射就可以做到。

反射的功能简单来说就是在运行的时候动态地访问元数据，如同在深海中使用声纳雷达通过反射探测周围环境一般，它可以事无巨细地获得一个类的完整的属性和行为，依靠这些信息我们可以动态地实例化一个对象，这可以大大提高程序的灵活性，降低耦合。C#的另一个非常好用的特性[Attribute]的使用也依赖反射机制，也是我在项目中很爱用的一个特性，不过这里就不多做介绍了。

我们知道了借助反射机制我们可以动态地获得一个类地信息，那要如何让程序确定目标类型呢？我们只需要指定一套ID和类型名的对应规则，起名的时候严格遵守这个规则，就可以让程序在拿到一个ID的时候唯一确定对应的类，比如：枚举Cube_State.Default -> 类 Cube_State_Default，Condition类规则相同。

![image-20220304132256788](https://raw.githubusercontent.com/StarryJam/PicDock/main/imgimage-20220304132256788.png)

大致思路是我们首先需要一个类型工厂使用反射根据所给枚举值转为具体Type，然后状态工厂或条件工厂拿到具体类型之后我们有两种办法：一是用Activator.CreateInstance()方法动态调用构造函数实例化出具体的类型，二是用Type.GetConstructor()或Type.GetConstructors()获取构造函数之后调用；这里选用第二种方法，原因是我们后面在优化的时候会推荐将构造函数改为私有，第一种方法无法调用非共构造函数完成实例化。

```c#
//枚举转类型工厂
public abstract class Enum2TypeFactory
{
    //根据所给枚举返回完整类型名
    public static Type GetType(Enum typeEnum) 
    {
        //TODO 通过反射获取类型

        return type;
    }
}
```

```c#
//状态工厂
public class StateFactory
{
    public static State<T> GetState<T>(object stateMachine, Enum stateID)
    {
        Type stateType = Enum2TypeFactory.GetType(stateID);

        ConstructorInfo ctor = stateType.GetConstructors(BindingFlags.Instance | BindingFlags.Public | BindingFlags.NonPublic)[0];

        return (State<T>)ctor.Invoke(new object[] { stateMachine, stateID });
    }
}
```

```c#
//条件工厂
public class ConditionFactory
{
    public static Condition<T> GetCondition<T>(Enum conditionID)
    {
        Type conditionType = Enum2TypeFactory.GetType(conditionID);

        ConstructorInfo ctor = stateType.GetConstructors(BindingFlags.Instance | BindingFlags.Public | BindingFlags.NonPublic)[0];

        return (Condition<T>)ctor.Invoke(new object[] { conditionID });
    }
}
```

原则上我们的条件和方法类只有一个构造函数所以我们直接用GetConstrucors获取所有构造函数然后取第一个就行了，用GetConstructor去精确查找的话会比较复杂。GetConstructors方法的第一个参数BindingFlags是用来筛选反射内容的，其中BindingFlags.Instance（筛选实例成员）和BindingFlags.Static（筛选静态成员）二者至少有一项，这里需要的构造函数是非静态的所以选择BindingFlags.Instance，后面的Public和NonPublic是筛选共有和非公有成员，这里都选上。

然后完成我们的工厂类：

```c#
//枚举转类型工厂
public abstract class Enum2TypeFactory
{
    //根据所给枚举返回完整类型名
    public static Type GetType(Enum typeEnum)
    {
        //获取给定枚举值的枚举类型名（例如输入：Cube_State.Default，得到"Cube_State"）
        string enumTypeName = typeEnum.GetType().Name;
        //获取给定枚举值名（例如输入：Cube_State.Default，得到"Defaulte"）
        string enumValueName = Enum.GetName(typeEnum.GetType(), typeEnum);
        //拼接成目标类的名字
        string targetClassName = $"{enumName}_{enumValueName)}";
        Type type = Type.GetType(targetClassName);
        
        if (type == null)
            Debug.LogError($"枚举类型[{typeEnum}]没有找到对应的类，请检查");

        return type;
    }
}
```

这边用的都是最基础的反射方法直接看代码和下面贴的官方注释就OK了。

![image-20220306114602350](https://raw.githubusercontent.com/StarryJam/PicDock/main/imgimage-20220306114602350.png)

其实我几次反射的使用经验来看更重要的是想清楚需要动态获取的内容是什么，找到哪个流程是可以通过规则去做自动映射的，哪个步骤限制了项目的生产力，剩下的其实只是体力活而已，反射的使用并没有太多技巧性。

由于我们使用的是动态的实例化，我们可以比较方便地将类型转换放在工厂里完成，进一步封装不安全的代码，状态机里的可以直接用强类型的安全写法了：

![image-20220306143904306](https://raw.githubusercontent.com/StarryJam/PicDock/main/imgimage-20220306143904306.png)

测试之后依旧可以完美运行：

![cubeTest1](https://raw.githubusercontent.com/StarryJam/PicDock/main/imgcubeTest1.gif)



补充一点，因为反射是直接从元数据获取信息的，它并不受到访问权限的限制，所以这些状态类和条件类我们可以将其本身和他们的构造函数访问权限改为private，以免在代码中被意外实例化，顺便让代码补全的条目减少一些。

到此我们就完成了状态（条件）工厂的自动化升级。

最后要说的是，尽管反射可以让代码变得灵活，但是一定要注意不要在对运行效率影响很大的地方使用反射，也不要随意滥用反射，它不但会降低代码的运行效率，而且他会破坏代码的封装性，逃过编译器的检查，让代码更加难以理解且更可能出错。

