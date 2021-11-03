## Coroutine的浅析和简单实现

#### 前言

​		最近接了UI的活，需要配合美术把做好的UI接入到游戏里，考虑到未来一定会有UI动画的需求，我希望在设计UI架构的时候将简单的动画需求分离出来交给美术编辑，解放自己的生产力，于是做了一套简易的UI动画类。

​		实现了动画类之后我遇到了第二个棘手的问题：为了方便美术预览，一定不能让他运行游戏的时候才能看到动画的效果，那我的生产力是解放了，美术的工作反而会变得更麻烦，这是我不希望看到的，然而我的整套UI和动画都是基于协程实现的，可是Unity的协程只允许Monobehavier开启，而且编辑器模式下是没发启动的，所以我需要实现一个自己的协程类，可以在编辑器模式下运行。

先看最终实现的效果：

![UI动画](https://media.giphy.com/media/26c39506EWkBMVP8IO/giphy.gif)



​		要自己实现一个协程首先就要去了解一下Unity原生的协程的实现原理，在刚接触协程的时候就很奇怪为什么协程函数的返回值是IEnumerator，而且还用到yield return new WaitForSecond(1f)这种从没接触过的语法，不过那时候就只管能实现就行了，没有去深究原理，现在需要自己实现它，这些东西就绕不开了。



#### IEnumerator和yield retrun



​		首先需要了解一下写协程的时候函数返回的IEnumerator是什么。C#中有很多类都继承自IEnumerable（可枚举的）这个接口，只要继承它，这个类就可以用foreach语句遍历。这个接口只有一个接口方法：

```c#
public interface IEnumerable
{
  IEnumerator GetEnumerator();
}
```

​		该方法返回的就是在写协程函数的时候看到的IEnumerator（枚举器）。它的作用就是像游标一样挨个访问IEnumerable里的元素：

```c#
public interface IEnumerator
{
  //游标移动到下一个位置，如果已经移动到了最后一个元素后面，遍历结束，返回false
  bool MoveNext();

  //游标当前位置的元素
  object Current { get; }

  //重置游标到第一个元素之前
  void Reset();
}
```

​		上面我加入的备注只是IEnumerator抽象出来的期望用法，实际在实现的时候内部的操作可能并不是向外部描述的那样，为什么这么说在之后实现自己的等待类的时候会提到。

​		另一个需要解决的疑问就是yield return是什么。它是C#2里加入的语法，简单来说就是一种语法糖，只要函数用yield return来返回值编译器就会将函数里的内容包装成一个IEnumerator或者是IEnumerable对象，函数内的代码也会被分成代码块，在每次移动游标的时候执行”游标扫过的代码“。这里先贴一段代码看一下用yield return和普通方式实现遍历int数组并输出偶数项，两者有什么区别。

```c#
public class YieldTest : MonoBehaviour
{
    private List<int> nums = new List<int>() {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
    private void Start()
    {
        //使用方式一输出
        Debug.LogWarning("Function 1 Started");
        foreach (var num in GetEvenNumbers(nums))
        {
            Debug.Log(num);
        }
        Debug.LogWarning("Function 1 Ended");
        
        //使用方式二输出
        Debug.LogWarning("Function 2 Started");
        foreach (var num in GetEvenNumbersByYield(nums))
        {
            Debug.Log(num);
        }
        Debug.LogWarning("Function 2 Ended");
    }
    
    //方式一：直接返回生成的偶数数组
    IEnumerable GetEvenNumbers(IEnumerable<int> nums)
    {
        List<int> evenNums = new List<int>();
        foreach (var num in nums)
        {
            if(num % 2 == 0)
                evenNums.Add(num);
        }

        return evenNums;
    }

    //方式二：使用yield return
    IEnumerable GetEvenNumbersByYield(IEnumerable<int> nums)
    {
        foreach (var num in nums)
        {
            if (num % 2 == 0)
                yield return num;
        }
    }
}
```

​		可以看到最明显的区别就是方法一会先一股脑地先遍历一遍生成一个偶数数组，输出的时候再一次遍历了生成的偶数数组，总共经历了两次遍历，而使用yield return只经历了一次遍历就完成了输出偶数的工作。用它可以很轻松的实现代码重用和减少冗余的遍历来优化性能，由于它并不是我们今天的主角而且我目前掌握的知识有限，这里就不多介绍了，这部分之后另外开坑，上述的内容对于我们实现自己的协程来说已经足够了。



#### YieldInstruction

​		上文简单解释了IEnumerator和yield return的原理，接下来要解决的是yield return后面跟着的一系列WaitForXXX类是什么。很遗憾我看源码发现这些类本身并不实现什么方法，他们的基类YieldInstrution更是一个空类：

```c#
public sealed class WaitForSeconds : YieldInstruction
{
  internal float m_Seconds;

  public WaitForSeconds(float seconds) => this.m_Seconds = seconds;
}
```

```c#
/// <summary>
///   <para>Base class for all yield instructions.</para>
/// </summary>
public class YieldInstruction
{
}
```

​		不过好在Unity有一个CustomYieldInstruction的基类方便开发者实现自己的等待类，我们可以从它下手：

```c#
public abstract class CustomYieldInstruction : IEnumerator
{
  /// <summary>
  ///   <para>Indicates if coroutine should be kept suspended.</para>
  /// </summary>
  public abstract bool keepWaiting { get; }

  public object Current => (object) null;

  public bool MoveNext() => this.keepWaiting;

  public void Reset()
  {
  }
}
```

​		当时看到这里我基本上就通了，等待类被包装成一个枚举器，协程持有被返回的等待类时每一帧会调用MoveNext()方法，直到方法返回false遍结束等待，而MoveNext()只会返回属性keepWaiting的值，所以只要实现访问器keepWaiting就可以控制协程的等待了，于是我们可以根据这些重新实现一个自己的WaitForSeconds类：

```c#
public class StarryJamWaitForSeconds : CustomYieldInstruction
{
    private float m_tick = 0;

    public StarryJamWaitForSeconds(float duration)
    {
        m_tick = duration;
    }
    
    public override bool keepWaiting
    {
        get
        {
            if (m_tick <= 0)
                return false;

            m_tick -= Time.deltaTime;
            return true;
        }
    }
}
```

​		这就是为什么我上面说IEnumerator在真正实现的时候可以并不如向外部描述那般遍历元素，实际上它的MoveNext()函数的作用是触发状态的更新并返回是否满足退出条件，用IEnumerator来实现等待类有一个最大的好处就是实现了”形式上的统一“，即协程中yield return返回出来的对象都是IEnumerator（或者null），这样就可以用统一的方式处理协程和等待类之间的相互嵌套。



#### 实现自己的协程类

​		有了上面的铺垫之后，我们就可以着手实现自己的简易协程类了，首先我们协程类本身也需要是一个IEnumerator方便协程的嵌套，然后我们在创建一个协程的时候需要传入一个IEnumerator作为内容：

```c#
public class StarryJamCoroutine : IEnumerator
{
    //协程的内容
    private IEnumerator _originRoutine;

    public StarryJamCoroutine(IEnumerator routine)
    {
        _originRoutine = routine;
    }
    
    public bool MoveNext()
    {
        //TODO
    }

    public void Reset()
    {
        //TODO
    }

    public object Current { get; }
}
```

​		MoveNext()的函数体里要实现些什么呢，最简单的情况就是返回值并不是一个IEnumerator（一般来说是null），那我们什么都不需要做，再下一帧继续MoveNext()就可以继续协程，如果是IEnumerator会怎么样呢？考虑到普通函数就是一个递归的过程，协程出了不是同步之外形式上其实是一样的，那么很简单我们就用栈来实现递归遍历就行了，遇到新的IEnumerator就推入栈，栈顶遍历完就推出。

​		让我们给MoveNext()、Current和Reset()加入亿点点细节：

```c#
public class StarryJamCoroutine : IEnumerator
{
    private IEnumerator _originRoutine;
    private Stack<IEnumerator> _routineStack = new Stack<IEnumerator>();

    public StarryJamCoroutine(IEnumerator routine)
    {
        _originRoutine = routine;
        _routineStack.Push(routine);
    }
    
    public bool MoveNext()
    {
        //所有内容都执行完成则退出协程
        if (_routineStack.Count == 0)
            return false;
        
        var curStep = _routineStack.Peek();
        if (curStep.MoveNext())
        {
            if (curStep.Current is IEnumerator cur)
            {
                //如果返回值是IEnumerator就推入栈，并立即执行推入元素的MoveNext()
                _routineStack.Push(cur);
                return MoveNext();
            }

            return true;
        }
        else
        {
            //当前迭代器已经完成则弹出尝试走外层的迭代器
            _routineStack.Pop();
            return MoveNext();
        }
    }

    public void Reset()
    {
        _routineStack.Clear();
        _originRoutine.Reset();
        _routineStack.Push(_originRoutine);
    }

    public object Current => _routineStack.Peek().Current;
}
```

​		这样一个简易协程类基本上就实现了，因为上面提到的”形式上的统一“，我们甚至不需要对等待类做额外的处理，不得不说这是很妙的一个设计。至于为什么在推入新的嵌套协程时需要再一次调用MoveNext()也很好解释，因为我们在开启一个协程的时候并不期望它到下一帧开始才启动执行，而是立即开始，嵌套的协程也是如此。

​		最后我们只需要一个全局的协程管理器去每帧执行协程就可以了：

```c#
public class StarryJamCoroutineManager : MonoBehaviour
{
    public static StarryJamCoroutineManager Instance => _instance;
    private static StarryJamCoroutineManager _instance = null;

    private List<StarryJamCoroutine> _coroutines = new List<StarryJamCoroutine>();
    
    private void Awake()
    {
        _instance = this;
    }

    private void Update()
    {
        for (int i = 0; i < _coroutines.Count; i++)
        {
            if (!_coroutines[i].MoveNext())
            {
                _coroutines.RemoveAt(i);
                i--;
            }
        }
    }

    public void StartCoroutine(IEnumerator routine)
    {
        _coroutines.Add(new StarryJamCoroutine(routine));
    }
}
```



#### 最终效果

​		创建一个空物体挂载一个NumberCounter脚本，执行效果如下：

![112](https://raw.githubusercontent.com/StarryJam/PicDock/main/img112.png)

​		实际观察等待的时间好像有一点不稳定，但是总体实现了我们的协程功能，其他的细节日后可以优化~

​		

​		Demo项目在github同一层目录下有需要可以下载下来配合文章内容消化

