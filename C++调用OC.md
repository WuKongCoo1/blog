# 前言
　　最近项目中为了方便维护，底层统一使用C++编写。由于是项目是做屏幕共享sdk，所以只能底层的压缩、编解码使用C++，屏幕捕获部分Mac和win就自己实现了。那么问题就来了，因为是面向接口编程，所以项目的入口都是c++来写的，而屏幕捕获是需要oc部分的代码，就需要C++调用oc代码了。

# 准备
之前只做过OC调动C++，于是Google了一下，在Stack Overflow上找到了这个[回答](https://stackoverflow.com/questions/1061005/calling-objective-c-method-from-c-member-function)。要看具体描述的可以去链接看看，实现思路一共有两种，我在这里大概描述一下。第一种，由于C++是不能直接调用OC的，所以需要通过C语言作为中间层，即C++调用C，C调用OC，这样就达到了C++调用OC的目的。第二种OC是可以调用C++的，通过在外部声名C++类，然后类具体实现放在OC类中，这样C++类就能够调用OC类了，其他需要调用OC的类，只需要调用外部声名的类即可。

# 实现
具体的实现方式有两种，第一种是C语言方法接收oc对象指针和参数，然后把指针桥接为具体的oc对象。第二种是用C++进行包装，先声名一个C++类，这里称为A。然后在OC类中，这里称为B，对A进行实现，因为这个实现实在OC语言里的，所以在这里是可以直接调用OC代码的。接下来声名一个C++类C。类C通过持有类A来调用OC类B，即A（C++）->C（C++）->B(OC类)

## 实现方式一 by C

**MyObject-C-Interface.h**

```
int MyObjectDoSomethingWith (void *myObjectInstance, void *parameter);
```

**MyObject.h**

```
@interface MyObject : NSObject
{
    int someVar;
}

- (int)doSomethingWith:(void *)aParameter;
@end

```

**MyObject.mm**

```
@implementation MyObject

int MyObjectDoSomethingWith (void *self, void *aParameter)
{
    // 通过将self指针桥接为oc 对象来调用oc方法
    return [(__bridge id)self doSomethingWith:aParameter];
}

- (int) doSomethingWith:(void *) aParameter
{
    //将void *指针强转为对应的类型
    int* param = (int *)aParameter;
    return *param / 2 ;
}

- (void)dealloc
{
    NSLog(@"%s", __func__);
}

@end
```

**MyCPPClass.h**

```
class MyCPPClass {
    
    
public:
    MyCPPClass();
    ~MyCPPClass();
    int someMethod (void *objectiveCObject, void *aParameter);
    void *self;
    
    void setSelf(void *self);
};

```

**MyCPPClass.cpp**

```
#include "MyObject-C-Interface.h"

MyCPPClass::MyCPPClass()
{
    
}

MyCPPClass::~MyCPPClass()
{
    
}


int MyCPPClass::someMethod (void *objectiveCObject, void *aParameter)
{
    // To invoke an Objective-C method from C++, use
    // the C trampoline function
    return MyObjectDoSomethingWith (objectiveCObject, aParameter);
}

void MyCPPClass::setSelf(void *aSelf)
{
    self = aSelf;
}

```
**main.mm**

```
#include "MyCPPClass.hpp"
#import "MyObject.h"

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // insert code here...
        NSLog(@"Hello, World!");
        MyObject *object = [[MyObject alloc] init];
        MyCPPClass *c = new MyCPPClass();
        c->setSelf((__bridge void *)object);
        int a = 12;
        int result = c->someMethod((__bridge void *)object, &a);
        NSLog(@"%d", result);
    }
    return 0;
}
```
运行结果如下：
/Users/lineworks/Desktop/写作相关/blog/C++调用OC.md
![](/Users/lineworks/Desktop/屏幕快照 2019-06-26 下午2.57.24.png)
![](./../../../屏幕快照 2019-06-26 下午2.57.24.png)




