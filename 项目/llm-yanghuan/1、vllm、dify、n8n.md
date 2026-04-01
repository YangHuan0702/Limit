## vLLM安装与使用
vLLM简单来说就是我们跑模型的应用，有点类似于java application与tomcat的感觉，同类的产品还有ollama与SGLang；它们三者的本质都是将**模型跑起来且对外提供API**，但是这三者的定位与场景差别很大：

|      | Ollama                             | vLLM               | SGLang                   |
| ---- | ---------------------------------- | ------------------ | ------------------------ |
| 定位   | 本地开发/个人使用                          | 生产级高并发推理服务         | 结构化输出/高效推理               |
| 目标硬件 | Mac/普通PC，支持Application Silicon、CPU | NVIDIA GPU，需要CUDA  | NVIDIA GPU，需要CUDA        |
| 并发能力 | 一次基本只能处理一个请求                       | PagedAttention、高并发 | RadixAttention、更高效       |
| 安装难度 | 一条命令搞定                             | 需要CUDA环境           | 需要CUDA环境                 |
| 适合场景 | 本地测试、调试prompt、个人工具                 | API服务、多用户产品、高QPS   | Agent多轮调用、JSON结构化输出、批量推理 |
## 核心概念：vLLM 为什么快？

传统推理框架最大的问题是 **KV Cache 内存浪费**。每个请求的 KV Cache 预先分配一块连续内存，但实际用多少是动态的，导致大量碎片和浪费。vLLM 的核心创新是 **PagedAttention**：

传统方式：
![[Pasted image 20260326163907.png]]

PagedAttention：
![[Pasted image 20260326163953.png]]


## 安装vLLM

1、确认环境
```
# 确认CUDA版本（vLLM要求CUDA 11.8+）
nvidia-smi

# 确认python版本(需要3.9+)
python3 --version
```

2、环境搭建
这里建议创建虚拟环境进行使用
```
# 创建并使用虚拟环境
conda craete -n vllm python=3.11 -y
conda activate llm

# 安装vLLM
pip install vllm

# 推荐从HuggingFace或者ModelScope，国内用ModelScope会更快点
pip install modelscope

modelscope download --model 'Qwen/Qwen2.5-7B-Instruct' --local_dir '/home/models'
```

3、运行模型
```
# 最简单的启动方式，对外暴露 OpenAI 兼容的 API
python3 -m vllm.entrypoints.openai.api_server \
    --model ./models/Qwen2.5-7B-Instruct \
    --served-model-name qwen2.5-7b \
    --host 0.0.0.0 \
    --port 8000 \
    --max-model-len 4096
    
    
# 因为我的服务器的原因，所以我跑的命令是：
python -m vllm.entrypoints.openai.api_server  --model "qwen/Qwen2.5-3B-Instruct" --port 8100  --max-model-len 4096  --gpu-memory-utilization 0.7 --dtype float16  --enforce-eage
```

![[Pasted image 20260326202532.png]]

出现`Application startup complete.`就证明是成功了嘛
需要说明的是，这玩意就是个工具，对于工具的使用我觉得没必要写太多的说明啊啥的，有啥不懂的不是很清晰的可以直接看它的官方文档：https://docs.vllm.ai/en/latest/cli/serve/







## dify的安装与使用
Dify是目前主流的LLM应用平台，dify平台中包含了RAG、上下文、工具调用等我们在`llm-chat-service`中做的功能， 当然了它做的更好，在这些基础上它又衍生出了聊天助手、工作流、Agent、文本生成等功能，架构图如下：
![[Pasted image 20260326174258.png]]

Dify最大的价值在于：把做LLM应用最繁琐的部分都给封装好了，就好比是一个框架一样；更重要的是它全部开源了。

#### 安装dify
```
# 下载代码
git clone https://github.com/langgenius/dify.git
cd dify/docker

docker compose up -d
```
这玩意我为了简单快速，直接用docker compose跑起来的，它依赖较多的中间件，在实际企业运用的话肯定是让它接入到已经搭建好的中间件服务中。
然后用浏览器打开使用就行了，它的端口是80。



#### 安装n8n
n8n — 自动化工作流引擎
核心思维模型：一切都是**节点（Node）**，节点之间传递**数据（JSON）**。  
触发节点 → 处理节点 → 输出节点，串成一条流。

三类核心节点：
- Trigger 触发器：定时 / Webhook / 邮件到达
- Action 动作：HTTP 请求、数据库读写、发消息
- Logic 逻辑：IF 判断、循环、数据转换

AI 相关节点：
- AI Agent 让 LLM 自主决策
- Chat Model 接 OpenAI / Ollama / vLLM
- Embeddings 文本向量化
- Vector Store 存取向量数据

简单又高效的部署：
```
docker run -d -p 5678:5678 n8nio/n8n
```

