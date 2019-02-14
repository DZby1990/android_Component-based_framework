# 智能硬件及收银台等类似场景下，大团队的android组件化架构演进实践

## 背景

可以说自从移动端开发兴起以来，绝大多数公司的android团队规模都不大，大多都是不超过3个人做一个相对独立的项目。因此，绝大部分的android开发人员缺乏组件化和团队开发的经验和思维，对整体框架的概念也十分模糊。这样的结果，是导致一些相对需要较多android开发人员的公司，整个团队自上而下都呈现一种比较混乱的状态，例如一些收银台公司和智能硬件公司，一个事业部小几十个android开发人员负责一系列相关联的应用，在我听闻过得公司里，30几个安卓，领导不敢合并代码，或者一个项目工程需要编译半个小时，甚至合并一次代码花一夜时间这种事也是屡见不鲜。针对以上情况，结合笔者个人最近几年的相关工作，来抛砖引玉给大家阐述一下笔者个人对于这方面的认识和总结。

## 场景

抛开场景谈架构，必然是不合适的。这里就来先说说场景。上文中其实也有提到，本文主要的场景就是智能硬件及收银台场景，那么这个场景有什么特点呢？我们单独拿收银台来说，这样可以比较具体的阐述。收银台，看着虽然只有三个字，但是了解业务的同学应该知道，收银台这块内容，在不同的业务场景下，需求都是有一定差别的。这里细分开来，大的类别至少包含了商超零售版收银台，餐饮版收银台。再往下说，零售商超收银台，不同的商家需要不同的外接设备，一些大型的批发市场需要大型电子秤，一些便利店需要客显屏幕，或者需要接扫码，收银设备，还有各式各样的打印机也是必不可少的；而餐饮收银台，有些商户会需要子母机，有些商户会需要排座位；而且不同的商家，对收银台账号权限，会员，库存，报表，换班的需求也都是五花八门；甚至同样的需求，收银台的形式可以是pad，也可以是pos机，甚至就是一个普通的手机，在这些设备上的界面和交互必然是完全不同的。

说了这么多，总结下来，就是业务场景较多，针对不同的场景需要不同的应用才可以支撑，但是在不同场景的应用中，又有很多的基本需求是一致的，甚至完全一致的业务功能需求，需要完全不同的界面和交互。

这样，对我们的整体架构就提出了一定的要求。

## 高内聚低耦合的思路

针对这样一个整体场景，我们必然是在多个小组的情况下，整体推进各个项目。而每一个项目必然不能闭门造车，否则肯定会造成大量的重复劳动，使得效率降低，甚至随着项目的深入迭代，变得越来越低。同时，不同功能模块必须拥有隔离式的变更能力，且功能模块本身也需要细分，需要将界面交互与业务功能完全抽离。还需要在不同的应用中可以达到一定程度的可供配置。针对同样的功能，不同的硬件和后端环境，也需要达到业务实现的可供配置。

因此，我们首先对于架构要求的梳理就是模块可配，功能可配，界面可配，业务实现可配。这也正符合“高内聚低耦合”思路想法。

## Rome was not built in a day

#### 一口肯定是吃不成一个胖子的。针对上面的要求和思路，我们首先需要我们的version1版本的架构。

首先，我们迫切的需要解决第一个问题，大团队下的协作开发问题。我们对此提出一个要求，每个人负责的功能完全隔离，且组件可配。

针对上面的要求，我们的第一个想法就是maven仓库。通过maven仓库将module打包的aar包，放到gitlab上供远程调用。这里具体的实现，我会在专门的篇幅中进行说明，本文就不赘述。

在完成了上述的功能之后，我们所有的组件，都可以通过build.gradle，在dependencies 中进行拉取。但是这还没有完，我们在不断的更新迭代的过程中发现，android studio随着版本的提高，对于不同组件下依赖包的版本一致性的要求在不断提高。而且，随着组件的不断增加，统一化管理组件也势在必行。因此，我们在gitlab上，增加了一张配置表，专门配置所有组件的版本。以下的参考的配置表格式：

   ```objc 
ext {
//这里的ext是必须的

    android = [
            compileSdkVersion   : 25,
            buildToolsVersion   : '25.0.0',
            applicationId       : 'com.example.app',
            minSdkVersion       : 17,
            targetSdkVersion    : 25,
            versionCode         : 1,
            versionName         : 'v 1.1.0',
            defaultPublishConfig: 'release',
            publishNonDefault   : true
    ]

    dependencies = [
            "appcompat-v7"      : 'com.android.support:appcompat-v7:25.1.0',
            "support-design"    : 'com.android.support:design:+',
            "support-v4"        : 'com.android.support:support-v4:25.3.1',
            "recyclerview-v7"   : 'com.android.support:recyclerview-v7:23.2.0',
            "crashreport"       : 'com.tencent.bugly:crashreport:latest.release',
            "crashreport-native": 'com.tencent.bugly:nativecrashreport:latest.release',
    ]
}
```
这张表直接放在gitlab上，我们使用的时候，只需要在项目build.gradle中配置
apply from: 'http://gitlab.ops.xxx.so/android-dev/XCommon/raw/master/config.gradle'
def urls = rootProject.ext.urls 
然后在allprojects 中引入仓库地址，再在module的build.gradle中引入即可，以下是示例代码：

   ```objc 
allprojects {
repositories {
   maven {
            url urls["support-v4"]
        }
}
def cfg = rootProject.ext.android // 工程配置
def libs = rootProject.ext.dependencies // 库依赖
dependencies {
compile libs[' support-v4']
}

```
然后我们还需要在build.gradle中增加一段

   ```objc 
configurations.all {
    // Check for updates every build
    resolutionStrategy.cacheChangingModulesFor 0, 'seconds'
}
```
这一段的作用，就是保证在项目rebuild的时候，可以拉取最新的所有maven仓库。

完成了第一个步骤之后，我们的整体框架就可以顺利的将所有组件module，拆分到各个工程当中了。这个时候，你可能会问，这样完全隔离之后，调试怎么弄，难道都要在主工程做么？我们的解决方案是，在组件工程中增加主工程的代码，模拟各个场景的情况，跳转到相应的组件界面。

## 思考version1

在完成了上述工作之后，我们反过来思考一下，和我们最初提出的标准和要求相比，做到了哪些。基本上，只做到了模块可配，这个模块是大粒度的，举例来说，一个app，分为商品，会员，购物车功能模块，以上的方式只是将这三个功能模块的module分拆到工程当中，可以说和我们的初衷还是相去甚远。

那么接下来我们需要思考分解我们的模块粒度，将模块分解到各个组件。这一块也是组件化拆分的重要一环。涉及到了组件的复用性，为未来的硬件设备接入的驱动化方案和界面替换，功能实现替换奠定基础。


## 模块分拆

#### 前面已经提到了我们模块分拆的目标，即为未来的硬件设备接入和界面替换，功能实现奠定基础。

首先是分拆我们的view层，因为我们说到过，这样做的场景就是，同样的一套功能，可能有多套前面，比如pad，手机等。

我们单独将所有的view层都命名为xxxServiceImplView，以方便我们统一管理。这部分就是将我们原有的所有业务功能实现全部剥离，只包含activity及其页面状态的变化。

然后我们要谈谈业务功能具体应该如何剥离。这一块，其实很类似很多网站后端的做法，即接口与接口实现分离。我们需要定义xxxService和xxxServiceImpl。此处，我们以会员模块为例。我们已经将一整块的Member模块，分拆成了MemberServiceImplView，MemberService和MemberServiceImpl。MemberService组件中包含了所有业务接口，而MemberServiceImpl则包含了所有对MemberService业务接口的实现，至于MemberServiceImpl则包含了所有activity及其页面状态的变化。并且，我们在这里强调，数据模型必须转换成自己的业务模型，才可以提供给别的组件使用。这是什么意思呢，比如我们在MemberServiceImpl中实现了后端接口的交互，后端交给你的数据，就是数据模型。这个模型我们是不适合拿来给别的组件使用的。而且我们也要对自己提出更高的要求，需要从自己个人的角度理解业务模型，我们需要知道我们实现产品的功能，需要哪些参数，我们根据这些，来设计自己的业务模型，然后通过适配器模式，把后端的数据模型转换成自己的业务模型。

在这样做了之后，我们来看看有什么好处。MemberServiceImpl作为业务功能的实现，在做数据接口的时候，不管数据模型是否进行了调整，只要在适配器中重新修改数据模型即可。而且，更重要的是，假如我们是打印机之类的硬件设备，可能在实现方法上市完全不一样的两套代码，对于前段端面调用来说，我们已经封装了自己的业务模型，只需要调用Service接口print(String printDetail,Listener listener)即可。具体的实现完全不用去关心，也不用因为打印机设备的改变，而修改一整套打印逻辑。
说完了上面这些内容，接下来我们需要管理我们每一个Service和ServiceImpl实现的对应关系，这个我们应该怎么做呢？我们需要有一个组件来专门注册和管理我们的服务实现类。这个组件，我们称之为busService。

简单说一下这个组件的实现逻辑。首先，我们需要定义一个接口类，描述他的基本功能

   ```objc 
/**
     *
     * 注册业务接口，需要两个参数
     *
     * @param busServiceClass
     *        业务接口
     * @param busServiceImplClass
     *        业务接口具体实现
     */
    <TBusService extends IBusService,TBusServiceImpl extends TBusService> void register(Class<TBusService> busServiceClass, Class<TBusServiceImpl> busServiceImplClass);

    /**
     *
     * 取消注册业务接口，如果该业务接口已经有实例了，那么取消注册也会把实例删除
     *
     * @param busServiceClass
     *        业务接口
     */
    <TBusService extends IBusService> void unRegister(Class<TBusService> busServiceClass);


    /**
     *
     * 根据业务接口获取具体实现业务接口的实例
     *
     * @param busServiceClass
     *        业务接口
     * @return
     */
    <TBusService extends IBusService> TBusService getBusService(Class<TBusService> busServiceClass);


    /**
     *
     * 根据业务接口删除具体实现的实例，但是不会取消注册
     *
     * @param busServiceClass
     *        业务接口
     */
    <TBusService extends IBusService> void deleteBusService(Class<TBusService> busServiceClass);


    /**
     * 清空所有注册的实例
     */
    void clearAll();
    
```
以上备注中就已经详细描述了busService本身所具备的功能。而这个接口的实现类，必然是单例的，具体实现是比较容易的，这里就不加赘述。同时，我们为了方便以后的扩展，我们给这个实现类设置了代理，我们的所有工程中都使用名为BusServiceFactory的代理类来实现相关功能。

看完了以上的组件管理实现，其实我们也很清楚了，busService这相当于是一个中间件，统一管理所有的服务和服务实现。对于我们后续开发来说就十分方便了，所有的实现类，只需要在application里进行配置，一行代码就可以搞定所有替换工作。而所有使用服务相关功能的代码，都不需要有任何改变。

其实，讲完了上述两个部分，剩下来的，基本就都在view组件当中了。我们看看view组件，现在是处于一个什么样的状态。当前的view组件其实已经和后端api接口没有任何直接关系了，我们使用了自己的业务模型，完全通过service中的接口获取数据。这个时候，不管是我的view组件是pad版还是手机版，只需要将view组件重写即可。

## 再次回头思考

每当我们完成一个部分的时候，都要回头思考这个部分完成了我们预想的哪些内容，还有什么需要优化的。

以上的模块分拆到三个组件，确实完成了很多预想的架构模式。但是，我们还有一个十分常见的场景，就是不同的商家可能对某些模块有不同的需要。最常见的就是一些水果店，他们的会员体系，其实就是一个手机号加用户名称，并不需要你高大全的内容。这部分，也是需要我们能够随时配置的。

针对以上场景，我们将原来拆分成三块的组件，拆分成四块，新增一个viewservice,这个viewservice主要就是一个view功能的配置组件，合view组件分拆之后，针对一种场景，我们只需要配置一次viewservice即可。

谈完了上面这些，我们来看一下我们一个功能模块，现在的组件结构图：
 
![image](https://github.com/DZby1990/android_Component-based_framework/blob/master/pic/01.png "")

从上面可以看出，MemberServiceImplView和MemberViewService是直接提供给壳工程使用的。MemberServiceImplView通过配置MemberService的实现类MemberServiceImpl，然后直接使用MemberService的接口来实现功能。MemberViewService则可以直接提供给壳工程配置MemberServiceImplView所提供的业务功能。当然，我们也都是有默认的功能配置的，即使没有配置MemberViewService，也能实现默认的功能配置。而到了这一步，我们整体上已经把握了主要的一个努力方向，接下来还有很多细节工作需要完善。

## 组件间通信

#### 组件间通信我们需要梳理一下，分横向和纵向。

纵向上分为模块内部组件之间的调用和功能模块对工具模块的调用，针对模块内部组件之间的调用，view和serviceImpl，service，直接通过注册接口和接口实现的对应关系，提供接口给view组件调用即可。而功能模块对工具模块的调用，在后面的文章中就会有详细说明。

横向上即组件与组件之间的调用，这部分我们需要思考一下。我们针对“高内聚低耦合”这个方针，思考怎么样的模块，是给别人调用起来最舒服的。首先，我们的模块间调用，其实就是activity的跳转，简单点的，直接一个Intent就搞定了，但是这里有什么问题呢。这里我们需要付出比较大的沟通成本，因为不同模块间的跳转，我们都需要约定一系列的参数名。这样约定的话会比较繁琐，而且写错一个字母，都会导致问题。针对上面的现象，我们决定一致采用静态方法的方式来做。什么意思呢，就是在被吊起的activity中写一个静态方法start(Context context,string params)，将需要的参数暴露出来，这样组件间调用，只要通过方法下成员变量的名称，就可以知道参数的含义。

到这里，其实大部分的组件见场景就已经可以了，其实也是很简单的一个处理过程。但是在开发的过程当中，我们的一些工具模块，有很多不同的应用场景，需要各种各样不同的弹窗或者是处理逻辑，比如说摄像头，我们的弹框，调用的接口，返回什么内容，在不同的场景下都是不同的，如果用一堆if (){}else{}会使维护和调用变得越来越麻烦。如何解决这个问题也提上了日程。

## 工具模块的架构优化

上面说到了，我们希望工具模块本身功能是很纯粹的，不包含业务功能的。但是，工具模块有很多必不可少的，必须要在模块内弹窗和处理逻辑，我们希望把这部分内容交给业务模块来实现。

说到这里，我们第一个想法就是在业务模块获取工具模块activity的实例，这样，我们不管对他做什么都是可以的了。这部分灵感来源于leakcanary，他的入口类里实现了Application.ActivityLifecycleCallbacks接口。这个接口本身暴露了我们所需要的一切内容。这个接口本身会在所有activity生命周期发生变化的时候暴露生命周期变化activity的实例，这样我们就可以如愿在业务模块拿到工具模块activity的实例了。

拿到这个实例怎么做，首先，我们需要对这个方法进性包装，保证我们只会获取到我们想要的activity实例，以下是我们做的包装代码：
	
   ```objc 
public class StartActivityInstance<TActivity extends Activity> implements Application.ActivityLifecycleCallbacks{

private Class<TActivity> mActivityClass;
…
}
```
这里用泛型存入目标工具activity的class，对比刚onCreate的activity.getClass().equals(mActivityClass)是否为true即可。
然后，我们的工具模块要以什么方式暴露自己的功能呢？这里我们采用接口的方式去暴露我们的功能。什么意思呢，我们在被调起的工具模块中定义public的方法，让调用者可以塞入自己的监听，我们会在完成最纯粹的工具操作之后，通知调用者，开始接下来的操作。打个比方，我们在扫码工具模块中，塞入了setScanInfoListener(Listener listener),我们可以在Listener中定义相关的功能接口和返回的数据模型。当扫码模块扫到结果之后，塞入这个Listener通知调用界面。这个时候，我们调用者拿到的工具activity实例的作用就可以发挥出来了。你想让他扫到码之后调用a接口，拿到参数之后再弹个窗，可以，我们直接使用工具activity的实例context对象，在调用模块里跑接口a，拿到结果，往里塞界面就可以了，操作完一系列的操作，只需要finish()即可。
	
这里，我们还有一个小问题，我们有一种场景当a activity打开扫码b，扫到结果打开界面c，c又可以打开扫码界面d，这个时候，如果没有进行任何处理，扫码界面d的内容，会同时同步到a和c，这肯定是有问题的。针对上面问题的解决其实也很容易。我们在StartActivityInstance这个类里新加入一个全局变量private int mActivityHashCode = -1;
public void onActivityCreated(Activity activity, Bundle bundle) {}方法里，判断是否mActivityHashCode并且在给mActivityClass赋值之后，将当前工具	的hashCode()赋值给mActivityHashCode，这样就保证了每个业务模块界面，只会收到收到第一次打开的工具模块界面的监听回调。
	
## 回顾

经过以上的内容，我们已经将一个工程结构，变成了一个“壳”即主工程加上一系列的功能模块和一系列的工具模块和基础模块。一个业务模块，被分解成了四个组件，一个工具模块被拆分成了两个，或者四个模块，因为并不是所有的工具模块都包含界面。并且通过BusServiceFactory这个基础类，将各个组件联系起来。而主工程，只需要在gradle中配置所有需要加载的组件，并且在application中进行相关实现的配置，就可以了。

以下是目前我们的架构图：
![image](https://github.com/DZby1990/android_Component-based_framework/blob/master/pic/02.png "")
 


到这里，其实我们的整体架构已经达到了一定的完成度和可用性。回头来看，我们已经实现了开头提到的模块可配，功能可配，界面可配，业务实现可配。并且互相之间保持了相当高的独立性。业务实现组件，更是实现了互相之间的完全独立。同时，我们的一个组件就相当于一个工程，不仅解决了大型项目编译慢，合并复杂的问题，同时不同组件也具备单元测试的能力，不需要每次都通过主工程来进行调试。

## 选择和取舍

说了上面这么多，必然有很多人心理存在着各种各样的问题。我从我个人的角度，来谈谈读者心中可能的问题和我的答案。

#### 问题1：通篇讲述的组件化，还有当下热门的插件化，到底有什么区别，为什么笔者在这里只谈组件化？

首先，组件化和插件化本身是两个话题，只能说这两个话题有一定的联系。组件化关注的是整个工程，乃至一系列相关工程的整体架构的优化。他的初衷是保证大型项目在不断迭代的过程中能保持1+1>2的效果，能让迭代成为锦上添花越来越省力的事情。而插件化更多的是出于业务场景的需求，需要我们提高用户留存，降低用户感知，是由于这方面的考虑而做的工作。这两块并不矛盾，可以说是各司其职，在很多场景下可以共同发力，发挥作用，这在我后续的android硬件设备驱动化方案中就会提及。

#### 问题2：曾经某宝面试官问过我这样的问题，针对笔者这一套东西，为什么不用他们的Arouter？

其实这里笔者描述的组件化架构，和Arouter并不是一个东西，Arouter本身是一个路由框架，并不具备组件化的能力，只能说，可以成为组件化路上的一个有力帮手。

#### 问题3：和微信首先提出的p工程相比，有什么优缺点呢？

这个问题问到点子上了，也是笔者个人在此之前也思考过的问题。

P工程作为微信提出的方案，被包括美团在内的许多大公司所接受使用，必然是有一定的先进性的。它从编译上隔离了每一个p工程，这样，它的组件在拼合成一个完整应用的时候，就可以防止错误引用。而且在一定程度上缓解了module过多时的性能问题。

#### 但是，它依然没有解决几个问题：

（一）、没有解决不同模块之间的开发权限问题
使用p工程，我们依然只能依靠互相之间的约定，来约束互相之间的开发权限。通过负责人制度，来帮助管理。依然没有从根本上解决权限的问题。

（二）、工程编译依然很慢
据朋友所在公司使用p工程的结果来看，速度依然不容乐观，需要花大量的时间在编译当中。

#### 讲了上面的缺点，自然就是我们架构的优点：

（一）、针对上面的权限，由于我们从根本上隔离了所有组件，所以，我们可以直接借助git进行权限管理，哪些组件，谁负责谁开发，别人只能看，不能改。问题追溯，代码审查，都会有明确的责任人。

（二）、由于我们其实就是提供给主工程各个组件的aar包来使用，其实和工程应用v4包，v7包没有本质区别，所以编译速度极快，和一般的小项目没有感官上的差别。

（三）、所有业务模块都有按需单独配置单元测试的能力
	
#### 那么，我们的缺点又在哪呢？

（一）、环境管理较为复杂，需要所有组件完全一致的开发环境，和组件的版本环境。如果as要升级，要是涉及到修改，那必然需要所有组件都进行一遍修改。

（二）、跨组件的调试较为复杂。因为工程间的完全隔离，对跨组件的调试，必然就变得复杂，需要在上游组件甚至是主工程中才可以调试。每一次跨组件的调试，出现需要修改的内容，必然需要组件重新打包，上游组件重新拉取最新的包，才能进行新一轮的调试。这个过程确实是比较耗时的，对组内成员的水平有一定要求，如果水平较低，需要不断修改调试的话，必然影响到整个开发进度。

说完了优缺点，其实总的来说就是一个选择和取舍的问题。我们需要通过对项目的理解，知道我们所迫切需要的，和可以舍弃的，这样才能优化出适合我们项目的架构。


## 关于作者

郑逸钧，来自某支付公司，资深android开发。个人主页：http://engineerblog.cn

