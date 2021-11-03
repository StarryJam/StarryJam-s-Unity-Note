## Coroutine的打开和关闭

#### 实验准备

一共有三种协程的打开方式和三种协程的关闭方式，三种方式形式上一一对应，下文就将他们称为a,b,c和α,β,γ：

```c
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class CoroutineTest : MonoBehaviour
{
    //记录打开的协程的ID
    private int CoInt = 0;
    //帧计数
    private int frameCount = 0;

    private IEnumerator routine;
    
    private IEnumerator routine1;
    private IEnumerator routine2;
    
    void Start()
    {
        //a 直接传入方法名
        //StartCoroutine("TestCoroutine");
        //b 传入方法函数的返回值
        //StartCoroutine(TestCoroutine());
        //c 传入保存的IEnumerator
        //routine = TestCoroutine();
        //StartCoroutine(routine);

        //α 用方法名关闭协程
        //StopCoroutine("TestCoroutine");
        //β 用方法的返回值关闭协程
        //StopCoroutine(TestCoroutine());
        //γ 用保存的IEnumerator关闭协程
        //StopCoroutine(routine);
    }

    void Update()
    {
        frameCount++;
    }


    IEnumerator TestCoroutine()
    {
        //ID++
        int coInt = this.CoInt++;
        
        Debug.LogWarning($"Coroutine{coInt} Started.");
        
        //总共进行五次循环
        int step = 0;
        while (step < 5)
        {
            step++;
            Debug.Log($"CoInt: {coInt} ==== step : {step}, frameCount: {frameCount}");
            yield return null;
        }
        Debug.LogWarning($"Coroutine{coInt} Finished.");
    }
}
```

协程输出的结果

![](https://raw.githubusercontent.com/StarryJam/PicDock/main/imgavatar.png)

**注：**

1. 还有一种直接传入Coroutine的stop方式由于比较好理解这里就不做实验了；

2. StopAllCorotine方法可以直接停掉对应Monobehavior发起的全部协程同样比较简单粗暴这里也不涉及了



#### 实验

##### 实验一：打开协程

一共尝试三种方式打开协程

方法a直接传入方法名；方法b传入方法函数的返回值；方法c则是用变量将方法返回的枚举器保存，然后将变量传入。结果：三种方式都成功打开协程



##### 实验二：关闭协程

对应的三种关闭和打开协程的方法

```c#
//a
StartCoroutine("TestCoroutine");
//b
StartCoroutine(TestCoroutine());
//c
routine = TestCoroutine();
StartCoroutine(routine);

//α
StopCoroutine("TestCoroutine");
//β
StopCoroutine(TestCoroutine());
//γ
StopCoroutine(routine);
```

结果：下面是三种打开方式和关闭方式的组合能否关闭单个协程的表格

|      |  a   |  b   |  c   |
| :--: | :--: | :--: | :--: |
|  α   |  √   |  ×   |  ×   |
|  β   |  ×   |  ×   |  ×   |
|  γ   |  ×   |  ×   |  √   |



##### 实验三：多个相同协程的打开

代码：

```c#
//a组
StartCoroutine("TestCoroutine");
StartCoroutine("TestCoroutine");

//b组
StartCoroutine(TestCoroutine());
StartCoroutine(TestCoroutine());

//c1组
routine = TestCoroutine();
StartCoroutine(routine);
StartCoroutine(routine);

//c2组
routine1 = TestCoroutine();
routine2 = TestCoroutine();
StartCoroutine(routine1);
StartCoroutine(routine2);
```

**注：**

这里的c指的是只使用用一个变量去保存方法传出的IEnumerator，代码如下。最后结论的时候会提到多个变量分别保存返回值的用法。



结果：

|  a   |  b   |  c1  |  c2  |
| :--: | :--: | :--: | :--: |
|  √   |  √   |  ×   |  √   |

值得注意的是上面c1的用法不但无法打开多个协程，还会导致一个协程在一帧循环里被执行多次：

![](https://raw.githubusercontent.com/StarryJam/PicDock/main/imgavatar.png)

调用两次c的情况下三帧内就完成了五次循环



##### 实验四：多个相同协程的关闭

由于b没有方法可以停止，c1则无法正确打开多个协程，所以只需要测试a和c2就可以了

代码：

```c#
//a-α
StartCoroutine("TestCoroutine");
StartCoroutine("TestCoroutine");
StopCoroutine("TestCoroutine");

//c2-γ2
routine = TestCoroutine();
routine2 = TestCoroutine();
StartCoroutine(routine1);
StartCoroutine(routine2);
StopCoroutine(routine1);
StopCoroutine(routine2);
```



结果：调用一次α就会停止全部用a开启的协程，c2-γ2则可以如预期关闭对应的协程



#### 结论

单纯的开启单个协程的话a，b，c三种方法都OK；

如果说需要控制单个协程的关闭的话用a-α或c-γ的组合可以完成；

多个相同协程的打开用a，c可以完成

如果涉及多个相同协程的关闭，α可以一次关闭所有由a打开的相同协程，c-γ的组合则需要多个变量来完成。
