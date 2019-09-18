**原文地址：https://upday.github.io/blog/model-view-controller/**

###**Android架构模式：MVC**

一年前，在upday中，大部分的Android团队所创建的应用远不如我们所期望的健壮与稳定。我们试图理解为什么我们的代码如此糟糕，我们发现了两个罪魁祸首：易变的UI与呆板的架构。这个应用已经在六个月中重新设计了四次。最终选择的设计模式似乎是MVC，但已远不是它本来的样子了。
让我们一起来探索下什么是MVC；近年来它是如何在Android中使用的；如何最大限度地提高可测试性；以及它的一些优点与缺点。

####**MVC模式**

通常UI逻辑要比业务逻辑更易变，因而前端工程师需要一种能够分离出用户界面的方案，MVC如是。

 - **Model**-数据层，负责管理业务逻辑并维护网络和数据库接口。
 - **View**-视图层，可视化的数据模型。
 - **Controller**-逻辑层，处理用户行为并更新必要的数据模型。

![enter image description here](https://upday.github.io/images/blog/mvc/mvc.png)

然而，这就意味着，Controller与View都依赖于Model：Controller更新数据，View接收数据。但是，对于当时的桌面和前端开发而言，关键点在于：Model应独立于View而测试。因此，诞生了几种MVC的变种。最著名的当是**Passive Model**与**Active Model**，以下是更多细节：

####**Passive Model**

在这个模型中，只有Controller可以操作Model。Controller基于用户行为来修改Model。当Model更新后，Controller及时通知View做必要的更新。此时，View会向Model请求数据。

![enter image description here](https://upday.github.io/images/blog/mvc/mvc_passive_model_sequence.png)

####**Active Model**

当Controller不是唯一可以修改Model的类时，Model需要一种方式来通知View和其它类更新。这里借助观察者模式来实现。Model持有一组观察者对象来处理更新。View实现观察者接口并向Model注册。

![enter image description here](https://upday.github.io/images/blog/mvc/mvc_active_model.png)

每当Model更新时，都会遍历观察者集合，并调用它们的`update`方法。此方法在View中的实现会触发向Model请求最新数据的操作。

![enter image description here](https://upday.github.io/images/blog/mvc/mvc_active_model_sequence.png)

####**Android中的MVC**

2011年左右，Android开始越来越受欢迎时，架构问题自然而然显现了，由于MVC是当时最受欢迎的UI模型之一，开发者就试着在Android中应用它。
如果你在StackOverflow上搜索类似“如何在Android中使用MVC”的问题，大多数会说，在Android中，Activity即是View也是Controller。回头来看，这真的是很疯狂的。不过在当时，重点是使Model可测试，并且View与Controller的实现通常视具体平台而定。

####**MVC在Android应该怎么使用**

如今，这个问题很好回答了。Activity、Fragment和View(Android控件)应该作为MVC中的***V***。Controller与Model应该是其它的类，并且不应继承和使用任何Android中的类。
当Controller需要通知View更新时，如何关联它们成了一个问题。在Passive Model模型中，Controller需要持有View的引用。为了实现可测试性，最简单的方法是创建一个BaseView接口，让Activity/Fragment/View继承它。这样Controller可以引用BaseView。

####**优点**

MVC高度支持关注分离。这个优势不仅提高了代码的可测试性，还使得代码更容易扩展，可以相当简洁地实现新功能。
Model类没有任何Android类的引用因而便于做单元测试。Controller不继承或实现任何Android类，仅有一个View接口的引用。这样，对Controller做单元测试亦是可行的。
如果View遵循了**单一职能原则**，那么它们的职责就仅仅是依据用户输入更新Controller并显示Model的数据，不必实现任何业务逻辑。这样，UI测试就应该足以覆盖View中的方法。

####**缺点**

#####**View同时依赖于Controller与Model**

View会因为对Model的依赖而变得异常复杂。为了减少View中的逻辑，Model就要对所有需要显示的数据，提供可测试的方法。在Active Model模型中，这成倍地增加了类和方法的数量，因为需要为每一种数据类型创建观察者。
由于View同时依赖于Controller与Model，在UI的中逻辑变化可能会导致多个类的更新，降低了灵活性。

#####**谁来处理UI逻辑？**

依据MVC模式，Controller更新Model，View从Model中获取数据并显示。但是谁来决定如何显示数据？是Model还是View？考虑下面的例子：现在有一个User，包含firstname和lastname。在View中需要显示为“lastname, firstname”格式，如“Doe, John”
如果Model仅提供元数据，则View中的代码应该为：

    String firstName = userModel.getFirstName();
    String lastName = userModel.getLastName();
    nameTextView.setText(lastName + ", " + firstName)

这说明是由View来负责处理UI逻辑。但这让UI逻辑几乎不可测试。
另一种方案是让Model仅暴露需要显示的数据，对View隐藏所有业务逻辑。但这样一来，Model就会同时持有业务逻辑与UI逻辑。虽然也可测试，但是Model隐式地依赖于View了。

    String name = userModel.getDisplayName();
    nameTextView.setText(name);

####**结语**

在早期的Android开发中，MVC模式使得许多开发者都感到困惑，并导致大量代码难于进行单元测试。
无论是View对Model的依赖还是让View处理来UI逻辑，都导致了代码库出现不可修复的错误，除非重构整个应用。那么架构的新目标是什么，又为什么？请看下篇文章。

> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbMjIwNzY1MDczXX0=
-->