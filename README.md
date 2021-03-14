# 高并发系统设计的方法和思考

	设计高并发系统的目的：将业务规则采用技术实现，做到系统的高可用，高并发，高性能，从而保证系统的稳定性。
		 提升技术能力，创造价值。
		 技术是需要业务作为支撑。
		 没有稳定性，一切都免谈。

# 通用设计方法

	1. Scale-out（横向扩展）：分而治之是一种常见的高并发系统设计方法，采用分布式部署的方式把流量分流开，让每个服务器都承担一部分并发和流量。
	2. 缓存：使用缓存来提高系统的性能，就好比用“拓宽河道”的方式抵抗高并发大流量的冲击。
	3. 异步：在某些场景下，未处理完成之前我们可以让请求先返回，在数据准备好之后再通知请求方，这样可以在单位时间内处理更多的请求。

	横向拓展：主要是分布式部署，流量切分
	缓存： 采用缓存提高并发，缩短响应时间
	异步： 服务逻辑采用异步解耦，采用异步的方式，后端处理时会把请求丢到消息队列中，同时快速响应用户，告诉用户我们正在排队处理，然后释放出资源来处理更多的请求
		  调用方不需要等待方法逻辑执行完成就可以返回执行其他的逻辑，在被调用方法执行完毕后再通过回调、事件通知等方式将结果反馈给调用方。

# 性能优化的4大指导原则

	1.不要过早优化，优化的前提：要结合具体业务场景，具体分析
	2.系统优化遵循“二八原则”，用20%的成本解决80%的性能问题，需要抓住主要矛盾，优先优化主要的性能瓶颈点
	3.性能优化需要有数据支撑，要时刻了解响应时间的减少了多少和吞吐量的提升了多少
	4.性能优化是随着业务的复杂性持续优化（罗马不是一天建成的，要不断寻找性能出现的瓶颈，持续优化）

# 性能优化度量指标

	1.请求平均响应时间
	2.响应时间的最大值和最小值
	3.分位值，p90,p95,p75不同区间响应时间是多少

# 高并发下的性能优化

	1.提高系统的处理核心数，增加系统并行处理能力（在资源利用，吞吐量，响应时间提升的时候，找到拐点）
	2.减少单次任务响应时间
		对cpu密集型选择高效的算法，提升运算能力；
		对于i/o密集型（比如db,cache,web系统等），减少i/o等待；
			比如： 
				数据库访问慢，是否有锁表，是否有全表扫描，索引是否合理，是否有大sql（一般是join操作）
				业务逻辑是否有缓存，缓存是否击穿或穿透
				网络是否有延迟，网卡是否有丢包情况等

# 系统高可用
	
	系统具备较高的无故障运行的能力
	通常来讲，一个高并发大流量的系统，系统出现故障比系统性能低更损伤用户的使用体验。
	想象一下，一个日活用户过百万的系统，一分钟的故障可能会影响到上千的用户。
	而且随着系统日活的增加，一分钟的故障时间影响到的用户数也随之增加，系统对于可用性的要求也会更高。

	度量指标：
		MTBF（Mean Time Between Failure）是平均故障间隔的意思，代表两次故障的间隔时间，也就是系统正常运转的平均时间。这个时间越长，系统稳定性越高。
		MTTR（Mean Time To Repair）表示故障的平均恢复时间，也可以理解为平均故障时间。这个值越小，故障对于用户的影响越小。

		可用性与 MTBF 和 MTTR 的值息息相关，我们可以用下面的公式表示它们之间的关系：
			Availability = MTBF / (MTBF + MTTR)
		这个公式计算出的结果是一个比例，而这个比例代表着系统的可用性。一般来说，我们会使用几个九来描述系统的可用性。

		系统可用性		年故障时间		日故障时间
		90%	1个9			36.5d 			2.4h
		99% 2个9     	3.65d 			74.4min
		99.9% 3个9      8h 				7.44min
		99.99% 4个9     52min   			8.6s
		99.999% 5个9    5min            0.86s
		99.9999% 6个g   32s 				86ms

		一般来说1个9，2个9很容易达到；故障时间越短，故障恢复时间越短，系统就越稳定；
		核心业务系统的可用性，需要达到四个九，非核心系统的可用性最多容忍到三个九。
		在实际工作中，你可能听到过类似的说法，只是不同级别，不同业务场景的系统对于可用性要求是不一样的；

# 高可用设计思路
	
	一个成熟系统的可用性需要从系统设计和系统运维两方面来做保障，两者共同作用，缺一不可；

	系统设计：
		在做系统设计的时候，要把发生故障作为一个重要的考虑点，预先考虑如何自动化地发现故障，发生故障之后要如何解决;
		系统设计过程中，要保证服务稳定性得到保证；
			比如：故障检测（一般是心跳检测），故障自动转移，超时控制，服务降级，服务限流；
			故障检测的方法： 使用最广泛的故障检测机制是“心跳”。你可以在客户端上定期地向主节点发送心跳包，也可以从备份节点上定期发送心跳包。
						   当一段时间内未收到心跳包，就可以认为主节点已经发生故障，可以触发选主的操作。
						   选主的结果需要在多个备份节点上达成一致，所以会使用某一种分布式一致性算法，比方说 Paxos，Raft；
			超时控制：
				超时控制实际上就是不让请求一直保持，而是在经过一定时间之后让请求失败，释放资源给接下来的请求使用。
				这对于用户来说是有损的，但是却是必要的，因为它牺牲了少量的请求却保证了整体系统的可用性。
			服务降级：
				降级是为了保证核心服务的稳定而牺牲非核心服务的做法。
				比方说我们发一条微博会先经过反垃圾服务检测，检测内容是否是广告，通过后才会完成诸如写数据库等逻辑；
			服务限流：
				它通过对并发的请求进行限速来保护系统；
				比如对于 Web 应用，我限制单机只能处理每秒 1000 次的请求，超过的部分直接返回错误给客户端。
				虽然这种做法损害了用户的使用体验，但是它是在极端并发下的无奈之举，是短暂的行为，因此是可以接受的。


	系统运维：
		运维层面，要做好灰度发布，故障演练，快速响应

		灰度发布： 系统的变更不是一次性地推到线上的，而是按照一定比例逐步推进的。一般情况下，灰度发布是以机器维度进行的。
				  比方说，我们先在 10% 的机器上进行变更，同时观察 Dashboard 上的系统性能指标以及错误日志。
				  如果运行了一段时间之后系统指标比较平稳并且没有出现大量的错误日志，那么再推动全量变更。
				  灰度发布给了开发和运维同学绝佳的机会，让他们能在线上流量上观察变更带来的影响，是保证系统高可用的重要关卡。

		故障演练： 对系统进行一些破坏性的手段，观察在出现局部故障时，整体的系统表现是怎样的，从而发现系统中存在的，潜在的可用性问题。
				  比如一个复杂的高并发系统依赖了太多的组件，比方说磁盘，数据库，网卡等，这些组件随时随地都可能会发生故障，而一旦它们发生故障，
				  会不会如蝴蝶效应一般造成整体服务不可用呢？我们并不知道，因此，故障演练尤为重要。
				  对于故障演练，一般建议搭建一套和线上部署结构一模一样的线下系统，然后在这套系统上做故障演练，从而避免对生产系统造成影响。
   
# 开发和运维眼中的高可用的方法
	
	从开发和运维角度上来看，提升可用性的方法是不同的：
		对于开发来说：
			开发注重的是如何处理故障，关键词是冗余和取舍。冗余指的是有备用节点，集群来顶替出故障的服务。
			比如文中提到的故障转移，还有多活架构等等；取舍指的是丢卒保车，保障主体服务的安全

		对于运维来说：
			从运维角度来看则更偏保守，注重的是如何避免故障的发生。
			比如更关注变更管理以及如何做故障的演练，当出现系统出现故障，怎么快速响应解决系统故障。

	只有开发和运维两个角度结合起来，才可以真正保证系统的高可用；
	与此同时，提供系统的高可用性，还需要考虑投入的成本，系统优化要合理，要考虑满足高可用的同时，是否可以带来收益，维持系统的稳定性；

# 系统的高可拓展性

	从架构设计上来说，高可扩展性是一个设计的指标，它表示可以通过增加机器的方式来线性提高系统的处理能力，从而承担更高的流量和并发。

	在单机系统中通过增加处理核心的方式，来增加系统的并行处理能力，但这个方式并不总奏效。
	因为当并行的任务数较多时，系统会因为争抢资源而达到性能上的拐点，系统处理能力不升反降。
	而对于由多台机器组成的集群系统来说也是如此。集群系统中，不同的系统分层上可能存在一些“瓶颈点”，这些瓶颈点制约着系统的横向扩展能力。
	比方说，你系统的流量是每秒 1000 次请求，对数据库的请求量也是每秒 1000 次。
	如果流量增加 10 倍，虽然系统可以通过扩容正常服务，数据库却成了瓶颈。
	再比方说，单机网络带宽是 50Mbps，那么如果扩容到 30 台机器，前端负载均衡的带宽就超过了千兆带宽的限制，也会成为瓶颈点。

	其实，无状态的服务和组件更易于扩展，而像 MySQL 这种存储服务是有状态的，就比较难以扩展。
	因为向存储集群中增加或者减少机器时，会涉及大量数据的迁移，而一般传统的关系型数据库都不支持。
	这就是为什么提升系统扩展性会很复杂的主要原因。

	除此之外，从例子中你可以看到，我们需要站在整体架构的角度，而不仅仅是业务服务器的角度来考虑系统的扩展性 。
	所以说，数据库、缓存、依赖的第三方、负载均衡、交换机带宽等等都是系统扩展时需要考虑的因素。
	我们要知道系统并发到了某一个量级之后，哪一个因素会成为我们的瓶颈点，从而针对性地进行扩展。

# 高可扩展性的设计思路
	
	服务拆分：将复杂的系统，拆分为小模块，将复杂的问题简单化
		拆分是提升系统扩展性最重要的一个思路，它会把庞杂的系统拆分成独立的，有单一职责的模块。
		相对于大系统来说，考虑一个一个小模块的扩展性当然会简单一些，将复杂的问题简单化，这是我们的思路；

	1.存储层的拓展性：
		无论是存储的数据量，还是并发访问量，不同的业务模块之间的量级相差很大；
		比如说成熟社区中，关系的数据量是远远大于用户数据量的，但是用户数据的访问量却远比关系数据要大。
		所以假如存储目前的瓶颈点是容量，那么我们只需要针对关系模块的数据做拆分就好了，而不需要拆分用户模块的数据。
		所以存储拆分首先考虑的维度是业务维度。
		在垂直方向，按照业务纬度，将存储层的数据库拆分；
		比如用户库，内容库，订单库，商品库，库存库，评论库，商品优惠库等等；这么做还能隔离故障，某一个库“挂了”不会影响到其它的数据库。

		按照业务拆分服务和库存在的问题：
			按照业务拆分，在一定程度上提升了系统的扩展性，但系统运行时间长了之后，
			单一的业务数据库在容量和并发请求量上仍然会超过单机的限制。这时，我们就需要针对数据库做第二次拆分。

		数据按照一定的特征水平拆分：
			这次拆分是按照数据特征做水平的拆分，比如说我们可以给用户库增加两个节点，然后按照某些算法将用户的数据拆分到这三个库里面；

		水平拆分存在的问题：
			水平拆分之后，我们就可以让数据库突破单机的限制了。但这里要注意，我们不能随意地增加节点。
			因为一旦增加节点就需要手动地迁移数据，成本还是很高的。
			所以基于长远的考虑，我们最好一次性增加足够的节点以避免频繁的扩容。

		数据库按照业务和数据维度拆分存在的问题：
			拆分后，我们尽量不要使用事务。
			因为当一个事务中同时更新不同的数据库时，需要使用二阶段提交来协调所有数据库，要么全部更新成功，要么全部更新失败。
			这个协调的成本会随着资源的扩展不断升高，最终达到无法承受的程度。

	2.业务层的扩展性
		我们一般会从三个维度考虑业务层的拆分方案，它们分别是：业务维度，重要性维度和请求来源维度。
		1）我们需要把相同业务的服务拆分成单独的业务池，每个业务依赖自己的数据库资源，数据层面相互隔离；
			比方说社区系统中，我们可以按照业务的维度拆分成用户池、内容池、关系池、评论池、点赞池和搜索池；
			每个业务依赖独自的数据库资源，不会依赖其它业务的数据库资源。这样当某一个业务的接口成为瓶颈时，
			我们只需要扩展业务的池子，以及确认上下游的依赖方就可以了，这样就大大减少了扩容的复杂度。

		2）根据业务接口的重要程度，把业务分为核心池和非核心池，按照业务重要程度划分，提供业务层面的拓展性。
			打个比方，就关系池而言，关注、取消关注接口相对重要一些，可以放在核心池里面；
			拉黑和取消拉黑的操作就相对不那么重要，可以放在非核心池里面。
			这样，我们可以优先保证核心池的性能，当整体流量上升时优先扩容核心池，降级部分非核心池的接口，从而保证整体系统的稳定性。

	    3）根据接入客户端类型的不同做业务池的拆分；
	    	比如，服务于客户端接口的业务可以定义为外网池，服务于小程序或者h5页面的业务可以定义为H5池，服务于内部其它部门的业务可以定义为内网池等等。

# 对于未拆分的服务思考
	
	未做拆分的系统虽然可扩展性不强，但是却足够简单，无论是系统开发还是运行维护都不需要投入很大的精力。
	拆分之后，需求开发需要横跨多个系统多个小团队，排查问题也需要涉及多个系统，运行维护上，可能每个子系统都需要有专人来负责，
	对于团队是一个比较大的考验，也就是人力成本，这个考验是我们必须要经历的一个大坎，需要我们做好准备。
