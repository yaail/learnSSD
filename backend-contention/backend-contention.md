## SSD后端内存资源争用对公平性和性能的影响 
MQSim仿真器支持现有仿真器没有实现的多队列SSD设备模型，为许多研究，如流间干扰对性能的影响提供了方便。 
在文章[*MQSim: A Framework for Enabling Realistic Studies of Modern Multi-Queue SSD Devices*](https://www.usenix.org/conference/fast18/presentation/tavakkol)中，通过研究以下三个主要资源争用来源来验证流间干扰对公平性和性能影响： 
* 写缓存 
* 缓存映射表 
* 后端内存资源 
现尝试编译[MQSim源码](https://github.com/CMU-SAFARI/MQSim)，运行 fast18里的实例backend-contention，分析实验结果，研究后端内存资源争用对公平性和性能的影响。 
> 注：在运行过程中出了一点问题，工作负载定义文件workload-backend-contention-flow-1.xml有几行不必要的代码，需要删除后才能正常运行。 

实验的主要思路是，在屏蔽写缓存和缓存映射表争用的条件下，比较不同强度的流单独执行和同时运行时SSD的性能差别。 
实验设有两个不同的流Flow-1和Flow-2，Flow-1的强度低，Flow-2的强度高并逐渐增强（即队列深度逐渐增加），两个流都是随机访问读取，分别单独执行Flow-1、Flow-2，再同时执行Flow-1和Flow-2，执行命令如下： 
> MQSim.exe -i ssdconfig-backend-contention.xml -w workload-backend-contention-flow-1.xml 
> MQSim.exe -i ssdconfig-backend-contention.xml -w workload-backend-contention-flow-2.xml 
> MQSim.exe -i ssdconfig-backend-contention.xml -w workload-backend-contention-flow-1-flow-2.xml 
 
在输出文件.xml中，提取出延迟时间Device_Response_Time，列表如下： 
| Flow-2队列深度| 2 |  4 | 8 | 16 | 32 | 64 | 128 | 256 | 
|:-------:|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:| 
|Flow-1-alone| 92 | 92 | 92 | 92 | 92 | 92 | 92 | 92| 
|Flow-1-share| 98 | 104 | 116 | 140 | 191 | 306 | 550 | 1039| 
|Flow-2-alone| 92 | 98 | 110 | 134 | 184 |	299	| 542 | 1033| 
|Flow-2-share| 98 | 104 | 116 |	140 | 191 | 307 | 549 | 1038| 
其中Flow-1-alone表示Flow-1单独执行时的延时，Flow-1-share表示Flow-1和Flow-2同时执行时Flow-1的延时。Flow-2-alone和Flow-2-share同理。 
再提取出芯片级队列深度（即等待后端服务的请求数量）并取平均值，列表如下： 
|Flow-2队列深度| 2 |  4 | 8 | 16 | 32 | 64 | 128 | 256 | 
|:-------:|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:| 
|Flow-1|0.003909|0.003909|0.003909|0.003909|0.003909|0.003909|0.003909|0.003909| 
|Flow-2|0.003909|0.022377|0.091666|0.316094|0.966688|2.658375|6.460945|14.34906| 
|share|0.022341|0.052385|0.138594|0.384636|1.062051|2.771875|6.579433|14.43401| 
为对比同时执行和单独执行时的差别，分别计算Flow-2不同队列深度下延迟程度Slowdown（同时执行时的延时/单独执行时的延时），和队列深度变化值（同时执行时的平均队列深度/单独执行时的队列深度），并绘制表格，如下图所示： 
![flow-1](https://github.com/yaail/learnSSD/blob/master/backend-contention/Flow-1.PNG) 
![flow-2](https://github.com/yaail/learnSSD/blob/master/backend-contention/Flow-2.PNG) 
观察上图可知，Flow-1受Flow-2的影响，延迟时间和平均队列长度都迅速增长，当Flow-2的队列深度为256时，延迟时间增长到原先的11.3倍，芯片级队列深度增长到单独执行时的上千倍，但Flow-1对Flow-2的影响很小。 
以slowdown的较低值与较高值的比作为公平性的衡量标准，该比值越低，说明公平性越弱，如下图所示，随着Flow-2队列深度的增加，公平性逐渐减弱。 
![fairness](https://github.com/yaail/learnSSD/blob/master/backend-contention/Fairness.PNG) 
观察对比前面的表和图可知，高低强度的流相争，设备延迟时间会无限接近于高强度流单独执行的延迟时间，即高强度流可占用大部分后端资源，造成低强度流的请求向后延迟，等待请求的队列深度增加。SSD的性能和公平性逐渐降低。 