*本系列志在持续更新一些游戏开发相关技术和 Unreal 引擎的原理知识讲解，虽然谈不上细究源码的级别，但已在力求讲清楚 UE 的系统设计以及底层运作模型。对于不甘于在 UE 功能应用层浅尝辄止、正在尝试学习 UE 底层的朋友来说，本系列主要参照本人学习 UE 时的视角去探讨引擎是如何工作的，相信能够在一定程度上帮助你们理解 UE 底层机制以及它是如何支撑应用层的。*

*欢迎阅读该系列文章，并分享自己的理解或提出文章中的模糊、错误的地方（不排除有）。*

*想要阅读系列中其他内容或想要持续关注本系列更新可移步：https://github.com/feiyangyang11/Unreal-Engine-Internals.git。*

------

# UE 的反射机制

## 反射做什么

用 `UCLASS / UPROPERTY / UFUNTION`修饰一个对象 / 函数时，反射系统通过 UHT 和 UBT 为其建立：

```cpp
ClassA              // 被 UCLASS 纳入反射体系的类
  ↓
UClass               // ClassA 的运行时类型描述**对象**
  │
  ├─ 类本身的信息
  │   ├─ Name	// 类名
  │   ├─ SuperClass / SuperStruct	// 父类名
  │   ├─ PropertiesSize		// 描述这个类型实例的反射属性布局所需要的总尺寸，而不是描述 UClass 自己
  │   ├─ ClassFlags		// 这个类具有哪些 UE 级别的属性/特征
  │   ├─ ClassCastFlags		// 和 Cast<> 优化有关
  │   ├─ Interfaces		// 这个类实现了哪些 UE Interface
  │   └─ ...
  │
  ├─ Property 信息      // 某个被 UPROPERTY 等纳入反射体系的字段
  │   ↓
  │  FProperty
  │   ├─ Name	// 变量名
  │   ├─ Type	// FProperty 的具体子类表示的类型，如 FIntProperty、FFloatProperty……
  │   ├─ Offset 	// 成员变量相对于所属对象/Struct 起始地址的偏移
  │   ├─ ElementSize	// 单个元素大小（变量大小 = ArrayDim * ElementSize）
  │   ├─ ArrayDim
  │   ├─ PropertyFlags
  │   ├─ Metadata	// 给编辑器、蓝图、工具链使用的附加描述信息，如 Category、Tooltip……
  │   └─ 类型特有信息
  │
  ├─ Function 信息      // 被 UFUNCTION 纳入反射体系的函数
  │   ↓
  │  UFunction
  │   ├─ Name
  │   ├─ FunctionFlags
  │   ├─ Parameters
  │   ├─ ReturnValue
  │   ├─ ParmsSize
  │   ├─ Native函数入口等
  │   └─ ...
  │
  └─ CDO		//  Class Default Object
```



## 引入反射的原因

引入 UE 反射的目的，根本上是为了解决编写 UE C++ 代码时带来的问题

原生 C++ 编译器在编译期知道类布局、成员偏移、函数签名等信息，并据此生成机器码；但这些源码级类型结构通常不会以“**可供程序运行时统一遍历查询**”的形式保留下来

而 UE 需要：

1. 在 UE Editor 的 details 面板显示类的类型名称、成员的类型和名称、是否可编辑……
2. 在 UE 蓝图中调用 C++ 函数节点，需要知道函数的名称、调用权限、参数类型和名称……
3. 将对象序列化后写入 Save Game / Asset / Package，需要根据对象类型及其他属性选择相应的序列化方式……
4. 在判断某个 UObject 对象是否要 GC 时，还要找到对象引用的其他 UObject 对象并判断是否要 GC……
5. 在网络复制和 RPC 时知道对象是否 replicated、是否 reliable、其他描述信息……
6. 运行时动态函数调用与动态创建对象

核心是：**把原本只有编译器知道的类型结构，以数据的形式保留下来，让程序运行时也能查询和操作**

## UHT

UHT = `UnrealHeaderTool`，是为 UObject 系统服务的自定义解析与代码生成工具

UBT 扫描到代码中声明`UCLASS / UPROPERTY / UFUNCTION / GENERATED_BODY()`的部分时，调度 UHT **解析它们并生成额外 C++ 代码**，如`MyClass.generated.h`、`MyClass.gen.cpp`。然后 C++ 原生编译器就能通过这些额外代码去实现反射能力

### 读取 UCLASS

UHT 读取 UCLASS 声明的代码时

比如：

```cpp
UCLASS(Blueprintable, Abstract)
class UMyClass : public UObject

// 解析 Class Name、Parent、Specifier，生成对应的 Uclass 类型，生成描述代码
// 然后在运行时根据描述代码生成如下：
/*

UClass
├─ Name = UMyClass
├─ SuperClass = UObject
├─ ClassFlags
├─ Properties
├─ Functions
├─ CDO
└─ ...

```

### 读取 UPROPERTY

UHT 读取 UPROPERTY 声明的代码时

比如：

```cpp
UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Combat")
float Health;

// 解析 Name、Type 、Specifier、Metadata，生成对应的 UPROPERTY 类型，生成描述代码
// 然后在运行时根据描述代码生成如下：
/*

FFloatProperty
│
├─ Name = Health
├─ Offset = ...
├─ ElementSize = 4
├─ PropertyFlags
└─ Metadata
```

### 读取 UFUNCTION

UHT 读取 UFUNCTION 声明的代码时

比如：

```cpp
UFUNCTION(BlueprintCallable)
void TakeDamage(int32 Damage);

// 解析 Name、Return 、Specifier、Parameter，生成对应的 UFUNCTION 类型，生成描述代码
// 然后在运行时根据描述代码生成如下：
/*

UFunction
│
├─ Name = TakeDamage
├─ FunctionFlags
│
├─ Parameters
│   └─ Damage
│       ↓
│     FIntProperty
│
└─ Native函数相关信息
```

### GENERATED_BODY()

把 UHT 为这个类型生成的一部分 C++ 声明真正插进 class 里面

UCLASS` / `USTRUCT 等声明和额外代码补充都需要一些基础设施，如：

- `StaticClass`相关支持
- `StaticRegisterNatives`
- 序列化声明
- 类型声明
- 友元声明
- 构造辅助
- ...

而`GENERATED_BODY()`的这些代码来自`#include "XXX.generated.h"`，UE 要求它**作为当前 Header 的最后一个 include**

这些代码在 C++ 编译的预处理阶段被替换到`GENERATED_BODY()`当前所在的位置

### .generated.h

包含需要进入对应头文件和自定义类中（如`MyClass`）的一些起**声明作用的一些代码**，如类型声明、friend声明、RPC声明、构造辅助……它们**必须出现在 class 内部**，包含如下辅助接口：

- `static UClass* StaticClass()`：得到对应类的 UClass* 对象
- `static void StaticRegisterNativesAPlayer()`：注册这个类中需要进入 UE 函数系统的 Native C++ 函数
- `friend` 生成函数声明：允许 UHT 生成的外部构建代码访问类内部必要信息
- `MyClass(const FObjectInitializer& ObjectInitializer)`：构造器相关辅助宏，让类对象能按照 UE 的对象构造流程创建
- `virtual void Serialize(FArchive& Ar) override`：序列化相关接口
- ……

###  .gen.cpp

 UHT 扫描`UCLASS / USTRUCT / UPROPERTY / UFUNTION`等之后生成

`XXX.gen.cpp`：包含**自定义类对应的反射构造/注册实现代码**、Property描述数据、Function描述数据、UClass构建数据、Native函数注册、Metadata、静态注册代码等，这些数据都是**静态常量类型**，只支持内部链接，例如：

```cpp
// 类成员函数 Fire 的描述信息
static const UECodeGen_Private::FIntPropertyParams NewProp_Damage = {
    "Damage",
    nullptr,
    /* Flags ... */
};
static const UECodeGen_Private::FFunctionParams FuncParams = {
    /* OwnerClass = */ APlayer::StaticClass,
    /* Name = */ "Fire",
    /* 参数列表 */
};
```

以及

```cpp
// 类成员属性 Health 的 Property 描述
static const UECodeGen_Private::FIntPropertyParams NewProp_Health = {
    "Health",
    nullptr,
    /* PropertyFlags ... */
    /* Offset ... */
};
```

等等

总之，.generated.h 把“必须存在于类内部”的声明注入进去，.gen.cpp 在类外部生成类的反射描述、构建、注册实现

### 构建过程

```
开始 Build
   ↓
UBT 启动
   ↓
UBT 调用 UHT
   ↓
UHT 扫描 .h

识别：UCLASS / UPROPERTY / UFUNCTION / GENERATED_BODY / 以及各种 Specifier / Metadata
   ↓
UHT 根据这些声明生成 C++ 代码
   │
   ├─ Player.generated.h
   │     ↓
   │   生成一些必须注入 APlayer 类内部的声明/辅助接口
   │
   └─ Player.gen.cpp
         ↓
       生成描述：
       “APlayer 是什么类”
       “父类是谁”
       “有哪些 Property”
       “有哪些 UFunction”
       “Flags 是什么”
       “如何构造/注册 APlayer 的 UClass”
       等描述数据和代码
   ↓
然后普通 C++ 编译阶段开始
   ↓
预处理器处理 Player.h
   ↓
#include "Player.generated.h"
已经把 UHT 生成的宏定义包含进来
   ↓
遇到 GENERATED_BODY()
   ↓
展开成 generated.h 中针对 APlayer 生成的类内声明
   ↓
编译器同时编译：你写的代码 + generated.h 注入的代码 + xxx.gen.cpp
   ↓
生成 DLL / EXE
```

## UBT

UBT = `Unreal Build Tool`，是为 UE 自己的总构建调度器，**负责把整个 UE C++ 构建流程串起来**

### C# 构建配置文件

**UBT** 运行时首先读取 `XXX.Build.cs、XXX.Target.cs、XXXEditor.Target.cs` 这三个配置文件

#### 什么是模块

UE 里，**Module 是一个独立的 C++ 编译单元/功能单元**。一个项目可以有一个模块，也可以有很多模块

比如一个项目可能编写了：

```cpp
MyGame
├─ MyGame                // 主游戏模块
├─ Combat                // 战斗模块
├─ Inventory             // 背包模块
└─ MyGameEditor          // 编辑器扩展模块
```

每个模块通常都有自己的`XXX.Build.cs`

UE 引擎自己内部也是由大量模块组成的，可以根据项目按需配置：

```
Core
CoreUObject
Engine
Slate
SlateCore
UMG
AIModule
NavigationSystem
Networking
OnlineSubsystem
...
```

我们自己的游戏程序也是一个模块。创建自己的新项目时，通常会默认添加一个`Source/MyGame/MyGame.Build.cs`模块

#### XXX.Build.cs

它描述的是：**这个“模块”怎么编译（依赖哪些模块？公开依赖？私有依赖？）**，包括第三方库、平台宏、PCH、异常等编译配置

比如：

```c#
PublicDependencyModuleNames.AddRange(
    new string[] { "Core", "CoreUObject", "Engine" }//当前模块编译需要依赖这些模块
);
```

#### XXX.Target.cs

它描述的是：**最终要构建一个什么类型的目标程序（一般是游戏程序），以及这个目标包含哪些模块**

如：

```c#
Type = TargetType.Game;
//表示要构建的目标是一个游戏程序
```

又如：

```c#
ExtraModuleNames.Add("MyGame");
//表示这个 Game Target 里要包含 MyGame 模块
//还有其他配置，如是否启用某些引擎功能、平台/构建目标相关选项
```

#### XXXEditor.Target.cs

它描述的也是：**最终要构建一个什么类型的目标程序（一般是编辑器程序），以及这个目标包含哪些模块**

它的配置内容也和`XXX.Target.cs`差不多，主要是为了构建 Unreal Editor 版本的目标程序

除了包含游戏模块，还可以加载很多编辑器插件、开发工具……

### 工作流程

```
UBT
↓
读取项目和模块配置
↓
判断哪些文件需要重新构建
↓
如果有反射代码，调 UHT
↓
UHT 生成 generated.h / gen.cpp
↓
UBT 再调用真正的 C++ 编译器 MSVC / Clang
↓
编译 .cpp / .gen.cpp
↓
链接成 DLL / EXE
↓
模块加载 / UE 类型注册阶段
↓
执行 gen.cpp 中生成的构建/注册代码
↓
最终运行时创建：UClass 对象（描述 APlayer 的反射信息）
```

## 运行时反射

运行时，每个 UObject 类会创建一个属于它的 UClass 实例对象（包含类的完整反射信息），该类所有 UObject 对象共享这个实例

### `UObject → UClass` 的运行时关联

UHT 已有 `UClass/FProperty/UFunction` 的构建信息，那么运行时创建一个 UObject 实例以后，这个实例怎么找到那份 UClass？

答案是，每个 UObject 都内部维护了一个私有的 UClass* 类型的成员变量指针，指向类的 UClass 实例对象，通过`obj->GetClass()`或`UObjectClass::StaticClass()`可获取到这个指针。简化代码类似：

①`obj->GetClass()` 返回的是对象本身类型对应的 UClass 对象指针，**支持运行时多态场景**

```cpp
class UObjectClass
{
public:
	UClass* GetClass();//普通函数，继承自父类
private:
    UClass* ClassPrivate;   
};
//函数实现
UClass* UObjectClass::GetClass() const
{
    return ClassPrivate;
}
```

②`UObjectClass::StaticClass()`是**根据类名调用，和类强绑定，编译期就确定了返回类型**，由 UHT 生成，在 `xxx.generated.h` 中

```cpp
class UObjectClass
{
public:
	static UClass* StaticClass();//类静态函数，它是被 UHT 在编译时把 .generated.h 塞进来的 
};
//函数实现，内部保存静态指针变量，只有第一次访问时初始化
static UClass* StaticClass(){
	static UClass* s_ClassCache = nullptr;
	if(s_ClassCache == nullptr){//这里的写法是懒加载，但实际 UE 源码中会在引擎启动阶段就做好这些
		s_ClassCache = new UClass();
		s_ClassCache->FillFromClassInfo(&UObjectClass_ClassInfo);//将 xxx.gen.cpp 中的反射信息全部填入对象内
		s_ClassCache->AddToRoot();//加入到 RootSet 避免被错误 GC
	}
    return s_ClassCache;
}
//new 对象时会保证构造这个 UClass* 对象
template<typename T>
T* NewObject(UObject* Outer){//Outer 是这个对象的逻辑拥有者的指针，两者类似于 Pawn 拥有 Camera 的这种关系
	UClass* CLs = T::StaticClass();
	T* Instance = AllocateUObject<T>(CLs,Outer,...);//分配内存并调用构造函数，CLs 在此被赋值给成员 ClassPrivate
	return Instance;
}
```

### `UPROPERTY` 在 `UClass` 里怎么组织

被声明为 UPROPERTY 的类成员属性，在类实例中和正常成员变量被使用，而在 UClass 中以链表形式组织属性信息

UClass 中有一个关键字段`FField* ChildProperties`，它指向 child fields 链表头节点：

```cpp
UClass(APlayer)
│
└─ ChildProperties//FFields 类型， FIntProperty、FFloatProperty 等都是它的子类
       ↓
   FIntProperty Health
       ↓
   FFloatProperty Speed
       ↓
   FObjectProperty Weapon
       ↓
      nullptr
```

如何遍历？

```cpp
UClass* Class = Player->GetClass();
for (TFieldIterator<FProperty> It(Class); It; ++It)
{
    FProperty* Prop = *It;
    UE_LOG(
        LogTemp,
        Log,
        TEXT("%s"),
        *Prop->GetName()
    );
}
// 查特定属性的接口 FindFProperty() 就是按上述方式遍历的
```

如何通过 UPROPERTY 的反射信息找到该属性在类对象实例的实际位置？

```cpp
// 类对象中的属性成员在对象内存中排布
class APlayer
{
    ...
    int32 Health;
    float Speed;
}; 
// 假设 APlayer 实例起始地址 0x1000，对象内存布局如下：
0x1000
│
│ UObject / Actor / Character 父类数据	
│ ...
├── 0x1120   Health
├── 0x1124   Speed
└── ...
// 那么对于属性 Health，它在对象内的偏移量 offset = 0x1120 - 0x1000 = 0x120
// 这个 offset 保存在属性变量的 Offset 成员中
// offset 的实际使用方式有 int32* Health = reinterpret_cast<int32*>(Base + HealthPropertyOffset);
// 更推荐的 UE 写法是 void* Value = Prop->ContainerPtrToValuePtr<void>(Player); 
```

为什么要在反射中维护成员的`offset`呢？

因为普通 C++ 访问成员时，成员偏移由编译器直接写进机器码；但反射系统**面对的是运行时通用对象，不能提前写死成员地址**

所以 UE 需要在 `FProperty` 中保存 Offset，这样 Editor、序列化、GC、Blueprint 等通用系统拿到`UObject* + FProperty*`就能在不需要提前知道具体 C++ 类型的情况下访问任何类成员，**使 UE 能在运行时动态定位和操作成员变量**

通过反射访问属性的核心访问链：

```
Player
  ↓
GetClass()
  ↓
UClass
  ↓
找到 Health 对应 FProperty
  ↓
FProperty 中记录 Health 相对 Player Container 的 Offset
  ↓
Player地址 + Offset
  ↓
Health真实地址
```

### `UFunction` 在 `UClass` 里怎么组织

一个 `UFUNCTION` 最终对应一个 `UFunction` 对象，每个 `UFunction` 自己保存`Name 、FunctionFlags、参数信息……` 

 `UFunction` 对象也和`UProperty`类似，作为 `UField` 组织在`UClass`的成员 `TObjectPtr<UField> Children` 中

`UField`还有成员`TObjectPtr<UField> Next`用于形成链表，如：

```cpp
UClass(APlayer)
│
└─ Children
      ↓
   UFunction Fire
      ↓ Next
   UFunction Reload
      ↓
     ...
// 和 Property 算是两套不同的 Field 体系
```

对于函数查找，通常不像`UProperty`一样做线性遍历，因为这样在函数很多、调用频繁的时候效率不高

`UClass` 专门维护`TMap<FName, TObjectPtr<UFunction>> FuncMap`提供**按函数名O(1)定位函数指针**的能力

为什么只给`UFunction`提供这样的能力？因为**蓝图、Process Event、RPC 等都大量涉及函数查找**，所以专门建立`FuncMap`是合理的优化

如果想查询父类函数时，直接通过`SuperClass->FuncMap`去查询

并且，`UFunction`函数的调用需要依赖`ProcessEvent()`接口。因为反射场景下调用者调用函数时并不知道函数具体信息，只知道目标对象指针`UObject*`、函数描述指针 `UFunction*` 和参数内存。而`ProcessEvent()`中针对`UFunction*`中的反射信息去在运行时正确地执行函数，是 RPC 就发起网络调用，是 Native C++ 函数就正常执行，是 Blueprint 函数就进入 Blueprint VM

如：

```
Player对象 + UFunction(Fire) + 参数内存
   ↓
ProcessEvent(&APlayer,&UFunction(Fire),Params)
   ↓
判断这个 UFunction 应该怎么执行
   ↓
Native C++ / Blueprint VM / Event / RPC相关路径
   ↓
真正执行 Fire
```

​	

反射的基础到此为止，反射实际如何被引擎使用的、反射在各个功能中扮演什么角色，会在其他篇章讲解