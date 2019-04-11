> Dragger2是一个适用于Android的**依赖注入**框架，那么依赖注入又是什么呢？依赖注入是用于在目标类中初始化成员变量实例的一种方法，不需要手动编码实例化一个成员变量对象，而是将其他类已经初始化好的实例注入到目标类中。

## 为什么要用依赖注入
假设现在你需要在A类中创建一个B类型的对象，会像下面这样：
```
public class A{
    B b;
    public A(){
        //手动编码实例化B对象
        b = new B();
    }
}
```
看到上面代码你可能觉得没什么问题，可是当B的构造函数后期修改了，需要传入参数，就不得不去改A类的代码，这就违反了开闭原则，因此我们需要通过依赖注入的方式将实例化完成的对象注入目标类中。

***

# 通过沙拉制作演示@Inject、@Module、@Component、@Provides注解的作用

首先我们讲一下他的主要过程，沙拉制作需要**香蕉**、**苹果**、**沙拉**，因此我们需要将这三者注入到沙拉制作类中，作为它的成员变量。但为了不去显式调用这三个类的构造方法，我们是通过注入完成，此时就需要使用@Inject标注被注入类中哪些对象需要注入。那么此时你会发现，我们怎么知道这三个类的构造过程呢，此时就需要这这三个类中对他们的构造函数进行@Inject标注。最后我们需要一个Module类（类似工程）对这些构造对象的构造进行管理，这个Module会使用@Module进行标注，其中提供注入类实例的方法使用@Provides标注，最后我们需要一个桥梁，将这些Module和被注入类进行链接，这个类就被标注为@Component，其中需要提供inject()函数表明被注入的类是哪一个。

![](https://note.youdao.com/yws/public/resource/49e816a1db0c0fcfd7328a64bfaf2c91/xmlnote/D17105ADB13E4BF187737A27872EB10C/16481)

## @Inject
> @Inject用于标注需要注入的类，假设需要注入的类时B，必须同时在目标类和B类的构造方法上作此标注，提示**要注入的是哪个类，这个类的构造方法又是什么**；

- #### 构造函数标注@Inject
我们假设接下来是要制作一盘水果沙拉,需要香蕉、苹果和沙拉;
```
public class Apple {
    //注解构造函数，标注从哪里生成实例,构造函数不需要参数
    @Inject
    public Apple(){
        Log.i("Apple","提供一个苹果");
    }
}
```
```
public class Banana {

    @Inject
    public Banana(){
        Log.i("Banana","提供一个香蕉");
    }
}
```
```
public class Sala {
    
    @Inject
    public Sala(){
        Log.i("Sala","加入沙拉");
    }
}
```
为什么需要对他们的构造函数标注@Inject呢，因为接下来你会看到沙拉制作过程需要这三种原料，他们是沙拉制作类的成员变量，但是他们是通过注入方式实例化到沙拉制作类中的,所以我们必须指明他们是怎么构造的。

- 被注入类成员变量注入@Inject
```
public class SalaDemo {
    @Inject
    public Apple mApple;
    @Inject
    public Banana mBanana;

    public void makeSala(){
       ...
    }
}
```
当需要注入哪个对象时，需要在其上方标注@Inject，但是现在还不能使用，我们还不知道谁去实例化这些注入对象，请看接下来。

## @Module和@Provides
这两个标签主要用于提供注入的实例的，@Provides用于提供实例，@Module相当于工厂，对注入对象做统一管理。
```
//提供注入对象的工厂
@Module
public class SalaModule {
    //提供苹果实例
    @Provides
    public Apple provideApple(){
        return new Apple();
    }
    //提供香蕉实例
    @Provides
    public Banana provideBanana(){
        return new Banana();
    }
    //提供沙拉实例
    @Provides
    public Sala provideSala(){
        return new Sala();
    }
}
```
提供实例的工厂有了,还是不可以使用,我们还缺少一个桥梁，用于连接这个Module工厂和被注入类（沙拉制作类），请看接下来的@Component标签。

## @Component
@Component是桥梁或者注射器的作用，可以将Module想象成药水，而被注入类（沙拉制作类）想象成皮肤，怎么将Module注入到沙拉制作类呢，此时就需要一个注射器（Component）。
```
@Component(modules = {SalaModule.class})
public interface SalaComponent {
    //会与Module对接，返回苹果实例
    Apple provideApple();
    //会与Module对接，返回香蕉实例
    Banana provideBanana();
    
    //标注被注入类
    void inject(SalaDemo sala);
}
```
@Component(modules = {SalaModule.class})的modules中指明药水（Module），可以有多个，而inject方法要说明注入到哪种类型的东西上面去，中间提供Module返回的实例。

## 最终使用
```
public class SalaDemo {

    @Inject
    public Apple mApple;
    @Inject
    public Banana mBanana;

    public void makeSala(){
        //建一个桥梁
        SalaComponent salaComponent = DaggerSalaComponent.create();
        //将依赖类注入
        salaComponent.inject(this);
        Log.i("Sala","正在制作沙拉中");
    }
}
```
- #### log
```
2019-04-11 12:09:40.281 23838-23838/top.carrotguo.myapplication I/Apple: 提供一个苹果
2019-04-11 12:09:40.281 23838-23838/top.carrotguo.myapplication I/Banana: 提供一个香蕉
2019-04-11 12:09:40.281 23838-23838/top.carrotguo.myapplication I/Sala: 加入沙拉
2019-04-11 12:09:40.281 23838-23838/top.carrotguo.myapplication I/Sala: 正在制作沙拉中
```
*** 

## 注入对象需要构造函数的情况
上面的实例没有任何问题，但是当某一个注入对象需要构造函数怎么解决呢？假设现在我们多了一个橘子，橘子需要用刀切，橘子的构造方法需要一个刀的参数。
- 橘子类（带参构造函数）
```
public class Orange {
    private Knife mKnife;
    //带参构造函数
    @Inject
    public Orange(Knife knife){
        this.mKnife = knife;
    }
}
```

- 水果刀类
```
public class Knife {
    public static int id = 1;

    @Inject
    public Knife(){
        Log.i("Knife","提供一把水果刀"+id);
        id++;
    }
}
```
#### 让Module工厂类提供刀子和橘子
```
@Module
public class SalaModule {

    @Provides 
    public Orange provideOrange(Knife knife){
        return new Orange(knife);
    }
    
    @Provides
    public Knife provideKnife(){
        return new Knife();
    }
}
```

实际上，橘子需要的参数会从Module中获取，这就解决了参数问题，对于需要的参数，让Module提供即可,其余不变，这样就可注入橘子类了。
假设我们在沙拉类找那个也引用了刀子。
```
public class SalaDemo {

    @Inject
    public Apple mApple;
    @Inject
    public Banana mBanana;
    @Inject
    public Orange mOrange;
    @Inject
    public Knife mKnife;
    @Inject
    public Sala mSala;

    public void makeSala(){
        SalaComponent salaComponent = DaggerSalaComponent.create();
        //将依赖类注入
        salaComponent.inject(this);
        Log.i("Sala","正在制作沙拉中");
    }
}
```
- log
```
2019-04-11 12:26:03.756 24015-24015/top.carrotguo.myapplication I/Apple: 提供一个苹果
2019-04-11 12:26:03.759 24015-24015/top.carrotguo.myapplication I/Banana: 提供一个香蕉
2019-04-11 12:26:03.760 24015-24015/top.carrotguo.myapplication I/Knife: 提供一把水果刀1
2019-04-11 12:26:03.761 24015-24015/top.carrotguo.myapplication I/Knife: 提供一把水果刀2
2019-04-11 12:26:03.761 24015-24015/top.carrotguo.myapplication I/Sala: 加入沙拉
2019-04-11 12:26:03.762 24015-24015/top.carrotguo.myapplication I/Sala: 正在制作沙拉中
```
对象注入确实没有问题了，可是我们平时使用需要注意，注入的对象不是单例，这里生成了两把水果刀。

# @Qualifier
> 很多情况下，一个类的构造函数不止一个，那么这是时候使用注解怎么区分注入使用哪个构造方法构造的对象呢？这时候@Qualifier限定符就派上用场了，自定义一个@Qualifier就可以对注入哪个构造函数构造的对象进行标注。
- #### 香蕉类修改为两种构造方式
```
public class Banana {

    private String mColor;

    public Banana(){
        Log.i("Banana","提供一个香蕉");
    }

    public Banana(String color){
        this.mColor = color;
        Log.i("Banana","提供一个黄色香蕉");
    }
}
```
注意：这个时候就不要对构造函数进行@Inject标注了。

- #### 自定义限定符Type，限定方式是String类型
```
@Qualifier//限定符
@Documented
@Retention(RetentionPolicy.RUNTIME)
public @interface Type {
    String value() default "";//默认值为""
}
```
自定义一个限定符@Type,这样我们可以使用@Type区分注入哪个构造函数构造的对象
- #### 在Module中实例化两个Banana,并且使用@Type区分
注意，这里方法名也不可以是一样的.
```
@Module
public class SalaModule {
    @Type("normal")
    @Provides
    public Banana provideBanana(){
        return new Banana();
    }

    @Type("color")
    @Provides
    public Banana provideColorBanana(String color){
        return new Banana();
    }
}
```
- #### 在Component中与Module进行对接，生成实例的方法也要标记Type
```
@Component(modules = {SalaModule.class})
public interface SalaComponent {
    //会与Module对接，返回苹果实例
    Apple provideNormalApple();
    //会与Module对接，返回香蕉实例
    @Type("normal")
    Banana provideBanana();
    @Type("color")
    Banana provideColorBanana();
    //会与Module对接，返回橘子实例
    Orange provideOrange();
    //会与Module对接，返回水果刀实例
    Knife provideKnife();
    String provideString();

    //标注被注入类
    void inject(SalaDemo sala);
}
```

# 实现全局单例的@Singleton
Dagger支持单例，@Singleton可以实现全局单例，即整个app内，只有一个这样的对象存在，接下来看看如何实现的。对于@Singleton的例子，我们假设只有全局我们只需要一把刀切水果
- #### 在Module中标注单例
```
@Module
public class SalaModule {
    //全局只提供一把水果刀
    @Singleton
    @Provides
    public Knife provideKnife(){
        return new Knife();
    }
}
```
- #### 在component中标注有实例是单例
```
public interface SalaComponent {
    //会与Module对接，返回水果刀实例
    Knife provideKnife();
}
```
注意，我们并非在方法上标注，而是在Component上标注。     
至此，我们就实现了单例。

# 实现区域单例@Scope
从接下来我们当做之前的水果沙拉没发生过。但是接下来我们依然是做水果沙拉，要求是这样的，香蕉和苹果用同一把刀切，橘子用另外一把刀切。

#### 分析
1. 香蕉和苹果的刀是局部单例，对于他们两个类来说是**单例**；
2. 橘子的刀不能和香蕉苹果的一样，不需要**单例**；

以上的需求要我们实现刀的局部单例。
- #### 刀类
```
public class Knife {
    private static int id = 1;
    @Inject
    public Knife(){
        Log.i("Knife","买了一把刀"+id);
        id++;
    }
}
```
只需要标注构造函数即可；
- #### 自定义作用域
自定义作用域注解的标志是@KnifeScope
```
//自定义一个作用域
@Scope
@Retention(RetentionPolicy.RUNTIME)
public @interface KnifeScope {
}
```

- #### 提供局部单例的module
```
@Module
public class KnifeModule {

    //提供局部单例
    @KnifeScope
    @Provides
    public Knife provideKnife(){
        return new Knife();
    }
}
```
使用@KnifeScope标注了刀的局部单例特性,使用标注后，从这个module获取的刀就是局部单例的，那怎么体现局部呢？看接下来的component。
- #### 提供局部单例的Component
```
//在Component上标记@KnifeScope,标志这个Conponent中共享一个单例
@KnifeScope
@Component(modules = {KnifeModule.class})
public interface KnifeComponent {

    Knife provideKnife();

    //标记只能注入到苹果和香蕉中
    void inject(Apple apple);
    void inject(Banana banana);
}
```
Component提供了knife的获取实例，同时标注只有那些类能够被注入，达到了单例作用范围约束的效果。

- #### Component的初始化，必须提供一个单例的Component给其他对象
```
public class DemoActivity extends AppCompatActivity {
    private static KnifeComponent knifeComponent;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        //创建一个component
        knifeComponent = DaggerKnifeComponent.builder()
                            .knifeModule(new KnifeModule()).build();
    }

    /**
     * 提供component给Apple和Banana使用
     * @return
     */
    public static KnifeComponent getKnifeComponent(){
        return knifeComponent;
    }
}
```
我们先创建一个component实例给苹果和香蕉类注入时使用。

- #### 两个水果类（被注入类）
```
public class Apple {
    @Inject
    Knife mKnife;

    @Inject
    public Apple(){
        Log.i("Apple","提供一个苹果");
        DemoActivity.getKnifeComponent().inject(this);
        Log.i("Apple","用刀切苹果");
    }
}
```
```
public class Banana {
    @Inject
    Knife mKnife;

    @Inject
    public Banana(){
        Log.i("Banana","提供一个香蕉");
        DemoActivity.getKnifeComponent().inject(this);
        Log.i("Banana","用刀切香蕉");
    }
}
```

此时，苹果类和香蕉类都获得了同一把刀，可以看下面的log，knife的构造函数只跑了一次
```
2019-04-11 16:52:08.112 26324-26324/top.carrotguo.myapplication I/Apple: 提供一个苹果
2019-04-11 16:52:08.124 26324-26324/top.carrotguo.myapplication I/Knife: 买了一把刀1
2019-04-11 16:52:08.125 26324-26324/top.carrotguo.myapplication I/Apple: 用刀切苹果
2019-04-11 16:52:08.127 26324-26324/top.carrotguo.myapplication I/Banana: 提供一个香蕉
2019-04-11 16:52:08.127 26324-26324/top.carrotguo.myapplication I/Banana: 用刀切香蕉
2019-04-11 16:52:08.136 26324-26324/top.carrotguo.myapplication I/DemoActivity: 制作沙拉中
```

这时候橘子的刀去哪里获取呢，我们需要提供一个生成另外一把刀的Module和component。

- #### 生成普通刀的Module和Component
```
@Module
public class OtherKnifeModule {
    @Provides
    Knife provideKnife(){
        return new Knife();
    }
}
```
```
@Component(modules = {OtherKnifeModule.class})
public interface OtherKnifeComponent {
    Knife provideKnife();

    void inject(Orange orange);
}
```

- #### Orange中注入Knife
```
public class Orange {
    @Inject
    Knife mKnife;

    @Inject
    public Orange(){
        Log.i("Orange","提供一个橘子");
        OtherKnifeComponent otherKnifeComponent =
                                DaggerOtherKnifeComponent.builder()
                                                .otherKnifeModule(new OtherKnifeModule())
                                                .build();
        otherKnifeComponent.inject(this);
        Log.i("Orange","用刀切橘子");
    }
}
```
此时，橘子也注入了knife的实例，但是这个knife和香蕉苹果的刀不是同一个对象。

#### 效果演示
```
public class DemoActivity extends AppCompatActivity {
    private static KnifeComponent knifeComponent;

    @Inject
    Apple mApple;
    @Inject
    Banana mBanana;
    @Inject
    Orange mOrange;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        //创建一个component,让其他类可以操作它
        knifeComponent = DaggerKnifeComponent.builder()
                            .knifeModule(new KnifeModule()).build();
        //直接注入
        DaggerFruitComponent.create().inject(this);
        makeSala();
    }

    /**
     * 提供component给Apple和Banana使用
     * @return
     */
    public static KnifeComponent getKnifeComponent(){
        return knifeComponent;
    }

    public void makeSala(){
        Log.i("DemoActivity","制作沙拉中");
    }
}
```

- log
```
2019-04-11 16:52:08.112 26324-26324/top.carrotguo.myapplication I/Apple: 提供一个苹果
2019-04-11 16:52:08.124 26324-26324/top.carrotguo.myapplication I/Knife: 买了一把刀1
2019-04-11 16:52:08.125 26324-26324/top.carrotguo.myapplication I/Apple: 用刀切苹果
2019-04-11 16:52:08.127 26324-26324/top.carrotguo.myapplication I/Banana: 提供一个香蕉
2019-04-11 16:52:08.127 26324-26324/top.carrotguo.myapplication I/Banana: 用刀切香蕉
2019-04-11 16:52:08.127 26324-26324/top.carrotguo.myapplication I/Orange: 提供一个橘子
2019-04-11 16:52:08.127 26324-26324/top.carrotguo.myapplication I/Knife: 买了一把刀2
2019-04-11 16:52:08.128 26324-26324/top.carrotguo.myapplication I/Orange: 用刀切橘子
2019-04-11 16:52:08.136 26324-26324/top.carrotguo.myapplication I/DemoActivity: 制作沙拉中
```
明显可以看出，苹果和香蕉使用的是同一把刀，而到了橘子的时候，需要重新买一把刀，实现了局部全局。

# Component间的依赖
>加入现在在其他一个module和Component中已经完成了土豆的依赖提供，这时候就不需要再SalaComponent中再重新做这些操作了，直接让SalaComponent去依赖另外一个Component即可。
- #### Potato类
```
public class Potato {
    @Inject
    public Potato(){
        Log.i("Potato","加入土豆");
    }
}
```
- #### 之前已经存在的module和component
```
@Module
public class PotatoModule {

    @Provides
    Potato providePotato(){
        return new Potato();
    }
}
```
```
@Component(modules = {PotatoModule.class})
public interface PotatoComponent {
    Potato providePotato();
}
```

上面不是现在才创建的，而是之前就有的，那么我们不需要再次将Potato再次添加到sala的module和component中，只需要直接依赖。
```
@Component(modules = FruitModule.class,dependencies = PotatoComponent.class)
public interface FruitComponent {
    Apple provideApple();
    Banana provideBanana();
    Orange provideOrange();

    void inject(DemoActivity activity);
}
```
- #### 在需要注入的类中添加Component的创建及依赖
```
//创建PotatoComponent
PotatoComponent potatoComponent = DaggerPotatoComponent
                                    .builder()
                                    .potatoModule(new PotatoModule())
                                    .build();
//直接注入
FruitComponent fruitComponent = DaggerFruitComponent.builder()
                                    .fruitModule(new FruitModule())
                                    .potatoComponent(potatoComponent)       //将potatoComponent传入
                                    .build();
fruitComponent.inject(this);
```
此时该类也可以注入potato。

# Dagger的懒加载
1. 使用 ==Lazy<T>== 引用对象可以实现对象的懒加载，并且可保证无论加载多少次都返回同一个实例，并且在第一次加载时才加载。
2. 使用 ==Provide<T>== 引用对象也是在第一次加载时才加载，但是每次加载返回的实例可能不同，取决于module如何提供实例。

> Lazy强调延迟加载，Provider强调重新加载；

```
@Inject
Lazy<Apple> mLazyApple;
@Inject
Provider<Banana> mBanana;


//如何获取apple实例
Apple mApple = mLazyApple.get();
```