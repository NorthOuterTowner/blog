# 基于企业数据的 Agent 开发经历

## Chapter 1

本Agent作为满足企业内部日常工作的Agent，需要具备覆盖本部门日常工作的大量精细化功能，因此我们决定使用细粒度的skill完成工作，在我看来，我们skill的粒度之细已经达到了tool的水平，当然，因为当前skill的数量并不非常多，因此这种粒度的skill其实也可以的，并且这也更利于编排。

### 架构设计

总的来说，我们采用单个中心进行Agent调度的Agent Group架构，中心agent称为orchestra，同时具有若干agent将自己的功能进行描述，在收到用户请求时利用orchestra的能力进行agent的编排和调度，告知agent其需要执行的skill。

### Orchestra 执行流程

首先，Orchestra接收前端的数据，之后抽取用户和agent的交流历史，注入单次的prompt。*此处我也是第一次知道agent是通过这种方式实现上下文的读取的，我原本以为LLM具备记忆的能力，但是经过调研后发现其本身是不具备记忆的，这也是为什么再好的模型都有上下文长度的限制的原因*。

orchestra接收信息后，读取各个agent负责的功能，将任务拆解后依次分给多个agent，随后，agent根据每个skill暴露的skill.yaml，进行参数的猜测，即用用户的输入提取skill script执行所需要的输入参数。

随后调用对应的script脚本进行执行，并在执行完毕后将内容返回给agent，agent再返回给orchestra，最终，orchestra将结果返回用户。

### Agent 设计

初期，Agent主要存在以下几类，分别是Knowledge Agent，Document Agent， Web Agent等等，其中Knowledge Agent负责从LLM中获取知识，Document负责各种文档的撰写，Web Agent负责从网络API获取数据。

简单来说，此处Knowledge的逻辑是用自动化工具从公司已提供的LLM工具中获取输出，Document的逻辑是根据模板利用大模型/模板预设方法完成各种格式和要求的文档，Web直接从API中获取数据。