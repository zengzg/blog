## 准备工作 ##
　　本文就Android Gradle Plugin的源码做简要分析，其中Android Gradle Plugin版本为2.3.0，Gradle版本为3.3。我们通过Gradle的依赖管理来查阅源码，所以在dependencies中引入**com.android.tools.build:gradle:2.3.0**依赖后，就可以在工程的External Libraries下找到Android Gradle Plugin的源码了。

###简介###
　　Gradle作为一个异常强大的构建工具，为了满足不同平台的需求，比如：Java平台有Java构建逻辑，Android平台有Android构建逻辑。Gradle务必是要支持自定义构建的，这个功能正是由Gradle Plugin提供，而对应于Android工程的构建逻辑就是由Android Gradle Plugin实现的了。
　　AGP(请允许我使用缩写)其实包含多个Gradle Plugin，不过通常开发过程中，我们只会用到其中的两个——com.android.application、com.android.library，对应于Android工程的Application与Library模块；当然，除了这两个插件，AGP还包含：com.android.test、com.android.atom、com.android.instantapp等插件。这些类似于域名的东西被称为Gradle Plugin Id，用于唯一标示一个插件。如果大家接触过早期的AGP，相信还见过*android*与*android-library*两个插件ID,当然这样命名不太具唯一性，所以后面更名为了域名格式；到2.3.0版本我们依然可以使用这两个废弃（deprecated）的ID——`apply plugin: 'android'`，当然不推荐大家使用了。
　　AGP定义了标准的Android工程构建流程，但是对于不同的Android应用，又会有各自的构建需求，比如，Android版本，CUP架构，应用分发渠道等等的不同，要实现这些应用的差异性构建，需要使用到一个名为Gradle Plugin Extension的特性。对应于前面提到的AGP中各个插件都有与之对应的一个“Extension”。各个应用不同的构建需求，就是配置在“Extension”中。
 　　上面提到的插件及其扩展，在源码中都有与之对应的类来实现，比如：com.android.application插件对应于类AppPlugin.java与AppExtension.java；com.android.library插件对应于类LibraryPlugin.java与LibraryExtension.java。都在gradle-2.3.0库中，当然其它插件的实现也在这里。刚才提及的那些插件ID的定义（或者说命名），也可以在库中的META-INFO目录中找到。
 　　本节提到的一些Gradle相关的特性和名词，有不了解的可以参阅这篇文章：[Write Custom Plugin](https://docs.gradle.org/current/userguide/custom_plugins.html#sec:getting_input_from_the_build)，里面介绍了Gradle Plugin的实现及如何发布。接下来我们就正式进入源码吧。

###Application模块###
　　在Android工程中，Application模块所使用的插件就是com.android.application，先看下模块级构建文件build.gradle的内容：

    apply plugin: 'com.android.application'
    ...
    android {
        defaultConfig {...}
        signingConfigs {...}
        buildTypes {...}
        productFlavors {...}
    }
    ...
以上就是Application模块基本的构建文件的结构了。第一行代码表示：应用名为com.android.application的插件；之后的`android`代码块其实是为一个名为`android`的`extension object`赋值的过程，各个应用差异化构建的配置信息便是保存在这个对象里的。之所以能以DSL为`extension object`赋值，这得益于Groovy语言的支持：[Groovy DSL](http://www.groovy-lang.org/dsls.html#section-delegatesto)。
　　跟踪进第一行代码，最后会调用方法AppPlugin.apply()，不过这里只是简单地调用AppPlugin的父类BasePlugin的方法，而BasePlugin类是所有定义在AGP中的插件的父类，所以：
    
    protected void apply(@NonNull Project project) {
        ...
        threadRecorder.record(...
                this::configureProject);

        threadRecorder.record(...
                this::configureExtension);

        threadRecorder.record(...
                this::createTasks);
        ...
        }
    }
上面代码中threadRecorder对象负责记录代码块执行时间，并且清晰地指出应用AGP插件时，做了三件事：configureProject，configExtension，createTasks。所以执行脚本时，AGP各自插件的基本流程是一样的，下面来看下这三个基本阶段。

####configureProject####
　　在配置工程阶段，主要做的是创建一些后续阶段会用到的对象，其中最重要的当属androidBuilder：
    
    private void configureProject() {
        ...
        checkGradleVersion();
        ...
        sdkHandler = new SdkHandler(...);
        ...
        androidBuilder = new AndroidBuilder(...);
        ...
        // Apply the Java and Jacoco plugins.
        project.getPlugins().apply(JavaBasePlugin.class);
        project.getPlugins().apply(JacocoPlugin.class);
        ...
    }

配置工程方法，先会检测Gradle版本，目前2.3.0要求的Gradle最低版本是3.3。之后创建sdkHandler对象来统一处理Android SDK相关的逻辑。接下来就是创建androidBuilder，如果我们把Android工程的构建看作是一个对象的话，那androidBuilder就是用于创建这个对象的，里面有一些大家耳熟能详的功能，比如，合并manifest，转换为DEX字节等，这些功能我会在之后略作描述。方法最后应用了两个插件：Java、Jacoco，可以看出AGP中的插件都是基于Java插件扩展的，而Jacoco其实是之后才加入的，当然鉴于本文仅限于AGP，所以Java与Jacoco插件就不多深入了。

####configureExtension####
　　配置扩展阶段，其实做的事和之前阶段一样，也是创建对象，在创建的这些对象中，大家会看到个单词——***variant***，这就是常说的[构建变体](https://developer.android.com/studio/build/build-variants.html)。下面是方法：
    
    private void configureExtension() {
        final NamedDomainObjectContainer<BuildType> buildTypeContainer = 
            project.container(BuildType.class, new BuildTypeFactory(...));
        final NamedDomainObjectContainer<ProductFlavor> productFlavorContainer = ...
        final NamedDomainObjectContainer<SigningConfig> signingConfigContainer = ...

        extension = createExtension(...);
        ...
        dependencyManager = new DependencyManager(...);
        ndkHandler = new NdkHandler(...);
        taskManager = createTaskManager(...);
        variantFactory = createVariantFactory(...);
        variantManager = new VariantManager(...);
        ...
        
        // map the whenObjectAdded callbacks on the containers.
        signingConfigContainer.whenObjectAdded(variantManager::addSigningConfig);

        buildTypeContainer.whenObjectAdded(
                buildType -> {
                    ... 
                    variantManager.addBuildType(buildType);
                });

        productFlavorContainer.whenObjectAdded(variantManager::addProductFlavor);


        // create default Objects, signingConfig first as its used by the BuildTypes.
        variantFactory.createDefaultComponents(
                buildTypeContainer, productFlavorContainer, signingConfigContainer);
    }

我们可以把上面代码功能归纳为3个：

 1. 开头的3行代码创建了3个集合对象，分别持有[BuildType（构建类型）](https://developer.android.com/studio/build/build-variants.html#build-types)、[ProductFlavor（产品风味）](https://developer.android.com/studio/build/build-variants.html#product-flavors)和[SigningConfig（签署设置）](https://developer.android.com/studio/build/build-variants.html#signing)。结合后面的3行“whenObjectAdded”方法，可以看出，在配置Android工程时指定的构建类型、产品风味和签署设置都添加在variantManager对象中，由其来统一管理。

 2. 代码中间部分对象赋值块，我这里选3个来讲讲：createExtension()、createTaskManager()、createVariantFactory()，这3个方法都是定义在BasePlugin类中的，选这3个来讲，是因为它们创建的对象与具体插件类型相关的。在com.android.application插件中分别创建的是AppExtension、ApplicationTaskManager、ApplicationVariantFactory对象，这在AppPlugin类中可以找到的。这里帖下插件扩展对象的创建：

        @Override
        protected BaseExtension createExtension(...) {
            return project.getExtensions()
                .create(
                        "android",
                        AppExtension.class,
                        ...);
        }
这里命名了扩展对象名为***android***，这也是为什么在配置文件（build.gradle）中的DSL是以***android***为根节点的原因。

 3. 方法的最后是创建一些默认的构建配置：

        public void createDefaultComponents(...) {
            ...
            signingConfigs.create(DEBUG);
            buildTypes.create(DEBUG);
            buildTypes.create(RELEASE);
        }
上面代码片段在类ApplicationVariantFactory中，我们知道构建Android工程时，Android Studio会生成一个默认签名（debug.keystore）与两个默认构建类型（debug、release），这些就是在这里创建的了。方法`create()`我就不跟进去分析了，功能就是创建各自的BuildTyp与SigningConfig对象，当然这里也不是简单地使用构造方法创建，想要进一步了解的可以看下BuildTypeFactory类，这是在之前创建container集合对象时用到的，当然ProductFlavor与SigningConfig也有对应的工厂类。

####createTasks####
　　Android工程的构建是一个庞大而复杂的流程，AGP把它划分为许多许多相对独立的流程，每个流程都对应一个Gradle Task，本阶段的功能就是创建这些Task。关于Gradle Task这个概念，不清楚的可以看这里：[Projects and tasks](https://docs.gradle.org/current/userguide/tutorial_using_tasks.html#sec:projects_and_tasks)，里面有对Gradle Task非常精简的说明，还有如何创建一个Task的介绍。下面来看看代码：

    private void createTasks() {
        threadRecorder.record(...
            () -> taskManager.createTasksBeforeEvaluate(...));

        project.afterEvaluate(
            project -> threadRecorder.record(...
                () -> createAndroidTasks(...)));
    }

这里将任务的创建分为两部分，也就是在**evaluate**前后。这里的**evaluate**可以理解为build.gradle文件执行前后 ，也就是说：方法`taskManager.createTasksBeforeEvaluate()`是build.gradle文件执行时调用；方法`createAndroidTasks()`是在build.gradle文件执行完后调用。之所以这样分步创建，是因为很多构建任务都依赖于对插件扩展对象，也就是在build.gradle文件中的***android dsl***部分的具体配置信息。对于**evaluate**更准确的理解，大家可以去看下：[Project evaluation](https://docs.gradle.org/current/userguide/build_lifecycle.html#sec:project_evaluation)（工程评估）。在工程评估后创建的任务，会放在后面专门的一个章节，这里可以先来看看工程评估时所创建的任务：

    public void createTasksBeforeEvaluate(...) {
        androidTasks.create(tasks, UNINSTALL_ALL, uninstallAllTask -> {
            ...
            uninstallAllTask.setGroup(INSTALL_GROUP);
        });
        ...
        androidTasks.create(tasks, new SourceSetsTask.ConfigAction(extension));
        ...
    }
其实这个方法中所创建的任务不只有这两个，但是因为创建方式都相同，也就略去了。`androidTasks`与`tasks`对象都可以理解为任务集合，不同处是前者属于AGP，后者属于Gradle，AGP创建的任务会同时属于这两个集合，任务创建好后需要交给Gradle来管理才能生效的。`UNINSTALL_ALL`与`INSTALL_GROUP`是两个字符串常量，前者是任务名（这里为uninstallAll任务），后者是其所属的任务组名，我们可以在Android Studio界面中找到对应的显示地方，就在Gradle任务列表里面。下面创建的任务是一个名为**sourceSets**的任务，所属的任务组为**android**，由类SourceSetsTask实现，不过大家不要被这个任务名给欺骗了，其实它只做些*打印输出*的工作——`SourceSetsTask.generate(...)`。以上两个任务的区别是：uninstallAll其实是一个空任务，它什么也没做，存在的意义是作为任务依赖源（供其它任务hook）；sourceSets有其具体的实现和处理逻辑。

###Android Gradle Task###
　　在前面已经有讲过任务创建是分两步走的，本节我们就来看看这些需要配置信息才能创建的任务。下面是在**createTask**阶段调用的方法：

    final void createAndroidTasks(boolean force) {
        ...
        threadRecorder.record(...
                () -> {
                    variantManager.createAndroidTasks();
                    ...
                });
        ...
    }
我把方法中的大部分代码都略去了，因为它们和创建任务没有特别大的关系，不过这里略去的有些代码还是满有意思的，比如我们在配置扩展对象时Android Studio会提示必需要指明buildToolsVersion和compileSdkVersion，这些检查就是在这个方法中做的。上面方法中的variantManager对象是之前在**configExtension**阶段创建的对象，也就是构建变体管理器，下面是方法VariantManager.createAndroidTasks()：

    public void createAndroidTasks() {
        ...
        if (variantDataList.isEmpty()) {
            recorder.record(...
                    this::populateVariantDataList);
        }
        ...
        for (final BaseVariantData<? extends BaseVariantOutputData> variantData : variantDataList) {
            recorder.record(...
                    () -> createTasksForVariantData(tasks, variantData));
        }
        ...
    }
可以看出这里先会创建所有的构建变体，再为每个构建变体创建任务。

####创建构建变体####
　　我们知道构建变体是构建类型与产品风味共同决定的，更为精确的说法，其实是由构建类型与**组合产品风味**共同决定的。说于这个概念大家可以去查看下这里：[Configure Product Flavors ](https://developer.android.com/studio/build/build-variants.html#product-flavors)，里面有关于产品风味维度的解释。也就是说，真正起作用的是**组合产品风味**。这些在接下的代码中会有很好的体现，下面是VariantManager类中的代码：

    public void populateVariantDataList() {
        if (productFlavors.isEmpty()) {
            createVariantDataForProductFlavors(Collections.emptyList());
        } else {
            List<String> flavorDimensionList = extension.getFlavorDimensionList();

            // Create iterable to get GradleProductFlavor from ProductFlavorData.
            Iterable<CoreProductFlavor> flavorDsl =
                    Iterables.transform(
                            productFlavors.values(),
                            ProductFlavorData::getProductFlavor);

            // Get a list of all combinations of product flavors.
            List<ProductFlavorCombo<CoreProductFlavor>> flavorComboList =
                    ProductFlavorCombo.createCombinations(
                            flavorDimensionList,
                            flavorDsl);

            for (ProductFlavorCombo<CoreProductFlavor>  flavorCombo : flavorComboList) {
                //noinspection unchecked
                createVariantDataForProductFlavors(
                        (List<ProductFlavor>) (List) flavorCombo.getFlavorList());
            }
        }
    }
这个方法中，我真是一行代码也未删除。可以看出根据productFlavors与flavorDimensions计算出**组合产品风味**列表，也就是集合对象`flavorComboList`，这部分逻辑写在`ProductFlavorCombo.createCombinations（）`这个静态方法里，不过这里我就不跟踪进入分析了，里面唯一值得注意的就是有个递归逻辑，我是建议大家自己进去品味品味，更有助于理产品风味维度的概念。得到flavorComboList后，就是创建构建变体了：

    private void createVariantDataForProductFlavors(
            @NonNull List<ProductFlavor> productFlavorList) {
        ...
        for (BuildTypeData buildTypeData : buildTypes.values()) {
            ...
            BaseVariantData<?> variantData = createVariantData(
                        buildTypeData.getBuildType(),
                        productFlavorList);
            variantDataList.add(variantData);
            ...
        }
        ...
    }
上面代码中传入的`productFlavorList`就是所谓的**组合产品风味**了，与每个构建类型结合创建成一个构建变体，这些构建变体就在`variantDataList`对象里。至于`createVariantData()`方法是如何创建一个变体对象的，就不再深入了。总之，代码运行到这里，我们已经得到了构建变体列表，下一步便是以此来创建任务了。

####createTasksForVariantData####
　　经过前面一系列的步骤，终于来到的build.gradle脚本执行的最后一步：

    public void createTasksForVariantData(...) {

        final BuildTypeData buildTypeData = buildTypes.get(...);
        if (buildTypeData.getAssembleTask() == null) {
            buildTypeData.setAssembleTask(taskManager.createAssembleTask(tasks, buildTypeData));
        }
        ...
        createAssembleTaskForVariantData(...);
        if (variantType.isForTesting()) {
            ...
        } else {
            taskManager.createTasksForVariantData(...);
        }
    }
现在我们知道，每种构建变体包含：1个构建类型，多个（或没有）产品风味。上面代码开始就为此构建类型创建对应的名为**assemble+buildType**形式的任务，然后是方法`createAssembleTaskForVariantData（）`创建的是名为**assemble+flavors+buildType**与**assemble+flavors**形式的任务,这里的**flavors**表示的就是**组合产品风味**了。代码的最后的就是调用`taskManager.createTasksForVariantData()`，该方法在TaskManager类中是一个抽象方法，具体实现每个AGP插件都有一份，以com.android.application插件为例，实现类为ApplicationTaskManager，而创建的任务就是诸如：***compileXXX***、***generateXXX***、***processXXX***、***mergeXXX***等等一系列方法了，太多了，本文就不一一列举了。至此android工程的Gradle Task就创建完成了。任务相关的内容里除了创建外，更重要的一个应该就是任务的职能，也就是这个任务是做什么的，下面我就列举些来看看。

####清单合并####
　　合并清单文件任务的名称为***processXXXManifest***，任务的实现类为：`com.android.build.gradle.tasks.MergeManifests`，在gradle-core-2.3.0库中。来看看MergeManifests.doFullTaskAction方法：

    protected void doFullTaskAction() {
        getBuilder().mergeManifestsForApplication(...);
    }
这个方法就是任务执行时调用的了，也就说任务的职能是定义在此的。getBuilder()返回的就是之前说过的***androidBuilder***对象！所以清单合并其实是由androidBuilder做的：

    public void mergeManifestsForApplication(...) {
        ...
        MergingReport mergingReport = manifestMergerInvoker.merge();
        ...
    }
真要说起来清单文件的合并逻辑也是一个比较复杂的逻辑，在AGP中把清单文件的合并功能单独出来，作为一个模块：***manifest-merger***。上面代码中的列出的类和对象都是出自此模块。不过，具体的合并逻辑我就不在本文深入了。

####DEX转码####
　　DEX转码功能并不是由Gradle Task直接实现的，而是创建的TransformTask任务，并委托给Transform实现的。而DEX转码的实现就是DexTransform类，在gradle-core-2.3.0库中。该任务名为：transformClassesWithDexForXXX。下面是DexTransform.transform()方法：

    public void transform(...) {
        ...
        androidBuilder.convertByteCode(...);
        ...
    }
毫无疑问，也是由androidBuilder对象处理的。不过接下来的转换代码我就不再帖了，略多略复杂，本文就不涉及了。

###结束###
　　本文就AGP的功能从其源码上略作分析，其中很多细节并未深入，很多内容也未涉及到，权当给大家在整体上些许体会吧。

> Written with [StackEdit](https://stackedit.io/).