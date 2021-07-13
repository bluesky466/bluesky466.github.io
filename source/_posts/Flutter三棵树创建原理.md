title: Flutter三棵树创建原理
date: 2021-07-13 22:45:20
tags:
    - 技术相关
    - Flutter
---

flutter的渲染机制基本就是靠Widget、Element、RenderObject三棵树去实现的，这篇博客就来讲讲这三棵树是怎么创建的。

首先我们来看看这三者到底是个啥:

- Widget: 描述一个UI元素的配置数据，不可变，修改信息需要重新new

- Element: 通过Widget配置实例化出来的对象,它是可变的
- RenderObject: 真正的渲染对象

让我们用一个简单的demo来做讲解:

```dart
void main() {
  runApp(MyApp());
}

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter Demo',
      home: HelloWorldPage(),
    );
  }
}

class HelloWorldPage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Center(
      child: Text("Hello World", style: TextStyle(color: Colors.blue)),
    );
  }
}
```

上面的代码正在屏幕的中间显示了一个Hello World字符串。

{% img /Flutter三棵树创建原理/1.png %}

# runApp

在main函数里面只有一行runApp调用，追踪下去我们可以看到它主要做了三件事情:

```dart
void runApp(Widget app) {
  WidgetsFlutterBinding.ensureInitialized()
    ..scheduleAttachRootWidget(app)
    ..scheduleWarmUpFrame();
}

void scheduleAttachRootWidget(Widget rootWidget) {
    Timer.run(() {
      attachRootWidget(rootWidget);
    });
}
  
void attachRootWidget(Widget rootWidget) {
    ...
    _renderViewElement = RenderObjectToWidgetAdapter<RenderBox>(
      container: renderView,
      debugShortDescription: '[root]',
      child: rootWidget,
    ).attachToRenderTree(buildOwner!, renderViewElement as RenderObjectToWidgetElement<RenderBox>?);
    ...
}

RenderObjectToWidgetElement<T> attachToRenderTree(BuildOwner owner, [ RenderObjectToWidgetElement<T>? element ]) {
    ...
    element = createElement();
    ...
    element!.mount(null, null);
    ...
  return element!;
}
```

1. 创建RenderObjectToWidgetAdapter作为Widget树的根，将传入的Widget挂上去
2. 调用RenderObjectToWidgetAdapter.createElement创建Element
3. 调用Element.mount将它挂到Element树上，Element树的根节点的parent为null

# Element.mount

Element的mount方法是三棵树创建流程的关键步骤，不同类型的Element mount的流程不太一样。

## 1.RenderObjectElement会创建RenderObject

如果Element是RenderObjectElement类型的，那么它对应的Widget一定是RenderObjectWidget类型的，这是它的构造函数决定的:

```dart
abstract class RenderObjectElement extends Element {
  RenderObjectElement(RenderObjectWidget widget) : super(widget);
  ...
}
```

它在mount的时候会调用RenderObjectWidget.createRenderObject创建RenderObject然后将它挂到RenderObject树上:

```dart
RenderObject get renderObject => _renderObject!;

RenderObject? _renderObject;

void mount(Element? parent, Object? newSlot) {
    ...
    _renderObject = widget.createRenderObject(this);
    ...
    attachRenderObject(newSlot);
    ...
}


void attachRenderObject(Object? newSlot) {
    ...
    // 插入RenderObject树
    _ancestorRenderObjectElement = _findAncestorRenderObjectElement();
    _ancestorRenderObjectElement?.insertRenderObjectChild(renderObject, newSlot);
    ...
}
```

这个_findAncestorRenderObjectElement方法比较魔性，找的是祖先RenderObjectElement，其实就是往parent一层层查找，直到找的RenderObjectElement:

```dart
RenderObjectElement? _findAncestorRenderObjectElement() {
    Element? ancestor = _parent;
    while (ancestor != null && ancestor is! RenderObjectElement)
      ancestor = ancestor._parent;
    return ancestor as RenderObjectElement?;
}
```

insertRenderObjectChild方法将创建的RenderObject插入成为祖先RenderObjectElement的RenderObject的子节点，这样就把创建的RenderObject挂到了RenderObject树上。

## 2.创建子Element并mount到Element树

处理完本节点的RenderObject之后，就会创建子Element将它的parent设置成自己，mount到Element树上。

Element都是通过Widget.createElement创建的，而Element会保存创建它的Widget。所以可以通过这个Widget去获取子Widget，然后用子Widget去创建子Element。

子Widget的获取有两种方式，如果是在Widget的构造函数传入的，那么直接可以拿到它，例如上面的RenderObjectToWidgetAdapter，然后用它去createElement创建子Element:

```dart
// 子widget是child参数传进去的
RenderObjectToWidgetAdapter<RenderBox>(
      container: renderView,
      debugShortDescription: '[root]',
      child: rootWidget,
)


void mount(Element? parent, Object? newSlot) {
    ...
    _rebuild();
    ...
}

void _rebuild() {
    ...
    // widget.child拿到构造函数传进去的子widget，即rootWidget
    _child = updateChild(_child, widget.child, _rootChildSlot);
    ...
}

Element? updateChild(Element? child, Widget? newWidget, Object? newSlot) {
    ...
    newChild = inflateWidget(newWidget, newSlot);
    ...
}

Element inflateWidget(Widget newWidget, Object? newSlot) {
    ...
  // 创建子Element
    final Element newChild = newWidget.createElement();
    ...
  // 调用子Element的mount方法将它挂到Element树上，parent是第一个参数this
  newChild.mount(this, newSlot);
  ...
    return newChild;
}
```

像StatelessWidget这种子widget是build出来的，则在mount的时候会调用它的build方法创建子widget，然后用它去createElement创建子Element:

```dart
void mount(Element? parent, Object? newSlot) {
    ...
    _firstBuild();
    ...
}

void _firstBuild() {
    rebuild();
}

void rebuild() {
    ...
    performRebuild();
    ...
}

void performRebuild() {
    ...
    built = build();
    //updateChild在上面也有追踪这里就不列出来了，内部调用了built.createElement创建子Element并返回
    _child = updateChild(_child, built, slot);
    ...
}

Widget build() => widget.build(this);
```

最终得到的三棵树大概长下面的样子，由于没有分成所以看上去是链表而不是树，但是这不影响我们理解，一旦某些节点有多个child节点就是输了:

{% img /Flutter三棵树创建原理/2.png %}

Element通过widget成员持有Widget，如果是RenderObjectElement还通过renderObject成员持有RenderObject，可以看出来Element是连接Widget和RenderObject的桥梁。三个树的构建也都是通过递归mount Element去实现的。

当RenderObject树创建出来之后，Flutter的引擎就能遍历它去执行绘制将画面渲染出来了。

# mount流程解析

从上面的代码可以看得出来，mount是一个递归的过程，总结下来有下面几个步骤

1. Element如果是RenderObjectElement则创建RenderObject，并从祖先找到上一个RenderObjectElement，然后调用祖先RenderObjectElement的RenderObject的insertRenderObjectChild方法插入创建的RenderObject
2. 如果子widget需要build出来就调用build方法创建子widget，如果不需要直接在成员变量可以拿到子widget
3. 调用子widget的createElement创建子Element
4. 调用子Element的mount方法将子Element的parent设置成自己，然后子Element去到第1步

下面的动图展示了整个流程:

{% img /Flutter三棵树创建原理/3.gif %}

或者可以下载[PPT](https://github.com/bluesky466/bluesky466.github.io/blob/develop/source/Flutter%E4%B8%89%E6%A3%B5%E6%A0%91%E5%88%9B%E5%BB%BA%E5%8E%9F%E7%90%86/Flutter%E4%B8%89%E6%A3%B5%E6%A0%91%E6%9E%84%E5%BB%BA%E8%BF%87%E7%A8%8B.pptx)查看