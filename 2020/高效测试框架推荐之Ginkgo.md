自2015年开始，七牛工效团队一直使用Go语言+[Ginkgo](http://onsi.github.io/ginkgo/)的组合来编写自动化测试用例，积累了大约5000+的数量。在使用和维护过程中，我们觉得Ginkgo的很多设计理念和功能非常赞，因此特分享给大家。

>本篇不是该框架的入门指导。如果您也编写或维护过大量自动化测试用例，希望能获得一些共鸣.

## BDD(Behavior Driven Development)

要说Ginkgo最大的特点，笔者认为，那就是对BDD风格的支持。比如：

```
	Describe("delete app api", func() {
		It("should delete app permanently", func() {...})
		It("should delete app failed if services existed", func() {...})
```
It's about expressiveness。Ginkgo定义的DSL语法(Describe/Context/It)可以非常方便的帮助大家组织和编排测试用例。在BDD模式中，测试用例的标题书写，要非常注意表达，要能清晰的指明用例测试的业务场景。只有这样才能极大的增强用例的可读性，降低使用和维护的心智负担。

可读性这一点，在自动化测试用例设计原则上，非常重要。因为测试用例不同于一般意义上的程序，它在绝大部分场景下，看起来都像是一段段独立的方法，每个方法背后隐藏的业务逻辑也是细小的，不具通识性。这个问题在用例量少的情况下，还不明显。但当用例数量上到一定量级，你会发现，如国能快速理解用例到底是能做什么的，真的非常重要。而这正是BDD能补足的地方。

不过还是要强调，Ginkgo只是提供对BDD模式的支持，你的用例最终呈现的效果，还是依赖你自己的书写。

## 进程级并行，稳定高效

相应的我们知道，BDD框架，因为其DSL的深度嵌套支持，会存在一些共享上下文的资源，如此的话想做线程级的并发会比较困难。而Ginkgo巧妙的避开了这个问题，它通过在运行时，运行多个被测服务的进程，来达到真正的并行，稳定性大大提高。其使用姿势也非常简单，`ginkgo -p`命令就可以。在实践中，我们通常使用32核以上的服务器来跑集测，执行效率非常高。

这里有个细节，Ginkgo虽然并行执行测试用例，但其输出的日志和测试报告格式，仍然是整齐不错乱的，这是如何做到的呢？原来，通过源码会发现，ginkgo CLI工具在并行跑用例时，其内部会起一个监听随机端口的本地服务器，来做不同进程之间的消息同步，以及日志和报告的聚合工作，是不是很巧妙？

## 其他的一些Tips
Ginkgo框架的功能非常强大，对常见测试场景的都有比较好的支持，即使是一些略显复杂的场景，比如：

* 在平时的代码中，我们经常会看到需要做异步处理的测试用例。但是这块的逻辑如果处理不好，用例可能会因为死锁或者未设置超时时间而异常卡住，非常的恼人。好在Ginkgo专门提供了原生的异步支持，能大大降低此类问题的风险。类似用法：

	``` 
	It("should post to the channel, eventually", func(done Done) {
	    c := make(chan string, 0)
	    go DoSomething(c)
	    Expect(<-c).To(ContainSubstring("Done!"))
	    close(done)
	}, 0.2)
	```
* 针对分布式系统，我们在验收一些场景时，可能需要等待一段时间，目标结果才生效。而这个时间会因为不同集群负载而有所不同。所以简单的硬编码来sleep一个固定时间，很明显不合适。这种场景下若是使用Ginkgo对应的matcher库[Gomega](https://github.com/onsi/gomega)的[Eventually](http://onsi.github.io/gomega/#making-asynchronous-assertions)功能就非常的贴切，在大大提升用例稳定性的同时，最大可能的减少无用的等待时间。
* 笔者一直认为，自动化测试用例不应该仅仅是QA手中的工具，而应该尽可能多的作为业务验收服务，输出到CICD，灰度验证，线上验收等尽可能多的场景，以服务于整个业务线。同样利用Ginkgo我们可以很容易做到这一点：
	* CICD: 在定义suite时，使用`RunSpecWithDefaultReporters`方法，可以让测试结果既输出到stdout，还可以输出一份Junit格式的报告。这样就可以通过类似Jenkins的工具方便的呈现测试结果，而不用任何其他的额外操作。
	* TaaS(Test as a Service): 通过`ginkgo build`或者原生的`go test -c`命令，可以方便的将测试用例，编译成package.test的二进制文件。如此的话，我们就可以方便的进行测试服务分发。典型的，如交付给SRE同学，辅助其应对线上灰度场景下的测试验收。所以在测试用例的组织上，这里有个小建议，过往我会看到有同学会习惯一个目录就定义一个suite文件，这样编译出的二进制文件就非常多，不利于分发。所以建议不要定义太多的suite，可以一条产品就一个suite入口，其他的用例包通过`_`导入进来。比如:
	![](https://img2018.cnblogs.com/blog/293394/201908/293394-20190818142458389-1740638586.png)

另外，值得说道的是，Ginkgo框架在提供强大功能和灵活性的同时，有些地方也需要使用者特别留心：

* `DescribeTable`功能是对TableDriven模式的友好支持，但它的原理是通过`Entry`在用例执行之前，通过反射机制来自动生成`It`方法，所以如果期望类似`BeforeEach+It`的原生组合来使用`BeforeEach+Entry`的话，可能在值类型的变量传递上，会不符合预期。其实，相较于`DescribeTable+Entry`的模式，我个人更倾向于通过方法+多个`It`的原生组合来写用例，虽然代码量显得有点多，但是用例表达的逻辑主题会更清晰，可读性较高。类似如下:
![](https://img2018.cnblogs.com/blog/293394/201908/293394-20190818174530257-1327970380.png)
* Ginkgo CLI的focus和skip命令非常好用，能够灵活的指定想执行或者排除的测试用例。不过要注意的是，focus和skip传入的是正则表达式，而适配这个正则的，是组成用例的所有的Container标题的组合(Suite+Describe+Context+It), 这些标题从外到里拼接成的完整字符串，所以使用时当注意。

## 都有谁在用Ginkgo?
Ginkgo的官方文档非常详细，非常利于使用。另外，我们看到著名的容器云项目[Kubernetes](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-testing/e2e-tests.md)也是使用Ginkgo框架来编写其e2e测试用例。

最后，如果您也使用Go语言来编写测试用例，不妨尝试下Ginkgo。
