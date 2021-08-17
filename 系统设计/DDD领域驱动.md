在理解领域驱动的时候，网上很多大谈理论的文章，这种对于初学者不是太容易接受。根据我自己的学习经历，建议按照如下几个步骤学习：

1. 粗略的看一遍领域驱动的理论，做到心中有形，知道领域驱动是什么，解决什么问题的，大概有哪几个模块即可。
2. 找一个具体的项目(推荐阿里的cola4)，了解定义了几个module，每个module的作用是什么的，分别负责什么功能，了解每个module都依赖了其他哪些module，每个module之间是如何实现了相互调用的。
3. 对照着项目中的module和领域驱动的理论，分析每个module分别对应理论中的哪块功能，理解该module的存在意义是什么，为什么这样设计，符合领域驱动中的哪个观点。

​		领域驱动设计是一个软件架构的设计理念，是一种设计思想，在代码结构上，并没有所谓的“标准”可言，只要符合领域驱动的设计理念即可。对于不同的业务系统可以设计出符合该理念、适合自己系统的框架结构。

​		首先理解一下常见的MVC和MVCS模型。MVC是我们说的比较多的一种模型，M-数据模型，V-数据展示，C-控制层。其中M层主要包括各种数据库实体类(entity)，DAO，Mapper等和数据的存储获取有关的类。V层在web系统中主要就是各种页面，html、jsp、js、css等。而对于C的话就是常见的各种Controller,用来接收前端的各种增删改查交互，同时还负责从M层获取数据的包装和适配，请求参数的有效性校验，前端请求的各种跳转和重定向等。

​		但这种严格的MVC在实际开发中并不怎么用到，因为这种需要在C层做一些重复性的功能，比如AController里面需要生成一个组织架构树，用于页面展示树状组织架构，而BController里面有一个方法实获取菜单的树状结构，在这两个controller里面构造树结构的代码是重复性的，可以封装提取出来。假如把该方法放在AController里面，则在BController就要引入AController，同时调用AController的该方法，这样就会存在controller之间的依赖，并且controller也显得异常臃肿庞大。

​		为了解决这个问题，可以引入MVCS模型，这个模型也就是我们日常中用的最多的模型。这里的S是指的业务服务层，把一些通用的功能方法提取到S层，用来为各个Controller提供服务。在实际的开发中，一般会为各个小模块定义自己的MVCS，一个完整的模块通常会包括：UserList.jsp、UserController、UserService、UseerDao。而多个小模块组成了整个project。

![常见MVCS结构](imgs\image-20210816172345518.png)

上面的modules.goods、modules.system都是在同一个idea-module(idea的模块)下，以目录来做物理隔离的。有时候会根据情况把modules目录下的功能模块独立出来，当做一个project下面的独立idea-module来处理。

![独立idea-module](imgs\image-20210817100911050.png)

​		

​		那么什么是领域驱动呢？在我的理解中就是把MVCS中的S层更加细粒度化，在“领域”的维度上进行拆分，实现每个领域的“**高内聚低耦合**”，以便达到领域之间的物理和逻辑的隔离。每个**领域的业务高度内聚在一起**，通过充血模型，实现自己的领域业务，同时暴露出接口供外部调用。

​		在传统MVCS中的S层，通常会根据某个对象创建一个service，例如UserService、CompanyService。或者根据功能创建一个service，例如LoginService、MessageService。这些只是简单的把一些可公用的方法简单的在物理层面放在一个java文件中，实现了简单的物理隔离。这种维度的拆分过于微观，是在方法粒度上进行的隔离，虽然实现了方法的共享，但是在整个service中取糅合着各种各样的功能。例如：定义一个OrderService,里面有订单的添加、修改、删除逻辑，还有订单的结算逻辑。但从领域角度考虑，订单的增删改查属于订单管理的业务(领域)，而订单的结算属于支付相关的业务(领域)，两个不同的业务逻辑代码却在相同的service中，这样安排会显得逻辑混乱。而按照领域驱动的思想，把同一领域的相关操作整合在同一domain下面，且只负责与该领域相关的操作，同时暴露出功能接口供外部调用。领域之间可以直接相互依赖聚合。当领域内部操作需要依赖其他服务(例如修改db数据)时，可以通过自定义接口方式满足领域内部操作，然后由其他领域外部服务实现该接口。

下面就用一个具体的领域驱动框架cola4来详细剖析下领域驱动的设计理念。

项目示例在：https://github.com/alibaba/COLA/tree/master/samples ，可以下载后直接导入idea。

导入后的结构如下：

![cola-demo系统结构](imgs\image-20210817153316309.png)

分别对应cola中的五个模块：

![cola架构图](imgs\cola架构图.jpg)

这里的五个idea-module分别是client、adapter、application(app)、domain、infrastructure，五个module的依赖关系如图所示：

- client不依赖项目中的其他模块（图中的cola组件不属于项目中的模块）
- application依赖domain和client模块，同时会通过domain间接依赖infrastructure
- adapter直接依赖于application，同时通过application间接依赖client
- domain不依赖任何其他模块，其内部会在gateway中定义领域网关接口
- infrastructure依赖于domain，在gatewayImpl中实现领域接口

其五个模块的详细功能如下：

- adapter
  - 适配层，一般是充当controller，对不同的请求方(页面/RCP)提供服务。对于未前后端分离的web系统，可以把静态页面放在该module下面。该层主要依赖于client层。
  - adapter依赖于client，在controller接收到请求之后，需要调用定义在client中的interface执行后续流程。
  - controller中接收的入参和出参也是定义在client中
- client(facade)
  - 有些文章会把client叫做facade，其目的是用来暴露接口和定义传递数据的。在client里面会定义一些interface和DTO、BO、VO等，当框架支持CQRS的时候，也会把各种event放在该层中。
  - controller调用的接口都会定义在该层中，同时各种入参和出参也在该层定义。
  - client层不依赖其他任何模块
- application(facadeImpl)
  - application层实现了各种**功能**，供其他模块调用，主要是adapter中的各种controller。
  - application实现了定义在client中的各种接口，而client接口中定义的方法就是暴露出供adapter层调用的功能。
  - application中会定义各种executor来实现各种功能的具体逻辑
  - executor在执行业务逻辑的时候，通常会调用domain进行业务处理，其依赖于domain模块。如果是简单的逻辑也会直接调用infrastructure层，例如一些简单数据的存储等
- domains
  - 领域层主要包括领域对象(domain)、领域服务(service)、和领域网关(gateway)。
  - 领域对象entity不同于DO和DTO。DO(data object)是属于数据层的对象和db表做一一对应；DTO（data transmission object）是在adapter层定义的数据传输对象，充当传输媒介。而领域对象定义在domain中，按照充血模型定义，在其内部实现各个领域的业务操作。
  - domain不依赖其他任何层，当需要调用其他模块服务时，则根据依赖倒置原则，在gateway里面定义个接口，domain的业务层调用该接口的方法完成整个业务逻辑，而接口的实现则放在Infrastructure实现。gateway中的领域接口也可以直接供application层调用。
- infrastructure
  - infrastructure为基础服务层，主要处理和外部系统的交互。例如数据库操作、redis操作、RPC调用等
  - gatewayImpl实现domain层的领域网关
  - 支持各种config配置
  - infrastructure中需要对领域对象转换为DO进行操作

下面就看下example中的add请求是怎么处理的。

​		首先在adapter中定义了一个MetricsController，该controller对外提供了addATAMetric方法，同时里面依赖了client中的com.alibaba.craftsman.api.MetricsServiceI接口。同时注意入参ATAMetricAddCmd，同样定义在client中。

![control调用client中定义的接口服务](imgs\image-20210817192445859.png)

之后找到addATAMetric的实现类是在app中，如下图所示。同时app中又依赖于其内部定义的各个executor完成对应的操作。

![image-20210817193640886](imgs\image-20210817193640886.png)

executor以组件形式定义在app中，同时调用domain中的领域接口，执行后续的业务操作。

![image-20210817193959703](imgs\image-20210817193959703.png)

定义在domain中的领域网关:

![image-20210817194158565](imgs\image-20210817194158565.png)

而领域网关的实现放在了infrastructure中：

![image-20210817194359582](imgs\image-20210817194359582.png)

infrastructure接收到定义在domain中的MetricItem领域对象后，首先转换为MetricDO对象，然后调用metricMapper插入数据库中。由于这里使用了CQRS,对数据的写操作需要通知到读模块，因此这里发布了一个MetricItemCreatedEvent，通知读模块更新数据。

![image-20210817200125691](imgs\image-20210817200125691.png)



通过上面的方式，跟着controller提供的接口，顺藤摸瓜一步一步跟踪到infrastructure层，就会对整个cola项目结构有个清晰的理解，然后再结合着领域驱动的设计思想，大概就明白了各个层级和接口的设计目的。







参考文章：

https://www.bianchengquan.com/article/539687.html

https://blog.csdn.net/significantfrank/article/details/100074716

https://www.cnblogs.com/duanxz/p/9922170.html

