# sQueue

springboot + vue? or thymeleaf? + druid + mysql?+oracle + 
rabbitmq + redis + swgger2? + 

1、外部系统通过http调用排队叫号系统(his 核医学等外部业务系统)，
2、数据过来之后，通过字典表redis缓存，
先保存redis(使用redis的list集合保存每个科室的队列情况?-通过系统key+医院key+院区key+科室key) 
保存超时或者redis挂掉(超时设置?) 
之后保存数据库.
2、在redis无法使用时候保证系统可以使用数据库正常工作。
3、超时打印日志，使用apache storm进行日志分析。
4、设计表不光考虑到科室 院区 还考虑到医院 名称  为区域化设计做准备；
5、患者标识设计 急诊 军？ vip？不应设置不同字段flag 
不方便扩展，使用一个字段type无法支持多状态 即急诊又vip又军人
Type设置成多几位 第一位 1门诊/2住院 
第二位是否急诊 第三位 VIP 第四位 军人
门诊患者1000 住院2000 急诊 1100 VIP 1010 VIP急诊 0110 军急 1101
6、患者被叫过之后数据应落地 加入到mq消息队列(mq问题时容错?) 
做持久化或数据库已被查询等？
mq解耦 系统在拆分？ 业务处理？  查询处理系统？
7、缓存穿透  缓存失效? 系统启动加载缓存 虽然系统小也考虑进去
8、管理者领导查看院区队列情况？不同权限查看不同级别 医院-院区-科室-...
9、到诊确认？之前预约的插入到队列哪个位置？
如果需要到诊确认就设置两个SortSet队列 一个是明天预约的队列
 一个是明天正常的队列  预约的队列点到诊确认后才进入正常的队列
 //这个需要调研一下 妇婴好像是默认算时间
key为排队号 叫的时候分布式锁 锁定不让别人叫
叫完之后
入口考虑多支持? http 插库 socket连接? webservice?
10、sortset key设计 
需要一个表映射 根据患者Type取号前的标识 设计考虑个性化可是 院区 医院 default
普通排队患者A + 门诊原子INRC +
住院患者B + 住院原子INRC +

11、websocket 推送 每秒 推送

12、数据格式 医院-科室  
161-fsk-cs-queue1
A01 1 A02 2 A03 3
hash A01 id=123456 name=zs sex=nan

13、队列插入权限对应表 配置项
只有拥有对应权限才可以插入对应队列
token = 123456  hospital=161  dep = fsk 可以插入对应队列

14、模块拆分 
业务队列增删查改  后台管理(维护系统配置 字典表 )
统计查询系统数据分析 暂时不开发
 
15、队列状态 
未到诊 等待中 已呼叫 重叫 挂起 
退回查询已呼叫的列表? 队列在最前面
已呼叫直接插入数据库 redis留缓存zset 然后对过期时间进行设置
过期时间到当天晚上12点 就是列表只保留当天叫过的

16、医院结构表
医院 科室 诊室 
队列表 
队列no 队列名称 队列长度等
队列与信息绑定表；绑定队列表和医院结构表关系 --决定哪个诊室可以看到这个队列的信息；

17、患者绑定队列 插入时候
一个队列对应多诊室的时候咋显示?

18、外围系统请求插入队列 是否考虑使用mq先接 然后在处理
这样能缓冲 不直接接受外围系统请求  如果使用springboot独立部署开发可以只扩展这一块

19、手机端或者终端查询当前患者队列位置？
患者编号需要和队列建立联系 方便查询 ？ 

1、接受模块 requestRecive
外围系统请求接受本系统处理加入/删除队列 producer
队列名称 newPatientQueue
退费可能涉及删除队列；

2、请求处理模块 QueueHandler
本系统订阅队列newPatientQueue 作为 consumer 读取消息放到redis里展示 
队列中获取到 
请求参数token 校验权限判断是否有权限插入对应队列
队列号 患者信息
没问题则插入到redis 队列里
在消息处理完成之后加入到队列后证明某个队列有新患者需要websocket推送到前段 
把需要推送的队列 放入mq的changedQueue中  
后端检测推送模块进行消费推送
过程涉及 系统配置字典加载等  
展示 呼叫 退回 等操作  操作过的信息放入队列下一步
分布式读取消息队列 会有问题吗?
点击呼叫之后
请求controller 
放入persistenceQueue 做持久化

3、监控推送模块 monitorPush
需考虑 比如2台不同的大屏 链接不同的web 
其中一个取到了 队列更新信息 只能推送给自己链接到的大屏 另外一个就看不到
需要2个链接到同一台推送的服务器上
分布式 websocket 解决方案
消费changeQueue队列  考虑分布式读取
将读取到的数据 推送到指定的websocket端 

4、请求持久化模块
持久化操作 插入数据库


排队叫号终端支持? 有的厂商要求使用webservice给他推送
sQueue
