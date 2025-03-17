# 1、整体代码结构



## 1.1 第一层代码##


![image-20250315180500189](D:\2025\论文\MD文章\img\dify定制开发\image-20250315180500189.png)

说明： 

api: 相关的api，后端代码

dev：测试相关用例

web：前端，前段代码

sdk：对外sdk

docker：相关运行的容器

images：对应的图片



## 1.2 第二层代码结构



### 1.2.1 api目录##

![image-20250315181422951](D:\2025\论文\MD文章\img\dify定制开发\image-20250315181422951.png)

说明：

controllers：flask的mvc

core：核心代码

libs：支撑库

extensions：扩展

service：后端服务



#### core代码

![image-20250315181716202](D:\2025\论文\MD文章\img\dify定制开发\image-20250315181716202.png)

agent：代理相关

app：入口对话相关

callback_handler：回调处理

entities：实体

file：？

llm_generator： llm入口

model_runtime：模型调用监控

plugin：插件相关

prompt：提示词相关

rag：rag流程相关

tools工具相关

workflow：工作流相关

processor：文档处理



#### controllers代码

![image-20250315182345239](D:\2025\论文\MD文章\img\dify定制开发\image-20250315182345239.png)

service_api：对外api，分为app的和datset，其中daset主要处理文件上传

inner_api： 内部api

console：管理控制台



####  service代码

![image-20250315182719711](D:\2025\论文\MD文章\img\dify定制开发\image-20250315182719711.png)





###  1.2.2 web目录



当前未分析



## 1.3主要的路由

- service_api_bp: 服务 API 路由 (/api/v1)
- web_bp: Web API 路由 (/api)
- console_app_bp: 控制台 API 路由 (/console/api)
- files_bp: 文件处理 API 路由
- inner_api_bp: 内部 API 路由

## 1.4总结



# 2、整体流程

## 2.1 ChatAppGenerator整体流程

### 2.1.1 核心入口类：

ChatAppGenerator

AgentChatAppGenerator

AdvancedChatAppGenerator

WorkflowAppGenerator



### 2.1.2 api入口

***上层调用***

ChatAppGenerator



第一层：

PluginInvokeLLMApi

api.add_resource(PluginInvokeLLMApi, "/invoke/llm")

第二层：

PluginAppBackwardsInvocation::invoke_chat_app



### 2.1.3 ChatAppGenerator分析

***主体流程：***

\# init application generate entity

\# init generate records

\# init queue manager

\# new thread

\# return response or stream generator

***最终运行的类***：ChatAppRunner，对应的方法_generate_worker



### 2.1.4 ChatAppRunner分析

**初始化：**ChatAppRunner类中

   `\# `

`new thread`

​    `worker_thread = threading.Thread(`

​      `target=self._generate_worker,`

​      `kwargs={`

​        `"flask_app": current_app._get_current_object(),  # type: ignore`

​        `"application_generate_entity": application_generate_entity,`

​        `"queue_manager": queue_manager,`

​        `"conversation_id": conversation.id,`

​        `"message_id": message.id,`

​      `},`

​    `)`

ChatAppGenerateEntity绑定了对应的模型，转换，查询，输入输出和转换

对应的实现类：

ChatAppGenerateEntity



![](D:\2025\论文\MD文章\img\image-20250315121752826.png)



`runner.run(`

​          `application_generate_entity=application_generate_entity,`

​          `queue_manager=queue_manager,`

​          `conversation=conversation,`

​          `message=message,`

​        `)`



**主体流程**

1. 获得token
2. 组织提示词模板
3. 调用model处理输入的词，主要处理敏感词部分
4. 处理annotation的回复
5. 如果有外部工具，使用外部工具填充，将对应的信息放入input中
6. 从知识库中检索：**DatasetRetrieval**
7. 根据RAG返回的信息和提示词模板结合，生成对应输入的指令（包括历史信息）
8. 使用本机的model进行处理，如果有结果，直接返回
9. 使用外部模型处理
10. ​	处理针对最长token的限制
11. ​	获得对应处理的模型
12. ​	调用模型
13. 返回对应的结果



## 2.2 相关流程

###  2.2.1 RAG核心检索流程

主要的类：DatasetRetrieval

主要方法：retrieve

核心的类：

single_retrieve

multiple_retrieve



#### **single_retrieve主要流程：**

获得对应的配置

如果react_router，调用ReactMultiDatasetRouter返回信息，处理提示词

如果router，调用FunctionCallMultiDatasetRouter返回信息，处理提示词

如果返回了对应信息

​	如果为external知识库，ExternalDatasetService.fetch_external_knowledge_retrieval

​	否则，设置相关的参数和rank信息，**调用RetrievalService.retrieve方法**，保存相关日志到db

​	

#### **multiple_retrieve主要流程**

**主要流程**

针对每个dataset启动线程调用_retriever方法，调用相关的进行检索，最终调用的是**RetrievalServicede的retrieve方法**

等待线程完成

如果有**rank**的设置，调用DataPostProcessor进行处理

否则，进行对应的评分，返回对应的文档

保存相关的日志



#### **RetrievalServicede的retrieve方法**

**调用RetrievalService.retrieve的方法**，主要有三种方法，选择其中一个进行检索：

​		keyword_search

​		semantic_search

​		hybrid_search，混合索引需要DataPostProcessor来处理相关的数据



搜索分析：

1、keyword_search

keyword

self._keyword_processor.search(query, **kwargs)

2、semantic_search

vector

self._vector_processor.search_by_vector(query_vector, **kwargs)

3、full_text_index_search

self._vector_processor.search_by_full_text(query, **kwargs)



#### **rank处理**

**相关代码：**

`DataPostProcessor(tenant_id, reranking_mode, reranking_model, weights, False)`

`query=query, documents=all_documents, score_threshold=score_threshold, top_n=top_k`

​    if self.rerank_runner:

​      documents = self.rerank_runner.run(query, documents, score_threshold, top_n, user)

最终调用的是：

RerankRunnerFactory.create_rerank_runner实现的rank

对应的类：BaseRerankRunner，对应的子类为：RerankModelRunner和WeightRerankRunner



**rank的流程**：以WeightRerankRunner进行分析，相对比较简单，调用实例化的rank模型直接返回就完了





#### **processor文档**处理

对应的包：processor，基类：BaseIndexProcessor

三个子类是：qa paret_child,paragraph

**触发：**

对应触发的类的是：IndexingRunner，对应的主方法是run，通过消息触发，比如：@document_index_created.connect的消息，接受docum_index_evevt

发出docum_index_evevt消息的时机是：

**核心流程** run方法

1. 从dataset获得dataset
2. 获得process rule
3. 通过IndexProcessorFactory创建对应的processer
4. _extract方法拿到对应的文件
5. _transform做文件的转换，以QAIndexProcessor来说

- ​	_get_splitter获得分割器

- ​	针对每个文件进行如下操作

- ​		分割文件

- ​		_format_qa_document进行格式化文件处理，需要单独的启动线程，包括：通过模型获得响应，最后将相应直接进行format

 6. _load_segments保存对应的文件到数据库

 7. _load方法将对应的数据进行处理？？需要继续分析

    

## 2.4 AdvancedChatAppGenerator流程



# 3、优化以及建议

## 3.1知识库检索增强和优化

多知识库检索从代码看已经支持



## 3.2多模型协作调度

需要在获得model这一层再增加一层，但是怎么路由和保持上下文很关键，如果只是单轮的话没有问题？单轮这个好像没有什么竞争力



## 3.3 agent工具集成框架



## 3.4文档处理管道化



## 3.5用户行为分析

需要根据数据结构中记录的数据进行处理



## 2.6模型评估和基准测试系统

























