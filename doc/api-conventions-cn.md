API Conventions
===============

*本文档面向希望更深入地了解
Kubernetes API 结构，以及想要扩展 Kubernetes API 的开发人员。
有关在 kubectl 中使用资源的介绍，请参见 [the object management overview](https://kubernetes.io/docs/concepts/overview/working-with-objects/object-management/).*

**目录**

- [Types (Kinds)](#types-kinds)
    - [Resources](#resources)
    - [Objects](#objects)
        - [Metadata](#metadata)
        - [Spec and Status](#spec-and-status)
            - [Typical status properties](#typical-status-properties)
        - [References to related objects](#references-to-related-objects)
        - [Lists of named subobjects preferred over maps](#lists-of-named-subobjects-preferred-over-maps)
        - [Primitive types](#primitive-types)
        - [Constants](#constants)
        - [Unions](#unions)
    - [Lists and Simple kinds](#lists-and-simple-kinds)
- [Differing Representations](#differing-representations)
- [Verbs on Resources](#verbs-on-resources)
    - [PATCH operations](#patch-operations)
- [Short-names and Categories](#short-names-and-categories)
    - [Short-names](#short-names)
    - [Categories](#categories)
- [Idempotency](#idempotency)
- [Optional vs. Required](#optional-vs-required)
- [Defaulting](#defaulting)
    - [Static Defaults](#static-defaults)
    - [Admission Controlled Defaults](#admission-controlled-defaults)
    - [Controller-Assigned Defaults (aka Late Initialization)](#controller-assigned-defaults-aka-late-initialization)
    - [What May Be Defaulted](#what-may-be-defaulted)
    - [Considerations For PUT Operations](#considerations-for-put-operations)
- [Concurrency Control and Consistency](#concurrency-control-and-consistency)
- [Serialization Format](#serialization-format)
- [Units](#units)
- [Selecting Fields](#selecting-fields)
- [Object references](#object-references)
    - [Naming of the reference field](#naming-of-the-reference-field)
    - [Referencing resources with multiple versions](#referencing-resources-with-multiple-versions)
    - [Handling of resources that do not exist](#handling-of-resources-that-do-not-exist)
    - [Validation of fields](#validation-of-fields)
    - [Do not modify the referred object](#do-not-modify-the-referred-object)
    - [Minimize copying or printing values to the referrer object](#minimize-copying-or-printing-values-to-the-referrer-object)
    - [Object References Examples](#object-references-examples)
        - [Single resource reference](#single-resource-reference)
            - [Controller behavior](#controller-behavior)
        - [Multiple resource reference](#multiple-resource-reference)
            - [Kind vs. Resource](#kind-vs-resource)
            - [Controller behavior](#controller-behavior-1)
        - [Generic object reference](#generic-object-reference)
            - [Controller behavior](#controller-behavior-2)
        - [Field reference](#field-reference)
            - [Controller behavior](#controller-behavior-3)
- [HTTP Status codes](#http-status-codes)
    - [Success codes](#success-codes)
    - [Error codes](#error-codes)
- [Response Status Kind](#response-status-kind)
- [Events](#events)
- [Naming conventions](#naming-conventions)
    - [Namespace Names](#namespace-names)
- [Label, selector, and annotation conventions](#label-selector-and-annotation-conventions)
- [WebSockets and SPDY](#websockets-and-spdy)
- [Validation](#validation)
- [Automatic Resource Allocation And Deallocation](#automatic-resource-allocation-and-deallocation)
- [Representing Allocated Values](#representing-allocated-values)
    - [When to use a <code>spec</code> field](#when-to-use-a-spec-field)
    - [When to use a <code>status</code> field](#when-to-use-a-status-field)
        - [Sequencing operations](#sequencing-operations)
    - [When to use a different type](#when-to-use-a-different-type)


[Kubernetes API](https://kubernetes.io/docs/concepts/overview/kubernetes-api/) 的约定(以及
ecosystem)旨在简化客户端开发并确保配置
可以实施适用于各种用例的机制
一贯。

Kubernetes API 的一般风格是 RESTful - 客户端创建、更新、
删除或通过标准 HTTP 谓词检索对象的描述
(POST、PUT、DELETE 和 GET) - 这些 API 优先接受并返回
JSON格式。Kubernetes 还公开了非标准动词的其他端点，并且
允许替代内容类型。所有 JSON 接受并返回
服务器有一个架构，由`kind`和`apiVersion`字段标识。哪里
存在相关的 HTTP 标头字段，它们应该镜像 JSON 的内容
字段，但信息不应仅在 HTTP 标头中表示。

定义了以下术语：

* **Kind** 特定对象模式的名称(例如，`Cat`和`Dog`
  种类将具有不同的属性和属性)
* **Resource** 系统实体的表示形式，以 JSON 形式发送或检索
  通过 HTTP 连接到服务器。资源通过以下方式公开：
    * 集合 - 相同类型的资源列表，可以查询
    * 元素 - 单个资源，可通过 URL 寻址
* **API Group** 一组一起公开的资源，以及
  在`apiVersion`字段中显示的版本为`GROUP/VERSION`，例如
  `policy.k8s.io/v1`。

每个资源通常接受并返回单一类型的数据。一种可能是
由反映特定用例的多个资源接受或返回。为
实例中，`Pod`类型被公开为允许最终用户的`pods`资源
创建、更新和删除 Pod，而单独的`Pod 状态`资源(即
作用于 `Pod` kind) 允许自动化进程更新字段的子集
在该资源中。

资源在 API 组中绑定在一起 - 每个组可以有一个或多个
独立于其他 API 组而演变的版本，以及
该组具有一个或多个资源。组名通常位于域名中
form - Kubernetes 项目保留使用空组，全部为单个
单词名称(`扩展`、`应用`)以及任何以`*.k8s.io`结尾的组名称
它的唯一用途。选择组名称时，我们建议选择子域
您的群组或组织拥有，例如`widget.mycompany.com`。

版本字符串应匹配
[DNS_LABEL](https://git.k8s.io/design-proposals-archive/architecture/identifiers.md)
格式。


资源集合应全部为小写和复数形式，而种类为
骆驼大小写和单数。组名称必须为小写且为有效的 DNS
子域。
## Types (Kinds)

种类分为三类：

1. **对象**表示系统中的持久实体。

    创建 API 对象是意图的记录 - 一旦创建，系统将 努力确保资源存在。所有 API 对象都具有通用元数据。

    一个对象可能具有多个客户端可用于执行的资源 创建、更新、删除或获取的特定操作。

    示例：`pod`、`ReplicationController`、`service`、`namespace`、`node`。

2. **列表**是一个或多个(通常)的**资源**的集合 (偶尔)种类。

    列表类型的名称必须以`List`结尾。列表具有一组有限的 通用元数据。所有列表都使用必需的`items`字段来包含数组 他们返回的对象。任何具有`items`字段的
    类型都必须是列表类型。

    系统中定义的大多数对象都应该有一个端点，该端点返回 完整的资源集，以及返回 完整列表。某些对象可能是单例(当前用户、系统 defaults)，并且可能没有列表。

    此外，所有返回带有标签的对象的列表都应支持 label 过滤(参见 [标签文档](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/))， 大多数列表应该支持按字段筛选(请参阅
   [字段文档](https://kubernetes.io/docs/concepts/overview/working-with-objects/field-selectors/))。

    示例：`PodList`、`ServiceList`、`NodeList`。

    请注意，`kubectl`和其他工具有时会输出资源集合 作为 `kind： List`。请记住，`kind： List`不是 Kubernetes API 的一部分;是的 在这些工
    具中公开客户端代码中的实现细节，用于 处理混合资源组。

3. **简单**类型用于对对象的特定操作和 非持久性实体。

    鉴于它们的范围有限，它们具有相同的一组有限的公共元数据 作为列表。

    例如，当发生错误时返回`Status`类型，而不是 在系统中持续存在。

    许多简单的资源都是`子资源`，它们植根于 API 路径 特定资源。当资源希望公开替代操作或视图时 与单个资源紧密耦合，则应使用 new 子资源。常见的子资源包括：

    * `/binding`：用于绑定代表用户请求的资源(例如 Pod、 PersistentVolumeClaim)添加到集群基础架构资源(例如，Node、 PersistentVolume)。
    * `/status`：用于仅写入资源的`status`部分。为 例如，`/pods`端点只允许更新`metadata`和`spec`， 因为这些反映了最终用户的意图。自动化 流程应该能够 通过将更新的 Pod 类型发送到服务器来修改用户要查看的状态 `/pods/&lt;name&gt;/status` 端点 - 备用端点允许 要应用于更新的不同规则，以及要适当访问的规则 限制。
    * `/scale`：用于读取和写入资源的计数，其方式是 与特定资源架构无关。

    另外两个子资源`proxy`和`portforward`提供对 群集资源，如 [访问集群](https://kubernetes.io/docs/tasks/access-application-cluster/access-cluster/)。

    标准 REST 谓词(定义如下)必须返回单数 JSON 对象。一些 API 端点可能会偏离严格的 REST 模式，并返回 不是单一的 JSON 对象，例如 JSON 对象的流或非结构化 文本日志数据。

    一组通用的`元`API 对象用于所有 API 组，并且是 因此被视为名为`meta.k8s.io`的 API 组的一部分。这些类型可能 独立于使用它们的 API 组而发展，API 
    服务器可能允许 它们将以其通用形式加以解决。例如`ListOptions`、 `DeleteOptions`、`List`、`Status`、`WatchEvent` 和 `Scale`。对于历史 
    这些类型是每个现有 API 组的一部分的原因。通用工具，如 配额、垃圾回收、自动缩放程序和通用客户端(如 kubectl) 利用这些类型在不同资源之间定义
    一致的行为 类型，例如编程语言中的接口。

    术语`种类`是为这些`顶级`API 类型保留的。术语`类型` 应用于区分对象或子对象中的子类别。

### Resources

API 返回的所有 JSON 对象必须具有以下字段：

* kind：标识此对象应具有的架构的字符串
* apiVersion：一个字符串，用于标识对象的架构版本 应该有

这些字段是正确解码对象所必需的。他们可能是 默认情况下，由服务器从指定的 URL 路径填充，但客户端 可能需要知道这些值才能构造 URL 路径。

### Objects

#### Metadata

每个对象类型都必须在嵌套对象字段中具有以下元数据 称为`元数据`：

* namespace：命名空间是 DNS 兼容的标签，对象被细分 到。默认命名空间为`default`。看 [命名空间文档](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/) 了解更多信息。
* name：在当前中唯一标识此对象的字符串 命名空间(参见 [标识符文档](https://kubernetes.io/docs/concepts/overview/working-with-objects/names/))。 检索单个对象时，此值在路径中使用。
* uid：时间和空间值的唯一值(通常为 RFC 4122 生成的 标识符，请参阅 [标识符文档](https://kubernetes.io/docs/concepts/overview/working-with-objects/names/)) 用于区分已删除的同名对象 并重新创建

每个对象都应在名为 `元数据`：

* resourceVersion：标识此对象内部版本的字符串 客户端可以使用它来确定对象何时发生更改。此值 客户端必须将其视为不透明，并未经修改地传递回服务器。 客户端不应假定资源版本具有跨 命名空间、不同类型的资源或不同的服务器。(见 [并发控制](#concurrency-control-and-consistency)，了解更多信息 详细信息。
* generation：表示特定世代的序列号 所需状态。由系统设置并单调增加，每个资源。五月 进行比较，例如 RAW 和 WAW 一致性。
* creationTimestamp：表示日期和时间的 RFC 3339 日期的字符串 已创建对象
* deletionTimestamp：表示日期和时间的 RFC 3339 日期的字符串 之后，此资源将被删除。此字段由服务器在以下情况下设置 正常删除由用户请求，并且不能由 客户。该资源将被删除(在资源列表中不再可见，并且 无法通过名称访问)之后，除非对象具有 终结器集。如果设置了终结器，则对象的删除是
  至少推迟到终结器被移除。 设置 deletionTimestamp 后，此值可能无法取消设置或进一步设置 将来，尽管它可能会被缩短或资源可能会被删除 在此之前。
* labels：字符串键和值的映射，可用于组织和 对对象进行分类(参见 [标签文档](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/))
* 注解：外部可以使用的字符串键和值的映射 用于存储和检索有关此对象的任意元数据的工具(请参阅 [注释文档](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/))

标签旨在供最终用户(选择容器)用于组织目的 与此标签查询匹配)。注释支持第三方自动化和 使用其他元数据装饰对象以供自己使用的工具。#### Spec and Status

按照惯例，Kubernetes API 对规范进行了区分 对象的所需状态(名为`spec`的嵌套对象字段)和 对象在当前时间的状态(名为 `status`)。该规范是对所需状态的完整描述， 包括用户提供的配置设置，
[默认值](#defaulting) 由系统扩展，并初始化属性 或在创建其他生态系统组件后以其他方式更改(例如， schedulers、auto-scalers)，并使用 API 在稳定存储中持久化 对象。如果删除规范，则该对象将从 系统。

`status`总结了系统中对象的当前状态，并且是 通常通过自动化过程与对象一起持久化，但可以生成 在飞行中。 作为一般准则，`状态`中的字段应该是最新的 对实际状态的观察，但它们可能包含诸如
为响应 对象的`spec`。 有关详细信息，请参见 [below](#representing-allocated-values) 详。

同时具有`spec`和`status`节的类型可以(并且通常应该)具有不同的节 它们的授权范围。 这允许用户被授予完全写入权限 访问`spec`和对状态的只读访问，而相关控制器是 授予对`spec`的只读访问权限，但对 status 的完全写入访问权限。

当对象的新版本被 POST 或 PUT 时，`规范`将更新，并且 立即可用。随着时间的流逝，系统将努力将`状态`带入 行与`spec`行。该系统将朝着最新的`规范`发展 无论该节的先前版本如何。例如，如果一个值是 在一个 PUT 中从 2 更改为 5，然后在另一个 PUT 中回落到 3 个 在将`状态`更改为 3 之前，不需要在 5 处`触摸底座`。在其他
换句话说，系统的行为是*基于水平*而不是*基于边缘*的。这 在缺少中间状态更改的情况下实现稳健行为。

Kubernetes API 也是声明式 系统的配置架构。为了方便基于等级 声明式配置的操作和表达式，字段 规范应具有声明性名称而不是命令式名称，并且 语义 -- 它们表示期望的状态，而不是旨在产生 所需状态。

对象上的 PUT 和 POST 谓词必须忽略`status`值，以避免 在读-修改-写场景中意外覆盖`状态`。`/status` 必须提供子资源，以使系统组件能够更新 他们管理的资源。

否则，PUT 需要指定整个对象。因此，如果一个字段 省略，假定客户端想要清除该字段的值。这 PUT 谓词不接受部分更新。仅修改对象的一部分 可以通过对资源进行 GETting、修改部分规范、标签或 注释，然后将其放回原处。看
[并发控制](#concurrency-control-and-consistency)，关于 使用此模式时的读-修改-写一致性。某些对象可能会暴露 允许状态变更的替代资源表示形式，或者 对对象执行自定义操作。

表示物理资源的所有对象，其状态可能与 用户期望的意图应该具有`规范`和`状态`。其状态的对象 不能与用户期望的意图不同 可以只有`规范`，并且可以重命名 `spec` 替换为更合适的名称。

同时包含`spec`和`status`的对象不应包含其他 标准元数据字段以外的顶级字段。

某些未保留在系统中的对象 - 例如`SubjectAccessReview` 和其他 webhook 样式调用 - 可以选择添加 `spec` 和 `status` 来封装 `呼叫和响应`模式。`spec`是请求(通常是对 information)，而`status`是响应。对于这些类似 RPC 的对象，唯一的
操作可能是 POST，但在提交和 响应降低了这些客户端的复杂性。 

##### Typical status properties

**条件** 为更高级别的状态报告提供标准机制 从控制器。它们是一种扩展机制，允许工具和其他 控制器，用于收集有关资源的摘要信息，而无需 了解特定于资源的状态详细信息。条件应与更多人相辅相成
有关由 控制器，而不是更换它。例如，一个 可以通过检查`readyReplicas`、`replicas`和 部署的其他属性。但是，`可用`条件允许 其他组件，以避免在部署中重复可用性逻辑 控制器。

对象可能会报告多个条件，并且新类型的条件可能是 将来添加或由第三方控制器添加。因此，条件是 使用对象列表/切片表示，其中每个条件都有相似的 结构。此集合应被视为键为`type`的映射。

当条件遵循一些一致的约定时，它们最有用：

* 应添加条件以显式传达用户和 组件关心而不是要求推断这些属性 从其他观察结果来看。 一旦定义，条件的含义就不能是 任意更改 - 它成为 API 的一部分，并且具有相同的向后 - 以及 API 任何其他部分的向前兼容性问题。

* 控制者应在第一次将其条件应用于资源时 访问资源，即使`状态`为`未知`。这允许其他 系统中的组件，以了解条件存在和控制器 在协调该资源方面正在取得进展。 

* 并非所有控制者都会遵守之前关于报告的建议 `未知`或`假`值。对于已知条件，缺少 条件`status`应与`Unknown`解释相同，并且 通常表示对帐尚未完成(或者 资源状态可能尚不可观察)。

* 对于某些条件，`True`表示正常操作，而对于某些条件，`True`表示正常操作 conditions，`False`表示正常操作。(`正常-真实`条件 有时说具有`正极性`和`正常-假`条件 据说具有`负极性`。无需进一步了解 条件，则无法计算条件的一般摘要 在资源上。

* 条件类型名称应该对人类有意义;既不积极也不 一般情况下，建议使用负极性。负面条件 像`MemoryExhausted`这样的人可能比 `足够的记忆`。相反，`就绪`或`成功`可能更容易 理解比`失败`，因为`失败=未知`或`失败=假`可能 引起双阴性混淆。

* 条件类型名称应描述当前观察到的状态 资源，而不是描述当前状态转换。这 通常表示名称应为形容词(`Ready`、`OutOfDisk`) 或过去时动词(`Succeeded`、`Failed`)而不是现在时动词 (`部署`)。中间状态可以通过设置 条件为`未知`。

* 对于需要较长时间的状态转换(例如，超过 1 分钟)，将转换本身视为观察到的 州。在这些情况下，条件(例如`调整大小`)本身不应 是瞬态的，应该使用 `True`/`False`/`Unknown`模式。这允许其他观察者确定 控制器的上次更新，无论是成功还是失败。在案例中
  状态转换无法完成并继续 对账不可行，应使用 Reason 和 Message 来 表示转换失败。

* 在为资源设计条件时，有一个共同的 顶级条件，总结了更详细的条件。简单 使用者可以简单地查询顶级条件。虽然他们不是 一致的标准，可以使用`就绪`和`成功`条件类型 由 API 设计人员分别用于长时间运行和有界执行对象。

条件应遵循 [k8s.io/apimachinery/pkg/apis/meta/v1/types.go](https://github.com/kubernetes/apimachinery/blob/release-1.23/pkg/apis/meta/v1/types.go#L1432-L1492) 中包含的标准架构。 它应该作为 status 中的顶级元素包含在内，类似于

```go
// +listType=map
// +listMapKey=type
// +patchStrategy=merge
// +patchMergeKey=type
// +optional
Conditions []metav1.Condition `json:"conditions,omitempty" patchStrategy:"merge" patchMergeKey:"type" protobuf:"bytes,1,rep,name=conditions"`
```

`metav1.`条件`包括以下字段
```go
// type of condition in CamelCase or in foo.example.com/CamelCase.
// +required
Type string `json:"type" protobuf:"bytes,1,opt,name=type"`
// status of the condition, one of True, False, Unknown.
// +required
Status ConditionStatus `json:"status" protobuf:"bytes,2,opt,name=status"`
// observedGeneration represents the .metadata.generation that the condition was set based upon.
// For instance, if .metadata.generation is currently 12, but the .status.conditions[x].observedGeneration is 9, the condition is out of date
// with respect to the current state of the instance.
// +optional
ObservedGeneration int64 `json:"observedGeneration,omitempty" protobuf:"varint,3,opt,name=observedGeneration"`
// lastTransitionTime is the last time the condition transitioned from one status to another.
// This should be when the underlying condition changed.  If that is not known, then using the time when the API field changed is acceptable.
// +required
LastTransitionTime Time `json:"lastTransitionTime" protobuf:"bytes,4,opt,name=lastTransitionTime"`
// reason contains a programmatic identifier indicating the reason for the condition`s last transition.
// Producers of specific condition types may define expected values and meanings for this field,
// and whether the values are considered a guaranteed API.
// The value should be a CamelCase string.
// This field may not be empty.
// +required
Reason string `json:"reason" protobuf:"bytes,5,opt,name=reason"`
// message is a human readable message indicating details about the transition.
// This may be an empty string.
// +required
Message string `json:"message" protobuf:"bytes,6,opt,name=message"`
```

将来可能会添加其他字段。

需要使用`原因`字段。

条件类型应在 PascalCase 中命名。短条件名称为 首选(例如，`Ready`而不是`MyResourceReady`)。

条件`status`值可以是`True`、`False`或`Unknown`。缺少 条件应与`未知`相同。 控制器如何处理 `未知`取决于相关条件。

围绕条件的思考随着时间的推移而发展，因此有几种 广泛使用的非规范性示例。

通常，条件值可能会来回变化，但某些条件 转换可能是单调的，具体取决于资源和条件类型。 然而，条件是观察结果，而不是状态机本身，也不是状态机 我们为对象定义了全面的状态机，也为相关的行为定义了
具有状态转换。该系统是基于电平的，而不是边沿触发的， 并且应该假设一个开放的世界。

振荡条件类型的一个示例是`Ready`，它表示 据信，该物体在上次探测时已完全运行。一个 可能的单调条件可能是`成功`。的`True`状态 `成功`意味着已完成，并且资源不再 积极。仍处于活动状态的对象通常具有`Succeeded` 状态为`未知`的条件。

v1 API 中的某些资源包含名为`phase`的字段，并且关联 `message`、`reason`和其他状态字段。使用`phase`的模式是 荒废的。较新的 API 类型应改用条件。阶段是 本质上是一个状态机枚举字段，这与 [system-design ] 相矛盾
原则](https://git.k8s.io/design-proposals-archive/architecture/principles.md#control-logic)和 阻碍了进化，因为 [添加新的枚举值会向后中断
兼容性](api_changes.md)。而不是鼓励客户推断 隐式属性，我们更愿意显式地暴露单个 客户端需要监视的条件。条件还具有以下好处： 有可能在所有方面创造一些具有统一含义的条件 资源类型，同时仍然公开特定资源类型所特有的其他资源类型
资源类型。 有关详细信息，请参见 [#7856](http://issues.k8s.io/7856) 和 讨论。

在条件类型中，以及它们在 API 中出现的其他任何位置，**`Reason`** 是 旨在用一个词，CamelCase 表示原因类别 当前状态，而 **`Message`** 旨在成为人类可读的短语 或句子，其中可能包含单个事件的具体细节。 `Reason`旨在用于简洁的输出，例如单行
`kubectl get` 输出，并总结原因的发生，而 `消息`旨在以详细的状态说明呈现给用户， 例如`kubectl describe`输出。 

历史信息状态(例如，上次转换时间、故障计数)为 仅通过合理的努力提供，不保证不会丢失。

状态信息可能很大(尤其是与 其他资源的集合，例如对其他对象的引用列表 -- 见下文)和/或快速变化，例如 [资源使用情况](https://git.k8s.io/design-proposals-archive/scheduling/resources.md#usage-data)，应分开 对象，可能来自原始对象的引用。这有助于
确保 GET 和手表在大多数情况下保持合理的效率 客户端，它们可能不需要该数据。

一些资源报告了`observedGeneration`，这是`世代`最多 最近由负责对 资源的所需状态。例如，这可用于确保 报告的状态反映了最新的所需状态。

#### References to related objects

对松散耦合对象集的引用，例如
[豆荚](https://kubernetes.io/docs/concepts/workloads/pods/) 由一个
[复制控制器](https://kubernetes.io/docs/concepts/workloads/controllers/replicationcontroller/)，
通常最好使用
[标签选择器](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#label-selectors)。为了
确保单个对象的 GET 在时间和空间上保持有限，这些
集合可以通过单独的 API 查询进行查询，但不会在
引用对象的状态。

有关对特定对象的引用，请参阅 [对象引用](#object-references)。

在以下情况下，可以允许以推荐人的`状态`提及推荐人
参考资料是一对一的，不需要经常更新，
特别是以基于边缘的方式。

#### Lists of named subobjects preferred over maps

在 [#2004](http://issue.k8s.io/2004) 和其他地方讨论过。有
任何 API 对象中都没有子对象的映射。相反，惯例是
使用包含名称字段的子对象列表。这些约定，以及
如何更改列表、结构和映射的语义是
在 Kubernetes 中进行了更详细的描述
[文档](https://kubernetes.io/docs/reference/using-api/server-side-apply/#merge-strategy)。

例如：

```yaml
ports:
  - name: www
    containerPort: 80
```

vs.

```yaml
ports:
  www:
    containerPort: 80
```

此规则保持所有 JSON/YAML 键都是 API 中的字段的不变性
对象。唯一的例外是 API 中的纯映射(目前，labels、
选择器、注释、数据)，而不是子对象集。

#### Primitive types

* 查看 API 中的类似字段(例如端口、持续时间)并按照
  现有字段的约定。
* 不要使用枚举。请改用字符串的别名(例如`NodeConditionType`)。
* 所有数值字段都应进行边界检查，无论是太小还是负数
  而且太大了。
* 所有公共整型字段必须使用 Go `int32` 或 Go `int64` 类型，而不是
  `int`(大小不明确，具体取决于目标平台)。 内部
  类型可以使用`int`。
* 对于整型字段，首选 `int32` 而不是 `int64`，除非您需要表示
  大于`int32`的值。 请参阅有关限制的其他准则
  `int64` 和语言兼容性。
* 不要使用无符号整数，因为跨语言和
  图书馆。如果是这种情况，只需验证整数是否为非负数即可。
* 所有数字(例如`int32`、`int64`)都通过 Javascript 转换为`float64`
  和其他一些语言，因此任何预计会超过该语言的字段
  幅度或精度(例如，整数值> 53 位)
  应序列化并接受为字符串。`int64`字段必须是
  边界检查为在`-(2^53) < x < (2^53)`范围内。
* 尽可能避免浮点值，切勿在规范中使用浮点值。
  浮点值无法可靠地进行往返跳闸(编码和
  重新解码)而不改变，并具有不同的精度和表示
  跨语言和架构。
* 对`布尔`字段三思而后行。许多想法都是从布尔开始的，但最终
  趋向于一小部分相互排斥的选项。 规划未来
  通过将策略选项显式描述为字符串类型进行扩展
  别名(例如`TerminationMessagePolicy`)。
#### Constants

某些字段将具有允许值(枚举)的列表。这些值将
是字符串，它们将采用 CamelCase 格式，首字母为大写字母。
示例：`ClusterFirst`、`Pending`、`ClientIP`。当首字母缩略词或首字母缩写时
首字母缩略词中的每个字母都应为大写，例如`ClientIP`或
`TCPDelay`。使用专有名称或命令行可执行文件的名称时
作为一个常量，专有名称应该用一致的大小写表示 -
示例：`systemd`、`iptables`、`IPVS`、`cgroupfs`、`Docker`(作为通用
concept)、`docker`(作为命令行可执行文件)。如果使用专有名称
它有混合大写，如`eBPF`，应该保留更长的时间
常量，例如`eBPFDelegation`。

Kubernetes 中的所有 API 都必须利用这种风格的常量，包括
标志和配置文件。如果以前使用不一致的常量，
新标志应仅为 CamelCase，并且随着时间的推移，旧标志应更新为
接受 CamelCase 值以及不一致的常量。示例：
Kubelet 接受值为`none`的 `--topology-manager-policy` 标志，
`尽力而为`、`受限`和`单 numa 节点`。此标志应接受
`None`、`BestEffort`、`Restricted`和`SingleNUMANode`。如果是新的
值添加到标志中，这两种形式都应受支持。
#### Unions

有时，最多可以设置一组字段中的一个。 例如，
PodSpec 的 [volumes] 字段有 17 个不同的特定于卷类型的字段，例如
如`NFS`和`iSCSI`。 集合中的所有字段都应为
[可选](#optional 与必需)。

有时，当创建新类型时，API 设计者可能会预期
将来将需要 union，即使最初只允许一个字段。
在这种情况下，请务必将字段设为 [Optional](#optional-vs-required)
在验证中，如果未设置唯一字段，则仍可能返回错误。做
不为该字段设置默认值。

### Lists and Simple kinds

每个列表或简单类型都应在嵌套对象中具有以下元数据
名为`元数据`的字段：

* resourceVersion：标识对象通用版本的字符串
  在列表中返回。客户端必须将此值视为不透明，并且
  未经修改地传递回服务器。资源版本仅在
  单一类型资源上的单个命名空间。

服务器返回的每个简单种类，以及发送到服务器的任何简单种类
必须支持幂等性或乐观并发性应返回此
价值。由于简单资源通常用作输入替代操作
修改对象时，简单资源的资源版本应对应
对象的资源版本。


## Differing Representations

对于不同的客户端，API 可以以不同的方式表示单个实体，或者
在系统中发生某些转换后转换对象。在这些
情况下，一个请求对象可能有两种不同的表示形式
资源，或不同种类。

一个例子是 Service，它表示用户对集合进行分组的意图
在公共端口上具有共同行为的 Pod。当 Kubernetes 检测到 Pod 时
匹配服务选择器，则将 Pod 的 IP 地址和端口添加到
该服务的终结点资源。仅当
服务存在，但仅公开所选 Pod 的 IP 和端口。这
全方位服务由两个不同的资源表示 - 在原始资源下
用户创建的服务资源，以及 Endpoints 资源中的服务资源。

再举一个例子，`pod status`资源可以接受带有`pod`的 PUT
种类，对可以更改的字段有不同的规则。

Kubernetes 的未来版本可能允许对对象进行替代编码
JSON格式。

## Verbs on Resources

API 资源应使用传统的 REST 模式：

* GET /&lt;resourceNamePlural&gt; - 检索类型的列表
  &lt;resourceName&gt;，例如 GET /pods 返回 Pod 列表。
* POST /&lt;resourceNamePlural&gt; - 从 JSON 对象创建一个新资源
  由客户提供。
* GET /&lt;resourceNamePlural&gt;/&lt;name&gt; - 检索单个资源
  替换为给定名称，例如 GET /pods/first 返回一个名为 `first` 的 Pod。应该是
  恒定时间，并且资源的大小应受限。
* DELETE /&lt;resourceNamePlural&gt;/&lt;name&gt; - 删除单个资源
  替换为给定名称。DeleteOptions 可以指定 gracePeriodSeconds，可选
  删除对象之前的持续时间(以秒为单位)。个别种类可能
  声明提供默认宽限期的字段，不同种类的字段可以
  具有不同种类范围的默认宽限期。用户提供的宽限期
  覆盖默认宽限期，包括零宽限期(`现在`)。
* DELETE /&lt;resourceNamePlural&gt; - 删除类型列表
  &lt;resourceName&gt;，例如 DELETE /pods 一个 Pod 列表。
* PUT /&lt;resourceNamePlural&gt;/&lt;name&gt; - 更新或创建资源
  替换为客户端提供的 JSON 对象的给定名称。是否
  可以使用 PUT 请求创建资源，具体取决于特定资源的
  存储策略配置，特别是`AllowCreateOnUpdate()`返回
  价值。大多数内置类型不允许这样做。
* PATCH /&lt;resourceNamePlural&gt;/&lt;name&gt; - 有选择地修改
  资源的指定字段。请参阅更多信息[下文](#patch-operations)。
* GET /&lt;resourceNamePlural&gt;&quest;watch=true - 接收 JSON 流
  对应于对给定类型的任何资源所做的更改的对象
  时间。
### PATCH operations

该 API 支持三种不同的 PATCH 操作，这些操作由它们的
对应的 Content-Type 标头：

* JSON 补丁，`内容类型：application/json-patch+json`
    * 根据 [RFC6902](https://tools.ietf.org/html/rfc6902) 中的定义，JSON 补丁是
      对资源执行的一系列操作，例如 `{`op`： `add`，
      `path`： `/a/b/c`， `value`： [ `foo`， `bar` ]}`.有关如何使用的更多详细信息
      JSON Patch，请参阅 RFC。
* 合并补丁，`Content-Type： application/merge-patch+json`
    * 根据 [RFC7386](https://tools.ietf.org/html/rfc7386) 中的定义，合并补丁
      实质上是资源的部分表示形式。提交的 JSON 是
      与当前资源`合并`以创建一个新资源，然后新的资源是
      保存。有关如何使用 Merge Patch 的更多详细信息，请参阅 RFC。
* 战略合并补丁，`内容类型：application/strategic-merge-patch+json`
    * Strategic Merge Patch 是 Merge Patch 的自定义实现。对于一个
      有关其工作原理以及为什么需要引入它的详细说明，请参阅
      [这里](/contributors/devel/sig-api-machinery/strategic-merge-patch.md)。

## Short-names and Categories

资源实现者可以选择包括`短名称`和类别
在为资源类型发布的发现信息中，
客户端在解决不明确的用户调用时可以用作提示。

对于编译的资源，它们由 REST 处理程序 `ShortNames() []string` 和 `Categories() []string` 实现控制。

对于自定义资源，这些资源由 CustomResourceDefinition 中的`.spec.names.shortNames`和`.spec.names.categories`字段控制。
### Short-names

注意：由于短名称发生冲突(相互冲突或与资源类型冲突)时的不可预知行为，
除非 API 审阅者特别允许，否则不要向内置资源添加新的短名称。查看问题
[#117742](https://issue.k8s.io/117742#issuecomment-1545945336) 和 [#108573](http://issue.k8s.io/108573)。

客户端可能会使用发现中列出的`短名称`作为提示，以解决对单个资源的不明确用户调用。

内置短名称的示例包括：

* `ds` -> `apps/v* daemonsets`
* `sts` -> `apps/v* statefulsets`
* `hpa` -> `autoscaling/v* horizontalpodautoscalers`

例如，如果仅提供内置 API 类型，则`kubectl get sts`等同于`kubectl get statefulsets.v1.apps`。

短名称匹配的优先级可能低于资源类型的完全匹配。
因此，使用短名称会增加集群中出现不一致行为的可能性
安装自定义资源时，如果这些自定义资源类型与短名称重叠。

继续上面的示例，如果在集群中安装了`.spec.names.plural`设置为`sts`的自定义资源，
`kubectl get sts`将改为检索自定义资源的实例。

### Categories

注意：由于类别与资源类型发生冲突时的行为不一致，
以及难以知道何时可以安全地将新资源添加到现有类别，
除非 API 审阅者特别允许，否则不要向内置资源添加新类别。
请参阅问题 [#7547](https://github.com/kubernetes/kubernetes/issues/7547#issuecomment-355835279)
[#42885](https://github.com/kubernetes/kubernetes/issues/42885#issuecomment-531265679)，
以及[添加到`全部`类别的注意事项](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-cli/kubectl-conventions.md#rules-for-extending-special-resource-alias---all)
举例说明遇到的困难。

客户端可以将发现中列出的类别用作提示，以解析用户对多个资源的调用。

内置类别及其映射到的资源示例包括：
* `api-extensions`
    * `apiregistration.k8s.io/v* apiservices`
    * `admissionregistration.k8s.io/v* mutatingwebhookconfigurations`
    * `admissionregistration.k8s.io/v* validatingwebhookconfigurations`
    * `admissionregistration.k8s.io/v* validatingadmissionpolicies`
    * `admissionregistration.k8s.io/v* validatingadmissionpolicybindings`
    * `apiextensions.k8s.io/v* customresourcedefinitions`
* `all`
    * `v1 pods`
    * `v1 replicationcontrollers`
    * `v1 services`
    * `apps/v* daemonsets`
    * `apps/v* deployments`
    * `apps/v* replicasets`
    * `apps/v* statefulsets`
    * `autoscaling/v* horizontalpodautoscalers`
    * `batch/v* cronjobs`
    * `batch/v* jobs`

对于上述类别，并且仅提供内置 API 类型，`kubectl get all`将等同于
`kubectl get pods.v1.，replicationcontrollers.v1.，services.v1.，daemonsets.v1.apps，deployments.v1.apps，replicasets.v1.apps，statefulsets.v1.apps，horizontalpodautoscalers.v2.autoscaling，cronjobs.v1.batch，jobs.v1.batch，`.
## Idempotency

所有兼容的 Kubernetes API 都必须支持`名称幂等性`，并使用
HTTP 状态代码 409，当请求对具有
与系统中的现有对象同名。看
[标识符文档](https://kubernetes.io/docs/concepts/overview/working-with-objects/names/)
了解详情。

可以使用`metadata.generateName`请求系统生成的名称。
GenerateName 指示服务器应先使该名称唯一
坚持下去。该字段的非空值表示服务器应
尝试使名称唯一(并且返回给客户端的名称将是
与传递的名称不同)。此字段的值将与
服务器上的随机后缀(如果尚未提供 Name 字段)。这
提供的值必须在 Name 的规则内有效，并且可以被截断
后缀的长度。如果指定了此字段，并且名称不存在，
如果生成的名称，服务器将返回原因为`AlreadyExists`的 409
存在，客户端应重试(至少在等待
在 Retry-After 标头中指示(如果存在)。

## Optional vs. Required

字段必须是可选字段或必填字段。

可选字段具有以下属性：

- 它们在 Go 中有 `+optional` 注释标签。
- 它们是 Go 定义中的指针类型(例如 `AwesomeFlag *SomeFlag`)或
  具有内置的`nil`值(例如地图和切片)。
- API 服务器应允许使用此字段对资源进行 POST 和 PUTING 操作
  未凝固的。

在大多数情况下，可选字段还应具有 `omitempty` struct 标记(
`omitempty`选项指定应从 JSON 中省略该字段
如果字段的值为空，则进行编码)。但是，如果您想拥有
未提供的可选字段与未提供的可选字段的逻辑不同
空值，不要使用`omitempty`(例如 https://github.com/kubernetes/kubernetes/issues/34641)。

请注意，为了向后兼容，任何具有`omitempty`结构的字段
标记将被视为可选，但这在将来可能会更改，并且
强烈建议使用`+可选`注释标签。

必填字段具有相反的属性，即：

- 它们没有`+可选`评论标签。
- 它们没有`omitempty`结构标签。
- 它们不是 Go 定义中的指针类型(例如`AnotherFlag、SomeFlag`)。
- API 服务器不应允许使用此字段对资源进行 POST 或 PUTing 操作
  未凝固的。

使用 `+optional` 或 `omitempty` 标记会导致 OpenAPI 文档
反映该字段是可选的。

使用指针可以将 unset 与该类型的零值区分开来。
在某些情况下，原则上不需要指针
可选字段，因为零值是禁止的，因此意味着未设置。那里
是代码库中的示例。然而：

- 对于实施者来说，可能很难预测所有情况，其中
  可能需要将值与零值区分开来
- 编码器输出中不会省略结构，即使指定了 omitempty，
  这是凌乱的;
- 对于 Go 用户来说，始终暗示可选的指针更清晰
  语言客户端，以及使用相应类型的任何其他客户端

因此，我们要求指针始终与不这样做的可选字段一起使用
具有内置的`nil`值。


## Defaulting

一般来说，我们希望在我们的 API 中显式表示默认值，
而不是断言`未指定的字段获得默认行为`。 这
很重要，因此：
- 默认值可能会在较新的 API 版本中演变和更改
- 存储的配置描述了完整的所需状态，使其更容易
  供系统确定如何实现状态，并供用户
  知道会发生什么

有 3 种不同的方法可以在创建或
更新(包括修补和应用)资源：

1. static：基于请求的 API 版本以及可能的其他字段
   resource，字段可以在 API 调用期间分配值
2. 准入控制：基于配置的准入控制器和
   可能在集群内或集群外的其他状态，可以分配字段
   API 调用期间的值
3.控制者：任意更改(在允许的范围内)可以
   在 API 调用完成后对资源进行

在决定使用哪种机制和管理
语义学。

### Static Defaults

静态默认值特定于每个 API 版本。 默认字段
使用`v1`API 创建对象时应用的值可能与
使用`v2`API 时应用的值。 在大多数情况下，这些值是
由 API 版本定义为文本值(例如，`如果此字段不是
指定它默认为 0`)。

在某些情况下，这些值可能是有条件的或确定性派生的
从其他字段(例如，`如果 otherField 为 X，则此字段默认为 0`或
`此字段默认为 otherField 的值`)。 请注意，此类派生
默认值在面对更新时会带来危险 - 如果`其他`字段
更改时，派生字段可能也必须更改。 静态默认值
逻辑不知道更新，也没有`先前值`的概念，这意味着
这种字段间关系成为用户的问题 - 他们必须更新
他们关心的领域和`其他`领域。

在极少数情况下，这些值可以从某个池中分配或确定
通过其他方法(例如，服务的 IP 和 IP 系列相关字段需要
请考虑其他配置设置)。

这些值由 API 服务器在解码时同步应用
版本化数据。 对于 CREATE 和 UPDATE 操作，这是相当的
直截了当 - 当 API 服务器收到(版本控制)请求时，
在进行任何进一步处理之前，将立即应用默认值。 当
API 调用完成时，将设置并存储所有静态默认值。
资源的后续 GET 将显式包含默认值。
但是，当从存储中读取对象时，静态默认值也适用(即
GET 操作)。 这意味着，当有人获取`较旧`的存储对象时，
自存储该对象以来添加到 API 的任何字段都将
默认，并根据存储的 API 版本返回。

静态默认值是逻辑上需要的值的最佳选择，
但它们的价值对大多数用户来说都很好。 静态默认值
不得考虑除作对象以外的任何状态(并且
服务 API 的复杂性就是一个例子)。

可以使用`+default=`标记在字段上指定默认值。原
将直接分配它们的值，而结构将通过
JSON 解组过程。没有`omitempty`json 标记的字段将
如果未分配默认值，则默认为其相应类型的零值。

参考 [defaulting docs](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/#defaulting)
了解更多信息。

### Admission Controlled Defaults

在某些情况下，设置一个不是派生自
有问题的对象。 例如，在创建 PersistentVolumeClaim 时，
必须指定存储类型。 对于许多用户来说，最好的答案是
`无论集群管理员为默认值决定了什么`。 StorageClass 是一个
与 PersistentVolumeClaim 不同的 API，以及哪一个表示为
默认值可能随时更改。 因此，这不符合静态条件
违约。

相反，我们可以提供一个内置的准入控制器或
MutatingWebhookConfiguration。 与静态默认值不同，这些可能会考虑
外部状态(例如 StorageClass 对象上的注释)时确定
默认值，并且必须处理诸如争用条件之类的事情(例如
StorageClass 被指定为默认值，但准入控制器尚未指定
但看到了那个更新)。 这些准入控制器是严格可选的，并且
可以禁用。 因此，以这种方式初始化的字段必须是
严格可选。

与静态默认值一样，它们与
问题，当 API 调用完成时，所有静态默认值都将是
设置。 资源的后续 GET 将包含默认值
明确地。

### Controller-Assigned Defaults (aka Late Initialization)

延迟初始化是指资源字段由系统控制器设置
创建/更新对象(异步)后。 例如，
scheduler 在创建 pod 后设置 pod.spec.nodeName 字段。 它
称其为`默认`有点牵强，但由于它是如此普遍和有用，它是
包括在这里。

与准入控制默认值一样，这些控制器可以考虑外部
状态 在决定设置什么值时，必须处理争用条件，并且可以
禁用。 以这种方式初始化的字段必须是严格可选的
(这意味着观察者将看到没有设置这些字段的对象，即
允许且语义正确)。

与所有控制器一样，必须注意不要破坏不相关的字段或
值(例如在数组中)。 使用修补程序或应用机制之一
建议用于促进控制器的组合和并发。

### What May Be Defaulted

所有形式的违约都应仅进行以下类型的修改：
- 设置以前未设置的字段
- 向地图添加键
- 向具有可合并语义的数组添加值
  (类型定义中的 `+listType=map` 标签或 `patchStrategy：`merge`` 属性)

特别是，我们永远不想更改或覆盖
用户。 如果他们请求无效的内容，他们应该得到一个错误。

这些规则确保：
1. 用户(具有足够的权限)可以覆盖任何系统默认值
   通过显式设置本来可以设置的字段来执行的行为
   违约
1.用户的更新可以与默认值合并

### Considerations For PUT Operations

一旦创建了一个对象并应用了默认值，它就非常
随着时间推移而进行更新很常见。 Kubernetes 提供了多种方式
更新对象，该对象在字段以外的字段中保留现有值
正在更新(例如，战略合并补丁和服务器端应用)。 有
但是，更新对象的不太明显的方法可能会产生不良交互
默认值 - PUT(又名`kubectl replace`)。

目标是，对于给定的输入(例如 YAML 文件)，对现有对象执行 PUT 操作
应产生与使用该输入创建对象相同的结果。
使用相同的输入第二次调用 PUT 应该是幂等的，并且应该
不更改资源。 即使是读-修改-写循环也不是完美的
面对版本偏差的解决方案。

当使用 PUT 更新对象时，API 服务器将看到`旧`对象
使用以前分配的默认值和具有新分配的`新`对象
违约。 对于静态默认值，如果 CREATE 和 PUT
使用了不同的 API 版本。 例如，API 的`v1`可能默认字段
设置为`false`，而`v2`默认为`true`。 如果对象是通过 API 创建的
v1 (field = `false`)，然后通过 API v2 替换，该字段将尝试
更改为`true`。 当分配值或
派生自相关对象之外的源(例如服务 IP)。

对于某些 API，这是可接受且可操作的。 对于其他人来说，这可能是
验证不允许。 在后一种情况下，用户将收到以下错误
尝试更改其 YAML 中甚至不存在的字段。 这是
添加新字段时尤其危险 - 旧客户甚至可能不知道
关于字段的存在，甚至使读取-修改-写入循环失败。
虽然它是`正确的`(从某种意义上说，这确实是他们所要求的
PUT)，它没有帮助，而且用户体验不好。

添加具有静态或准入控制默认值的字段时，必须
考虑。 如果字段在创建后是不可变的，请考虑将逻辑添加到
手动将值从`旧`对象`修补`到`新`对象中，当它有
被`未设置`，而不是返回错误或分配不同的值
(例如 服务 IP)。 这通常是用户的意思，即使它
不是他们说的。 这可能需要以不同的方式设置默认值
(例如 在理解更新的注册表代码中，而不是在
版本控制的默认代码，但未)。 小心检测和报告
指定了`new`值但与
`旧`值。

对于控制器默认的字段，情况更加令人不快。
控制器没有机会在 API 之前`修补`值
操作已提交。 如果允许`unset`值，则它将被保存，
任何监视客户端都将收到通知。 如果不允许使用`unset`值，或者
否则不允许突变，用户将收到错误，并且有
我们对此无能为力。

## Concurrency Control and Consistency

Kubernetes 利用 *资源版本* 的概念来实现乐观
并发。所有 Kubernetes 资源都有一个 `resourceVersion` 字段作为
他们的元数据。此 resourceVersion 是一个字符串，用于标识内部
客户端可以使用的对象的版本，以确定对象何时具有
改变。当记录即将更新时，将根据
预保存的值，如果不匹配，更新将失败并显示 StatusConflict
(HTTP 状态代码 409)。

每次修改对象时，服务器都会更改 resourceVersion。
如果 resourceVersion 包含在 PUT 操作中，则系统将验证
在资源期间没有其他成功的突变
读/修改/写周期，通过验证 resourceVersion 的当前值
与指定的值匹配。

resourceVersion 目前由 [etcd 的
mod_revision](https://etcd.io/docs/latest/learning/api/#key-value-pair).
但是，需要注意的是，应用程序不应依赖
Kubernetes 维护的版本控制系统的实现细节。我们可能
更改 resourceVersion 的实现，例如更改它
添加到时间戳或每个对象的计数器。

客户端知道 resourceVersion 的预期值的唯一方法是
已从服务器收到它以响应先前的操作，通常是
获取。客户端必须将此值视为不透明，并未经修改地传回
到服务器。客户端不应假定资源版本具有含义
跨命名空间、不同类型的资源或不同的服务器。
目前，resourceVersion 的值设置为与 etcd 的 sequencer 匹配。你
可以将其视为 API 服务器可用于对请求进行排序的逻辑时钟。
但是，我们预计 resourceVersion 的实现会在
future，例如我们按种类和/或命名空间或端口对状态进行分片
复制到另一个存储系统。

在发生冲突的情况下，此时正确的客户端操作是获取
资源，重新应用更改，然后再次尝试提交。这
机制可用于防止如下争用：

```
Client #1                                  Client #2
GET Foo                                    GET Foo
Set Foo.Bar = "one"                        Set Foo.Baz = "two"
PUT Foo                                    PUT Foo
```

当这些序列并行发生时，更改为 Foo.Bar 或
更改为 Foo.Baz 可能会丢失。

另一方面，当指定 resourceVersion 时，其中一个 PUT 将
失败，因为无论哪个写入成功，都会更改 Foo 的 resourceVersion。

resourceVersion 可以用作其他操作的前提条件(例如，GET、
DELETE)，例如在存在的情况下进行先写后读一致性
的缓存。

`Watch`操作使用查询参数指定 resourceVersion。它被使用
指定开始监视指定资源的时间点。这
可用于确保在资源的 GET 之间不会遗漏任何突变
(或资源列表)和后续的 Watch，即使当前版本的
该资源是较新的。这是目前列出的主要原因
operations(集合上的 GET)返回 resourceVersion。


## Serialization Format

API 可以返回任何资源的替代表示形式，以响应
接受标头或在备用终结点下，但默认序列化
API 响应的输入和输出必须是 JSON。

内置资源也接受 protobuf 编码。因为 proto 不是
自称，有一个信封包装器描述了
内容。

所有日期都应序列化为RFC3339字符串。

## Units

单位必须在字段名称中显式(例如，`timeoutSeconds`)，或者
必须指定为值的一部分(例如，`resource.数量`)。哪
首选方法是待定，尽管目前我们使用`fooSeconds`
持续时间的约定。

工期字段必须表示为整型字段，单位为
字段名称的一部分(例如`leaseDurationSeconds`)。我们不使用持续时间
，因为这将要求客户端实现 go 兼容解析。

## Selecting Fields

某些 API 可能需要识别 JSON 对象中的哪个字段无效，或者
引用要从单独资源中提取的值。当前
建议使用标准的 JavaScript 语法来访问该字段，
假设 JSON 对象已转换为 JavaScript 对象，但没有
前导点，例如`metadata.name`。

Examples:

* 在数组中第二项的对象`state`中找到字段`current`
  `字段`： `字段[1].状态.当前`

## Object references

命名空间类型的对象引用通常应仅引用
相同的命名空间。 由于命名空间是安全边界，因此跨命名空间
引用可能会产生意想不到的影响，包括：
1. 将一个命名空间的信息泄露到另一个命名空间中。放置状态消息甚至一些
   有关原始对象的内容。这是跨命名空间的问题。
2. 对其他命名空间的潜在入侵。通常，参考文献可以访问一条参考信息，因此
   能够表达`给我那边那个`在命名空间中是危险的，而无需额外的权限检查工作
   或从两个涉及的命名空间中选择加入。
3.一方无法解决的参照完整性问题。从 namespace/A 引用 namespace/B 并不意味着
   控制其他命名空间的权力。这意味着您可以引用无法创建或更新的事物。
4.删除语义不明确。如果命名空间资源被其他命名空间引用，则应删除
   引用的资源会导致删除，或者引用的资源应被强制保留。
5.创作语义不明确。如果引用的资源是在引用之后创建的，则无法知道它是否
   是预期的，或者是使用相同名称创建的不同。

内置类型和 ownerReferences 不支持跨命名空间引用。
如果非内置类型选择具有跨命名空间引用，则上述边缘情况的语义应为
明确描述，权限问题应得到解决。
这可以通过双重选择加入(推荐人和被推荐人的选择加入)或使用辅助权限来完成
入院时进行的检查。

### Naming of the reference field

引用字段的名称应采用`{field}Ref`格式，后缀中始终包含`Ref`。

应命名`{field}`组件以指示引用的目的。例如，`targetRef`
终结点指示对象引用指定目标。

可以让`{field}`组件指示资源类型。例如，引用时的`secretRef`
一个秘密。但是，如果字段扩展到
引用多个类型。

对于对象引用列表，该字段的格式应为`{field}Refs`，并具有相同的指导
如上图所示。

### Referencing resources with multiple versions

大多数资源将具有多个版本。例如，核心资源
在从 alpha 过渡到 GA 时，将进行版本更改。

控制器应假定资源的版本可能会更改，并包括适当的错误处理。

### Handling of resources that do not exist

在多种情况下，所需资源可能不存在。示例包括：

- 所需的资源版本不存在。
- 集群引导过程中的争用条件导致尚未添加资源。
- 用户错误。

编写控制器时应假设引用的资源可能不存在，并包括
错误处理，使用户清楚问题。

### Validation of fields

对象引用中使用的许多值都用作 API 路径的一部分。例如
对象名称在路径中用于标识对象。未经消毒，这些值可用于
尝试检索其他资源，例如使用具有语义含义的值，例如`..` 或 `/`。

让控制器在将字段用作 API 请求中的路径段之前对其进行验证，并向
告知用户验证失败。

请参见[对象名称和ID](https://kubernetes.io/docs/concepts/overview/working-with-objects/names/)
有关合法对象名称的详细信息。

### Do not modify the referred object

若要最大程度地减少潜在的权限提升向量，请不要修改所引用的对象，
或限制对同一命名空间中对象的修改，并限制允许的修改类型
(例如，HorizontalPodAutoscaler 控制器仅写入`/scale`子资源)。

### Minimize copying or printing values to the referrer object

由于控制器的权限可能与对象作者的权限不同
控制器正在管理，对象的作者可能没有权限
查看参照对象。因此，将有关引用对象的任何值复制到
反向链接对象可以被视为权限升级，使用户能够读取他们的值
以前无法访问。

同样的方案也适用于将有关引用对象的信息写入事件。

通常，不要将从引用对象检索到的信息写入或打印到规范、其他对象或日志中。

必要时，请考虑这些值是否是
引用对象的作者可以通过其他方式访问(例如，已经需要
正确填充对象引用)。

### Object References Examples

以下各节说明了各种对象引用方案的建议架构。

下面概述的架构旨在启用纯加法字段作为可引用的类型
对象扩展，因此向后兼容。

例如，可以从单个资源类型转到多个资源类型，而无需
架构中的重大更改。

#### Single resource reference

单一类型的对象引用很简单，因为控制器可以对标识对象所需的大多数限定符进行硬编码。因此，唯一需要提供的值是名称(和命名空间，尽管不鼓励跨命名空间引用)：
```yaml
# for a single resource, the suffix should be Ref, with the field name
# providing an indication as to the resource type referenced.
secretRef:
    name: foo
    # namespace would generally not be needed and is discouraged,
    # as explained above.
    namespace: foo-namespace
```

仅当意图始终仅引用单个资源时，才应使用此架构。
如果可以扩展到多个资源类型，请使用 [multiple resource reference](#multiple-resource-reference)。

##### Controller behavior

操作员应知道从中检索值所需的对象的版本、组和资源名称，并且可以使用发现客户端或直接构造 API 路径。
#### Multiple resource reference

当存在一组可供引用指向的有效资源类型时，将使用多类对象引用。

与单类对象引用一样，运算符可以提供缺失的字段，前提是存在的字段足以在一组受支持的类型中唯一标识对象资源类型。

```yaml
# guidance for the field name is the same as a single resource.
fooRef:
    group: sns.services.k8s.aws
    resource: topics
    name: foo
    namespace: foo-namespace
```

虽然并不总是需要帮助控制器识别资源类型，但包含`组`是为了避免资源存在于多个组中时出现歧义。它还为最终用户提供了清晰的信息，并允许复制粘贴引用，而不会因处理引用的不同控制器而更改引用类型。
##### Kind vs. Resource

对象引用中一个常见的混淆点是是否构造
带有`kind`或`resource`字段的引用。从历史上看，大多数对象
Kubernetes 中的引用使用了`kind`。这不像`资源`那样精确。
尽管`组`和`资源`的每个组合在
Kubernetes 对于`组`和`种类`并不总是如此。这是可能的
让多个资源使用相同的`种类`。

通常，Kubernetes 中的所有对象都有一个规范的主资源，例如
`pods`表示创建和删除 `Pod` 模式资源的方式。
虽然可能无法直接创建资源架构，例如
`Scale`对象，该对象仅在多个
工作负载，大多数对象引用通过其架构处理主资源。
在对象引用的上下文中，`kind`指的是架构，而不是
资源。

如果对象引用的实现将始终具有清晰的映射方式
kinds 到资源，在对象引用中使用 `kind` 是可以接受的。在
一般情况下，这要求实现在
种类和资源(使用`kind`的内置引用就是这种情况)。
依赖动态 kind 到资源的映射是不安全的。即使只有`种类`
最初动态映射到单个资源，也可以用于另一个资源
要挂载的资源引用相同的`种类`，可能会破坏任何
动态资源映射。

如果对象引用可用于引用任意类型的资源，并且
kind 和 resource 之间的映射可能是模棱两可的，`resource`应该是
在对象引用中使用。

Ingress API 提供了一个很好的例子，说明`kind`对于
对象引用。API 支持后端引用作为扩展点。
实现可以使用它来支持将流量转发到自定义目标
例如存储桶。重要的是，支持的目标类型显然是
由 API 的每个实现定义，并且没有歧义
资源类型映射到。这是因为每个 Ingress 实现都有一个
kind 到资源的硬编码映射。

如果上面的对象引用改用`kind`，则如下所示
`资源`的：

```yaml
fooRef:
    group: sns.services.k8s.aws
    kind: Topic
    name: foo
    namespace: foo-namespace
```

##### Controller behavior

操作员可以将 (group，resource) 的映射存储到所需的资源版本。从那里，它可以构造资源的完整路径，并检索对象。

也可以让控制器选择它通过发现客户端找到的版本。但是，由于架构可能因不同版本而异
资源，控制器还必须处理这些差异。

#### Generic object reference

当需要提供指向某个对象的指针以简化用户的发现时，将使用泛型对象引用。例如，这可用于引用发生的`core.v1.Event`的目标对象。

使用通用对象引用时，除了标准对象(例如 ObjectMeta)之外，无法提取有关引用对象的任何信息。由于任何标准字段都存在于资源的任何版本中，因此在这种情况下可以不包括版本：

```yaml
fooObjectRef:
    group: operator.openshift.io
    resource: openshiftapiservers
    name: cluster
    # namespace is unset if the resource is cluster-scoped, or lives in the
    # same namespace as the referrer.
```

##### Controller behavior

操作员应通过发现客户端查找资源(因为未提供版本)。由于任何可检索字段对所有对象都是通用的，因此任何版本的资源都应该这样做。
#### Field reference

当需要从引用对象中的特定字段中提取值时，使用字段引用。

字段引用不同于其他引用类型，因为操作员在引用之前对对象一无所知。由于对象的架构对于资源的不同版本可能不同，这意味着这种类型的引用需要`版本`。

```yaml
fooFieldRef:
   version: v1 # version of the resource
   # group is elided in the ConfigMap example, since it has a blank group in the OpenAPI spec.
   resource: configmaps
   fieldPath: data.foo
```

fieldPath 应指向单个值，并使用 [推荐的字段选择器表示法](#selecting-fields) 来表示字段路径。

##### Controller behavior

在此方案中，用户将提供所有必需的路径元素：组、版本、资源、名称，可能还有命名空间。
因此，控制器可以在不使用发现客户端的情况下构造 API 前缀并对其进行查询：

```
/apis/{group}/{version}/{resource}/
```

## HTTP Status codes

服务器将使用与 HTTP 规范匹配的 HTTP 状态代码进行响应。请参阅
部分，了解服务器将发送的状态代码类型。

API 可能会返回以下 HTTP 状态代码。

#### Success codes

* `200 状态正常`
    * 表示请求已成功完成。
* `201 状态已创建`
    * 表示创建 kind 的请求已成功完成。
* `204 状态无内容`
    * 表示请求成功完成，响应包含
      没有身体。
    * 在响应 HTTP OPTIONS 请求时返回。

#### Error codes

* `307 状态临时重定向`
    * 表示所请求资源的地址已更改。
    * 建议的客户端恢复行为：
        * 遵循重定向。

* `400 StatusBadRequest`
    * 表示请求无效。
    * 建议的客户端恢复行为：
        * 不要重试。修复请求。

* `401 状态未授权`
    * 表示可以访问并理解服务器请求，但
      拒绝采取任何进一步行动，因为客户必须提供
      授权。如果客户端已提供授权，则服务器是
      表示提供的授权不合适或无效。
    * 建议的客户端恢复行为：
        * 如果用户未提供授权信息，请提示他们
          相应的凭据。如果用户提供了授权信息，
          通知他们其凭据被拒绝，并选择再次提示他们。

* `403 状态禁止`
    * 表示可以访问并理解服务器请求，但
      拒绝采取任何进一步的操作，因为它被配置为拒绝访问
      客户端请求的资源的一些原因。
    * 建议的客户端恢复行为：
        * 不要重试。修复请求。

* `404 状态未找到`
    * 表示请求的资源不存在。
    * 建议的客户端恢复行为：
        * 不要重试。修复请求。

* `405 StatusMethodNotAllowed`
    * 指示客户端尝试对资源执行的操作
      代码不支持。
    * 建议的客户端恢复行为：
        * 不要重试。修复请求。

* `409 状态冲突`
    * 表示客户端尝试创建的资源已
      存在，或者由于冲突而无法完成请求的更新操作。
    * 建议的客户端恢复行为：
        * 如果创建新资源：
            * 更改标识符并重试，或 GET 并比较
              字段，并发出 PUT/update 以修改现有对象
              对象。
        * 如果更新现有资源：
            * 请参阅下面`状态`响应部分中的`冲突`，了解如何
              检索有关冲突性质的详细信息。
            * GET 并比较预先存在的对象中的字段，合并更改(如果
              根据前提条件仍然有效)，然后使用更新的请求重试
              (包括`ResourceVersion`)。

* `410 状态消失`
    * 表示该项目在服务器上不再可用，并且没有
      转发地址是已知的。
    * 建议的客户端恢复行为：
        * 不要重试。修复请求。

* `422 StatusUnprocessableEntity`
    * 表示无法完成请求的创建或更新操作
      由于作为请求的一部分提供的数据无效。
    * 建议的客户端恢复行为：
        * 不要重试。修复请求。

* `429 状态太多请求`
    * 表示已超出客户端速率限制或
      服务器收到的请求多于其可处理的请求数。
    * 建议的客户端恢复行为：
        * 从响应中读取`Retry-After`HTTP 标头，并至少等待
          在重试之前很久。

* `500 StatusInternalServerError`
    * 表示可以访问并理解服务器请求，但
      发生意外的内部错误，调用的结果是
      未知，或者服务器无法在合理的时间内完成操作(这可能
      由于临时服务器负载或与他人的暂时性通信问题
      服务器)。
    * 建议的客户端恢复行为：
        * 使用指数退避重试。

* `503 StatusServiceUnavailable`
    * 表示所需的服务不可用。
    * 建议的客户端恢复行为：
        * 使用指数退避重试。

* `504 StatusServerTimeout`
    * 表示无法在给定时间内完成请求。
      只有当客户端在
      请求。
    * 建议的客户端恢复行为：
        * 增加超时参数的值，并以指数重试
          退避。

## Response Status Kind

Kubernetes 将始终从任何 API 端点返回 `Status` 类型，当
发生错误。客户端应在适当的时候处理这些类型的对象。

在两种情况下，API 将返回`Status`类型：
* 当操作不成功时(即当服务器返回非
  2xx HTTP 状态代码)。
* 当 HTTP `DELETE` 调用成功时。

status 对象编码为 JSON，并作为响应的正文提供。
status 对象包含供 API 的人类和机器使用者使用的字段
获取有关故障原因的更多详细信息。中的信息
status 对象补充但不覆盖 HTTP 状态代码的
意义。当状态对象中的字段具有与一般相同的含义时
定义的 HTTP 标头，该标头与响应一起返回，即标头
应被视为具有更高的优先级。

**Example:**

```console
$ curl -v -k -H "Authorization: Bearer WhCDvq4VPpYhrcfmF6ei7V9qlbqTubUc" https://10.240.122.184:443/api/v1/namespaces/default/pods/grafana

> GET /api/v1/namespaces/default/pods/grafana HTTP/1.1
> User-Agent: curl/7.26.0
> Host: 10.240.122.184
> Accept: */*
> Authorization: Bearer WhCDvq4VPpYhrcfmF6ei7V9qlbqTubUc
>

< HTTP/1.1 404 Not Found
< Content-Type: application/json
< Date: Wed, 20 May 2015 18:10:42 GMT
< Content-Length: 232
<
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {},
  "status": "Failure",
  "message": "pods \"grafana\" not found",
  "reason": "NotFound",
  "details": {
    "name": "grafana",
    "kind": "pods"
  },
  "code": 404
}
```

`status`字段包含以下两个可能值之一：
* `成功`
* `失败`

`message`可能包含人类可读的错误描述

`reason`可能包含机器可读的单字 CamelCase 原因描述
此操作处于`失败`状态。如果此值为空，则没有
可用信息。`reason`阐明了 HTTP 状态代码，但未阐明
覆盖它。

`详细信息`可能包含与原因相关的扩展数据。每个原因都可能
定义自己的扩展详细信息。此字段是可选的，返回的数据是
不保证符合除原因类型定义的架构之外的任何架构。

`reason`和`details`字段的可能值：
* `坏请求`
    * 表示请求本身无效，因为请求没有
      有意义，例如删除只读对象。
    * 这与上面的`状态原因``无效`不同，后者表示
      API 调用可能会成功，但数据无效。
    * 返回 BadRequest 的 API 调用永远不会成功。
    * Http 状态码： `400 StatusBadRequest`

* `未经授权`
    * 表示可以访问并理解服务器请求，但
      拒绝采取任何进一步的行动，而客户没有提供适当的
      授权。如果客户端已提供授权，则此错误指示
      提供的凭据不足或无效。
    * 详细信息(可选)：
        * `kind string`
            * 未经授权的资源的 kind 属性(在某些操作上可能
              与请求的资源不同)。
        * `名称字符串`
            * 未经授权的资源的标识符。
    * HTTP 状态代码：`401 StatusUnauthorized`

* `禁止`
    * 表示可以访问并理解服务器请求，但
      拒绝采取任何进一步的操作，因为它被配置为拒绝访问
      客户端请求的资源的一些原因。
    * 详细信息(可选)：
        * `kind string`
            * 禁止资源的 kind 属性(在某些操作上可能
              与请求的资源不同)。
        * `名称字符串`
            * 禁止资源的标识符。
    * HTTP 状态代码：`403 StatusForbidden`

* `未找到`
    * 表示此操作所需的一个或多个资源不能
      被发现。
    * 详细信息(可选)：
        * `kind string`
            * 缺少资源的 kind 属性(在某些操作上可能
              与请求的资源不同)。
        * `名称字符串`
            * 缺少资源的标识符。
    * HTTP 状态代码：`404 StatusNotFound`

* `已经存在`
    * 表示您正在创建的资源已存在。
    * 详细信息(可选)：
        * `kind string`
            * 冲突资源的 kind 属性。
        * `名称字符串`
            * 冲突资源的标识符。
    * HTTP 状态代码：`409 StatusConflict`

* `冲突`
    * 表示由于
      冲突。客户端可能需要更改请求。每个资源都可以定义
      指示冲突性质的自定义详细信息。
    * HTTP 状态代码：`409 StatusConflict`

* `无效`
    * 表示无法完成请求的创建或更新操作
      由于作为请求的一部分提供的数据无效。
    * 详细信息(可选)：
        * `kind string`
            * 无效资源的 kind 属性
        * `名称字符串`
            * 无效资源的标识符
        * `原因`
            * 一个或多个`StatusCause`条目，指示所提供的数据
              无效的资源。`reason`、`message` 和 `field` 属性将
              被设置。
    * HTTP 状态代码：`422 StatusUnprocessableEntity`

* `超时`
    * 表示无法在给定时间内完成请求。
      如果服务器决定对
      客户端，或者如果服务器过载且无法在此处理请求
      时间。
    * Http 状态码： `429 TooManyRequests`
    * 服务器应设置`Retry-After`HTTP 标头并返回
      `retryAfterSeconds`在对象的详细信息字段中。值`0`是
      违约。

* `服务器超时`
    * 表示可以访问并理解服务器请求，但
      无法在合理的时间内完成操作。这可能是由于暂时的
      服务器负载或与另一台服务器的暂时性通信问题。
        * 详细信息(可选)：
            * `kind string`
                * 正在处理的资源的 kind 属性。
            * `名称字符串`
                * 正在尝试的操作。
    * 服务器应设置`Retry-After`HTTP 标头并返回
      `retryAfterSeconds`在对象的详细信息字段中。值`0`是
      违约。
    * Http 状态码：`504 StatusServerTimeout`

* `方法不被允许`
    * 指示客户端尝试对资源执行的操作
      代码不支持。
    * 例如，尝试删除只能创建的资源。
    * 返回 MethodNotAllowed 的 API 调用永远不会成功。
    * Http 状态码： `405 StatusMethodNotAllowed`

* `内部错误`
    * 表示发生了内部错误，这是意外的，结果
      的电话是未知的。
    * 详细信息(可选)：
        * `原因`
            * 原始错误。
    * Http 状态代码：`500 StatusInternalServerError``code`可能包含此状态的建议 HTTP 返回代码。

## Events

事件是对状态信息的补充，因为它们可以提供一些
除了当前或
以前的状态。为用户或管理员应的情况生成事件
被提醒。

为每个事件类别选择一个唯一、具体、简短的 CamelCase 原因。为
例如，`FreeDiskSpaceInvalid`是一个很好的事件原因，因为它很可能
仅指一种情况，但`已开始`不是一个很好的理由，因为它
即使与其他事件结合使用，也不足以指示开始的内容
领域。

`Error creating foo`或`Error creating foo %s`适用于：
事件消息，后者更可取，因为它更具信息性。

在客户端中累积重复事件，特别是对于频繁的事件，以
减少数据量、系统负载和暴露给用户的噪音。

## Naming conventions

* Go 字段名称必须为 PascalCase。JSON 字段名称必须为 camelCase。其他
  与首字母的大写相比，两者几乎总是匹配。
  两者都没有下划线或破折号。
* 字段和资源名称应该是声明式的，而不是命令式的(SomethingDoer，
  DoneBy、DoneAt)。
* 使用`节点`来指代
  群集上下文中的节点资源。使用`主机`，其中指的是
  单个物理/虚拟系统的属性，例如`hostname`，
  `hostPath`、`hostNetwork` 等。
* `FooController` 是一种已弃用的命名约定。命名以下类型
  被控制的事物(例如，`Job`而不是`JobController`)。
* 指定`某物`发生时间的字段名称应
  被称为`somethingTime`。不要使用`stamp`(例如，`creationTimestamp`)。
* 我们使用`fooSeconds`约定来表示持续时间，如 [单位
  小节](#units)。
    * `fooPeriodSeconds`是周期间隔和其他等待的首选
      句点(例如，在`fooIntervalSeconds`上)。
    * `fooTimeoutSeconds`是不活动/无响应截止时间的首选。
    * `fooDeadlineSeconds`是活动完成截止日期的首选。
* 不要在 API 中使用缩写，除非它们非常常见
  使用，例如`id`、`args`或`stdin`。
* 同样，首字母缩略词只应在非常广为人知的情况下使用。都
  首字母缩略词中的字母应具有相同的大小写，使用适当的大小写
  情况。例如，在字段名称的开头，首字母缩略词应
  全部小写，例如`httpGet`。在用作常量时，所有字母
  应为大写，例如`TCP`或`UDP`。
* 引用`Foo`类其他资源的字段名称应
  称为`fooName`。引用其他同类资源的字段的名称
  ObjectReference(或其子集)的`Foo`应称为`fooRef`。
* 更一般地说，如果可以的话，在字段名称中包括单位和/或键入
  不明确，并且它们不是由值或值类型指定的。
* 表示名为`fooable`的布尔属性的字段的名称应为
  称为`Fooable`，而不是`IsFooable`。
* 
### Namespace Names
* 命名空间的名称必须是
  [DNS_LABEL](https://git.k8s.io/design-proposals-archive/architecture/identifiers.md)。
* `kube-`前缀是为 Kubernetes 系统命名空间保留的，例如`kube-system`和`kube-public`。
*看
  [命名空间文档](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)
  了解更多信息。

## Label, selector, and annotation conventions

标签是用户的域。它们旨在促进组织和
使用对用户有意义的属性管理 API 资源，如
反对对系统有意义。将它们视为用户创建的 mp3 或电子邮件
收件箱标签，而不是程序用于存储的目录结构
其数据。前者使用户能够应用任意本体，而
后者以实施为中心，缺乏灵活性。用户将使用标签来
选择要操作的资源，在 CLI/UI 列中显示标签值等。
用户应始终保留对标签架构的全部权限和灵活性
它们应用于其命名空间中的标签。

但是，默认情况下，我们应该支持常见情况的便利。为
例如，我们现在在 ReplicationController 中所做的是自动设置 RC 的
默认情况下，选择器和标签添加到 Pod 模板中的标签(如果是)
尚未设置。这确保了选择器将与模板匹配，并且
可以使用与它创建的 Pod 相同的标签来管理 RC。注意
一旦我们泛化了选择器，就不一定可能
明确生成与任意选择器匹配的标签。

如果用户想要将其他标签应用于它未选择的 Pod
上，例如促进 Pod 的采用或期望某些
标签值将更改，它们可以将选择器设置为 Pod 的子集
标签。同样，RC 的标签可以初始化为 pod 的子集
模板的标签，或者可以包含其他/不同的标签。

对于在自己的命名空间中管理资源的规范用户来说，情况并非如此
很难始终如一地应用确保唯一性的架构。一个人只需要
确保某些标签键的至少一个值在 common 中不同
到所有其他可比资源。我们可以/应该提供验证工具
来检查一下。但是，制定类似于
[标签](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/) 使独特性变得简单明了。此外
使用范围相对较窄的命名空间(例如，每个环境、每个应用程序)可以
用于减少可能导致重叠的资源集。

如果用户可能正在运行具有不一致架构的杂项示例，
或者工具或组件需要以编程方式生成新对象
被选中，需要有一种直接的方法来生成唯一的标签
集。确保集合唯一性的一种简单方法是确保
单个标签值，例如使用资源名称、UID、资源哈希或
代号。

然而，uid 和哈希的问题包括它们没有语义
对用户来说，意义不易记住，也不容易识别，并且不是
可预言的。缺乏可预测性会阻碍用例的创建
从 Pod 复制控制器，例如人们在探索
系统，引导自托管集群，或删除并重新创建
新的 RC，采用前一个 pod 的 pod，例如重命名它。
世代数更可预测，也更清晰，假设有一个
逻辑顺序。幸运的是，对于部署来说，情况就是这样。对于作业，使用
创建时间戳在内部很常见。用户应该始终能够转动
关闭自动生成，以便允许上述某些场景。
请注意，自动生成的标签也将成为另一个需要的字段
克隆资源时剥离，在命名空间中，在新命名空间中，在
新集群等，在更新资源时需要忽略
通过补丁或读-修改-写序列。

在标签键中包含系统前缀对 UX 相当不利。前缀是
只有在用户无法选择标签键的情况下才需要，以便
以避免与用户定义的标签发生冲突。但是，我坚信
应始终允许用户选择要在其上使用的标签键
资源，因此应该始终可以覆盖默认标签键。

因此，支持自动生成唯一标签的资源应具有
`uniqueLabelKey`字段，以便用户可以根据需要指定密钥
to，但如果未指定，则可以默认设置它，例如 to the resource
类型，如 job、deployment 或 replicationController。该值需要
至少在空间上是独一无二的，在约伯的情况下，也许在时间上是独一无二的。

注释与标签具有非常不同的预期用途。他们是
主要由工具和系统扩展生成和使用，或者使用
由最终用户参与组件的非标准行为。 例如，一个
注释可用于指示资源的实例需要
非 Kubernetes 控制器的额外处理。注释可能带有
任意有效负载，包括 JSON 文档。 与标签一样，注释键可以
以管理域为前缀(例如`example.com/key-name`)。 无前缀
密钥(例如`key-name`)是为最终用户保留的。 第三方组件必须
使用带前缀的键。 `kubernetes.io`和`k8s.io`域下的键前缀
保留供 Kubernetes 项目使用，不得由
第三方。

在早期版本的 Kubernetes 中，一些正在开发的功能代表了新的
API 字段作为注释，通常采用`something.alpha.kubernetes.io/name`或
`something.beta.kubernetes.io/name`(取决于我们对它的信心)。这
模式已弃用。 一些这样的注释可能仍然存在，但没有新的
可以定义注释。 新的 API 字段现在被开发为常规字段。

关于使用标签、注释、污点和其他通用映射键的其他建议
Kubernetes 组件和工具：
- 键名应全部小写，单词之间用破折号分隔，而不是驼峰大小写
    - 例如，更喜欢`foo.kubernetes.io/foo-bar`而不是`foo.kubernetes.io/fooBar`，更喜欢
      `desired-replicas`而不是`DesiredReplicas`
- 无前缀的密钥保留给最终用户。 所有其他标签和批注必须带有前缀。
- `kubernetes.io`和`k8s.io`下的键前缀是为 Kubernetes 保留的
  项目。
    - 此类密钥实际上是 kubernetes API 的一部分，并且可能受制于
      弃用和兼容性策略。
    - `kubernetes.io`是标签和注释的首选形式，不应使用`k8s.io`
      用于新的映射键。
- 键名称(包括前缀)应足够精确，以便用户可以
  合理地了解它来自哪里以及它的用途。
- 键前缀应包含尽可能多的上下文。
    - 例如，更喜欢`subsystem.kubernetes.io/parameter`而不是`kubernetes.io/subsystem-parameter`
- 使用注解来存储控制器负责的 API 扩展
  资源不需要知道，不需要知道的实验田
  旨在用于普遍使用的 API 字段等。请注意，注释不是
  由API转换机制自动处理。

## WebSockets and SPDY

Kubernetes 公开的一些 API 操作涉及二进制文件的传输
客户端和容器之间的流，包括 attach、exec、portforward、
和日志记录。因此，API 通过可升级的 HTTP 公开某些操作
连接([RFC 2817](https://tools.ietf.org/html/rfc2817)中所述)通过
WebSocket 和 SPDY 协议。这些操作作为子资源公开，其中
它们的关联谓词(exec、log、attach 和 portforward)和
通过 GET(在浏览器中支持 JavaScript)和 POST(语义准确)。

目前使用的主要协议有两种：

1. 流媒体频道

在处理多个独立的二进制数据流时，例如
    远程执行 shell 命令(写入 STDIN、从 STDOUT 读取和
    STDERR) 或转发多个端口，这些流可以多路复用到
    单个 TCP 连接。Kubernetes 支持基于 SPDY 的成帧协议，该协议
    利用 SPDY 通道和多路复用的 WebSocket 成帧协议
    通过在每个二进制块前面加上前缀
    指示其通道的字节。WebSocket 协议支持可选的
    处理来自客户端的 base64 编码字节并返回的子协议
    来自服务器和基于字符的通道前缀(`0`，
    `1`， `2`) 便于在浏览器中使用 JavaScript。

2. 流式响应

流数据通道的默认日志输出是 HTTP 分块
    Transfer-Encoding，可以从
    服务器。基于浏览器的 JavaScript 在访问原始数据的能力方面受到限制
    来自分块响应的数据，尤其是当日志量非常大时
    返回，在将来的 API 调用中，可能需要传输大文件。
    流式处理 API 端点支持可选的 WebSocket 升级，该升级提供
    从服务器到客户端的单向通道，并将数据分块为二进制
    WebSocket 框架。公开了一个可选的 WebSocket 子协议，即 base64
    在将流返回到客户端之前对流进行编码。

如果客户端具有本机支持，则客户端应使用 SPDY 协议，或者
WebSockets 作为后备。请注意，WebSockets 容易受到 Head-of-Line 的影响
阻塞，因此客户端必须按顺序读取和处理每条消息。在
将来，将公开弃用 SPDY 的 HTTP/2 实现。


## Validation

API 服务器在收到 API 对象后进行验证。验证错误包括
标记并返回给处于`失败`状态且`原因`设置为
`无效`。为了促进一致的错误消息，我们要求
验证逻辑尽可能遵循以下准则(尽管
将存在特殊情况)。

* 尽可能精确。
* 告诉用户他们能做什么比告诉他们他们能做什么更有用
  不能做。
* 当断言肯定的要求时，使用`必须`。 示例：`必须是
  大于 0`，`必须匹配正则表达式 `[a-z]+``。 像`应该`这样的词意味着
  断言是可选的，必须避免使用。
* 当以否定形式断言格式要求时，请使用`不得`。
  示例：`不得包含`..``。 像`不应该`这样的词意味着
  断言是可选的，必须避免。
* 当断言行为要求是否定时，使用`可能不会`。
  示例：`当 otherField 为空时，可能无法指定`、`只能指定 `name`
  指定`。
* 引用文本字符串值时，在
  单引号。示例：`不得包含`..``。
* 引用其他字段名称时，请在反引号中注明该名称。
  示例：`必须大于 \`request\``。
* 指定不等式时，请使用文字而不是符号。 示例：`必须
  小于 256`，`必须大于或等于 0`。 不要用词
  如`大于`、`大于`、`大于`、`高于`等。
* 指定数值范围时，请尽可能使用非独占范围。

## Automatic Resource Allocation And Deallocation

API 对象通常是包含以下内容的 [union](#Unions) 对象：
1. 一个或多个字段，用于标识特定于 API 对象的`类型`(也称为`鉴别器`)。
2. 一组 N 个字段，在任何给定时间都只能设置其中一个字段 - 实际上是一个并集。

在 API 类型上运行的控制器通常根据
`类型`和/或用户提供的一些附加数据。一个典型的例子
这是`服务`API 对象，其中 IP 和网络端口等资源
将基于`Type`在 API 对象中设置。当用户未指定时
资源，它们将被分配，当用户指定确切的值时，它们将
被保留或拒绝。

当用户选择更改`鉴别器`值(例如，从`X型`更改为`Y型`)时，无需
更改任何其他字段，则系统应清除用于表示`类型 X`的字段
在联合中，同时释放附加到`X 型`的资源。这应该自动
无论这些值和资源是如何分配的(即，由用户保留或
由系统自动分配。一个具体的例子是`服务`API。系统
分配`NodePorts`和`ClusterIPs`等资源，并自动填写以下字段
如果服务类型为`NodePort`或`ClusterIP`(`鉴别器`值)，则表示它们。
当用户更改时，将自动清除这些资源和表示它们的字段
service type 设置为`ExternalName`，其中这些资源和字段值不再适用。

## Representing Allocated Values

许多 API 类型都包含代表用户从
一些较大的空间(例如，某个范围的 IP 地址或存储桶名称)。
这些分配通常由控制器异步驱动到
用户的 API 操作。 有时，用户可以请求特定值和
控制者必须确认或拒绝该请求。 有很多例子
这在 Kubernetes 中，并且有一些模式用于表示它。

所有这些的共同主题是系统不应该信任用户
，并且必须在之前验证或以其他方式确认此类请求
使用它们。

一些例子：

* 服务 `clusterIP`：用户可以在 `spec` 中请求特定的 IP，或者将
  分配了一个(在同一个`spec`字段中)。 如果请求特定 IP，则
  apiserver 将确认 IP 可用，否则将确认
  同步拒绝 API 操作(罕见)。 消费者阅读结果
  来自`spec`。 这是安全的，因为该值要么有效，要么永远不会
  数据处理。
* 服务 `loadBalancerIP`：用户可以在 `spec` 中请求特定的 IP，或者将
  被分配一个，在`状态`中报告。 如果特定 IP 是
  请求时，LB 控制器将确保 IP 可用或
  异步报告故障。 消费者从`status`中读取结果。
  这是安全的，因为大多数用户没有写入`status`的访问权限。
* PersistentVolumeClaims：用户可以在
  `spec` 或将被分配一个(在同一个 `spec` 字段中)。 如果特定的 PV
  请求时，音量控制器将确保卷是
  可用或异步报告故障。 消费者通过以下方式读取结果
  检查 PVC 和 PV。 这比其他的更复杂
  因为`spec`值是在确认之前存储的，这可以
  (假设，由于额外的检查)导致用户访问某人
  其他的PV。
* VolumeSnapshots：用户可以请求对特定源进行快照
  `规格`。 生成的快照的详细信息反映在`状态`中。

反例：

* 服务`externalIPs`：用户必须在`spec`中指定一个或多个特定IP。
  系统无法轻松验证这些 IP(根据其定义，它们是
  外部)。消费者从`spec`中读取结果。 这是不安全的，并且有
  导致不受信任的用户出现问题。

过去，API 约定规定`状态`字段始终来自
观察，这使得其中一些情况比必要的更复杂。
这些约定已经更新，允许`状态`保持这种分配
值。 不过，这不是一个放之四海而皆准的解决方案。

### When to use a `spec` field

新的 API 几乎不应该这样做。 相反，他们应该使用`status`。
如果我们这样做，PersistentVolumes 可能会更简单。

### When to use a `status` field

将此类值存储在`状态`中是最简单、最直接的
模式。 这在以下情况下是合适的：

* 分配的值与对象的其余部分高度耦合(例如 pod
  资源分配)
* 分配的值总是或几乎总是需要的(即大多数
  此类型将有一个值)
* 模式和控制器是先验已知的(即它不是扩展)
* 允许控制器写入`状态`(即
  它们通过其他`状态`字段引起问题的风险很低)。

此类值的使用者可以查看`最终`值的`状态`字段
或指示无法执行分配原因的错误或条件。

#### Sequencing operations

由于几乎所有事情都是异步发生的，几乎其他所有事情，
控制器实现应注意操作的顺序。
例如，控制器是在`状态`字段之前还是之后更新该字段
启动更改取决于需要向观察者做出哪些保证
系统。 在某些情况下，写入`状态`字段表示
确认或接受`spec`值，并且可以在之前编写它
冲动。 但是，如果客户端观察
`status`值，则控制器必须首先启动，并且
之后更新`状态`。 在极少数情况下，控制器需要
确认，然后启动，然后更新到`最终`值。

控制器必须注意如何处理`状态`字段
控制回路中断的情况(例如控制器崩溃和重新启动)，以及
必须以幂等和一致的方式行事。 这在以下情况下尤为重要
使用由消息器馈送的缓存，该缓存可能不会使用最近的写入进行更新。
使用 resourceVersion 前提条件来检测`冲突`是常见的
在这种情况下的模式。 请参阅 [本期](http://issue.k8s.io/105199)
例。

### When to use a different type

以不同的类型存储分配的值更复杂，但也更多
灵活。 这在以下情况下最合适：

* 分配的值是可选的(即此类型的许多实例不会
  完全有价值)
* 模式和控制器不是先验已知的(即它是一个扩展)
* 模式足够复杂(即负担没有意义
  主要类型与它)
* 这种类型的访问控制需要比`所有状态`更精细的粒度
* 分配值的生命周期与
  分配持有人

服务和终结点可以被视为此模式的一种形式，也可以
PersistentVolumes 和 PersistentVolumeClaims。

使用此模式时，必须考虑分配的
对象(谁清理它们以及何时清理它们)以及它们之间的`联系`和
主类型(通常使用相同的名称、object-ref 字段或选择器)。

总会有一些情况可以遵循任何一条路径，而这些情况将
需要人工评估来决定。 例如，服务`clusterIP`高度
与服务的其余部分耦合，大多数实例都使用它。 但它也是
严格可选，并且具有越来越复杂的相关字段架构。
可以对任何一条路径进行论证。