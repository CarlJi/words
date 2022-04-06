## 前言

Kubernetes的成功少不了大量工程师的共同参与，而他们之间如何高效的协作，非常值得我们探究。最近研究和使用了他们的e2e测试和框架，还是挺有启发的。

## 怎样才是好的e2e测试？

不同的人写出的测试用例千差万别，尤其在用例，可能由开发人员编写的情形下，其情形可想而知。要知道，绝大多数开发人员，可能并没有经历过大量测试用例场景的熏陶。所以如何持续输出高质量的e2e测试用例，确实是一个挑战。不过，Kubernetes社区非常聪明，他们抽象出来了一些共性的东西，来希望大家遵守。比如说

1. 拒绝“flaky”测试 - 也就是那些偶尔会失败，但是又非常难定位的问题。
2. 错误输出要详细，尤其是做断言时，相关信息要有。不过也不要打印太多无效信息，尤其是在case并未失败的情况。
3. make case run in anywhere。这一点很重要，因为你的case是提交到社区，可能在各种环境下，各种时间段内运行。面对着各种cloud provider，各种系统负载情况。所以你的case要尽可能稳定，比如APICall，能异步的，就不要假设是同步; 比如多用retry机制等。
4. 测试用例要执行的足够快。超过两分钟，就需要给这种测试打上[SLOW]标签。而有这种标签的测试用例，可以运行的场景就比较有限制了。谁又不希望自己写的用例都被尽可能的执行呢？很有激励性的一条规则。

另外，社区不过定下规则，还开发和维护了一系列的基础设施，来辅助上面规则的落地。我们接下来要讲的e2e框架就是其中之一。

## e2e 验收测试

搞过测试的应该都知道，在面对复杂系统测试时，我们通常有多套测试环境，但是测试代码通常只有一份。所以为了能更好的区分测试用例，通常采取打标签的方式来给用例分类。这在Kubernetes的e2e里，这也不例外。

Kubernetes默认将测试用例分为下面几类，需要开发者在实际开发用例时，合适的使用。

- 没标签的，默认测试用例是稳定的，支持并发，且运行足够快的
- [Slow] 执行比较慢的用例.(对于具体的时间阈值，Kubernetes不同的文档表示不一致，此处需要修复)
- [Serial] 不支持并发的测试用例，比如占用太多资源，还比如需要重启Node的
- [Disruptive] 会导致其他测试用例失败或者具有破坏性的测试用例
- [Flaky] 不稳定的用例，且很难修复。使用它要非常慎重，因为常规CI jobs并不会运行这些测试用例
- [Feature:.+] 围绕特定非默认Kubernetes集群功能或者非核心功能的测试用例，方便开发以及专项功能适配

当然除了以上标签，还有个比较重要的标签就是[Conformance], 此标签用于验收Kubernetes集群最小功能集，也就是我们常说的MAT测试。所以如果你有个私有部署的k8s集群，就可以通过这套用例来搞验收。方法也很简单，通过下面几步就可以执行：

```
# under kubernetes folder, compile test cases and ginkgo tool
make WHAT=test/e2e/e2e.test && make ginkgo

# setup for conformance tests
export KUBECONFIG=/path/to/kubeconfig
export KUBERNETES_CONFORMANCE_TEST=y
export KUBERNETES_PROVIDER=skeleton

# run all conformance tests
go run hack/e2e.go -v --test --test_args="--ginkgo.focus=\[Conformance\]"
```

注意，kubernetes的测试使用的镜像都放在GCR上了，如果你的集群在国内，且还不带翻墙功能，那可能会发现pod会因为下载不了镜像而启动失败。

## Kubernetes e2e test framework

研究Kubernetes的e2e测试框架,然后类比我们以往的经验，个人觉得，下面几点特性还是值得借鉴的:

### All e2e compiled into one binary, 单一独立二进制

在对服务端程序进行API测试时，我们经常会针对每个服务都创建一个ginkgo suite来框定测试用例的范围，这样做的好处是用例目标非常清晰，但是随着服务数量的增多，这样的suite会越来越来多。从组织上，看起来就稍显杂乱，而且不利于测试服务的输出。

比如，我们考虑这么一个场景，QA需要对新机房部署，或者私有机房进行服务验证。这时候，就通常需要copy所有代码到指定集群在运行了，非常的不方便，而且也容易造成代码泄露。

kubernetes显然也会有这个需求，所以他们改变写法，将所有的测试用例都编译进一个e2e.test的二进制，这样针对上面场景时，就可以直接使用这个可执行文件来操作，非常的方便。

当然可执行文件的方便少不了外部参数的自由注入，以及整体测试用例的精心标记。否则，测试代码写的不规范，需要频繁的针对特定环境修改，也是拒不方便的。

### Each case has a uniqe namespace, 每个case拥有唯一的空间

为每条测试用例创建一个独立的空间，是kubernetes e2e framework的一大精华。每条测试用例独享一个空间，彼此不冲突，从而根本上避免并发困扰，借助ginkgo的CLI来运行，会极大的提高执行效率。

而且这处代码的方式也非常优美，很有借鉴价值:

```
func NewFramework(baseName string, options FrameworkOptions, client clientset.Interface) *Framework {
   f := &Framework{
      BaseName:                 baseName,
      AddonResourceConstraints: make(map[string]ResourceConstraint),
      Options:                  options,
      ClientSet:                client,
   }

   BeforeEach(f.BeforeEach)
   AfterEach(f.AfterEach)

   return f
}
```

利用ginkgo 的BeforeEach的嵌套特定，虽然在Describe下就定义framework的初始化(如下)，但是在每个It执行前，上面的BeforeEach才会真正执行，所以并不会有冲突:

```
var _ = framework.KubeDescribe("GKE local SSD [Feature:GKELocalSSD]", func() {
   f := framework.NewDefaultFramework("localssd")
   It("should write and read from node local SSD [Feature:GKELocalSSD]", func() {
   ...
   })
})
```

当然e2e框架还负责case执行完的环境清理，并且是按需灵活配置。比如你希望，case失败保留现场,不删除namespace，那么就可以设置flag 参数 delete-namespace-on-failure为false来实现。

### Asynchronous wait，异步等待

几乎所有的Kubernetes操作都是异步的，所以不管是产品代码还是测试用例，都广泛的使用了这个异步等待库：kubernetes/vendor/k8s.io/apimachinery/pkg/util/wait。这个库，实现简单，精悍，非常值得学习。

另外，针对测试的异步验证，其实ginkgo(gomega)本身提供的[Eventualy](http://onsi.github.io/gomega/#making-asynchronous-assertions)，也是非常好用的。

### Suitable logs，打印合适的log

Kubernetes e2e 主要使用两种方式输出log，一个是使用glog库，另一个则是framework.Logf方法。glog本身是golang官方提供的log库，使用比较灵活。但是这里主要推荐的还是Framework.Logf。因为使用此方法的log会输出到GinkgoWriter里面，这样当我们使用ginkgo.RunSpecsWithDefaultAndCustomReporters方法时，log不光输出到控制台，也会保存在junit格式的xml文件里，非常方便在jenkins里展示测试结果。

### Clean code, 测试代码也可以很干净，优美

很多时候大家会觉得测试代码比较low，其实却不然。代码无所谓优劣，好坏还是依赖写代码的人。而且我想说，测试代码也是可以，并且应该写的很优美的，不然如何提升逼格？!。

我们从Kubernetes e2e能看到很多好的借鉴，比如：

- 抽取主干方法，以突出测试用例主体
- 采用数据驱动方式书写共性测试用例
- 注释工整，多少适宜
- 不输出低级别log
- 代码行长短适宜
- 方法名定义清晰，可读性强

## Kubernetes环境普适性的e2e测试框架

现实中，如果需要围绕k8s工作，你可能需要一套，自己的测试框架。不管是测试各种自定义的controller or watcher，还是测试运行在k8s里运行的私有服务。这套框架都适用于你:

https://github.com/CarlJi/golearn/tree/master/src/carlji.com/experiments/k8s_e2e_mat_framework

逻辑改动很小，只是在原有kubernetes e2e 框架基础上抽取了最小集合。以方便快速使用。

是不是很贴心？



> 童鞋，点个赞吧(⊙o⊙)？

## 参考文档

- https://github.com/thtanaka/kubernetes/blob/master/docs/devel/writing-good-e2e-tests.md
- https://github.com/thtanaka/kubernetes/blob/master/docs/devel/e2e-tests.md

## Contact me ?

Email: jinsdu@outlook.com

Blog: <http://www.cnblogs.com/jinsdu/>

Github: <https://github.com/CarlJi>

------