# 04-Hugging Face 生态入门

> 读完本文你将了解：Hugging Face 的三大核心组件（Models / Datasets / Spaces）、Model Card 解读方法、如何用 transformers 库加载和测试模型。

---

## 你大概率遇到过的问题

想用开源模型做实验或下游任务时：

- **模型在哪找**——Google 半天找到的是 paper 不是 weight
- **下载了模型结果不知道怎么用**，连 forward 函数长什么样都不知道
- **看到 Model Card 上写了一堆 metric**，不知道这些数字和你自己的任务有没有关系
- **HF 上几万个模型，不知道怎么选**，同名模型还有 various quantized 版本
- **想用自己的数据测试一个开源模型**，但不知道从哪里开始

这些问题说明一件事：**Hugging Face 已经成了 ML 社区的 GitHub**，不会用 HF 就相当于不会用包管理工具。

---

## 常见的误解

| 你以为的 | 实际的 |
|---------|--------|
| Hugging Face 只是一个模型下载站 | HF 是完整的 ML 开发平台——模型、数据集、训练、部署、社区都在上面 |
| transformers 库只支持 NLP 模型 | 现在也支持 vision、audio、multimodal 模型 |
| Model Card 里的分数可以直接用于对比模型好坏 | Model Card 分数是特定任务/数据集上的，不代表在你的数据上效果一样 |
| 所有 HF 模型都是官方上传的 | 任何人都能上传模型，质量和文档参差不齐 |

---

## Hugging Face 三大组件

### 1. Models：模型的 App Store

地址：https://huggingface.co/models

10 万+ 模型，支持按以下维度筛选：

- **Task**（text-classification、text-generation、image-classification、object-detection、automatic-speech-recognition 等）
- **Framework**（PyTorch、TensorFlow、JAX）
- **Language**（多语言模型 filter）
- **License**（MIT、Apache 2.0、CC-BY-NC、LLaMA 2 License 等）
- **Datasets**（在什么数据上 fine-tuned）

### 2. Datasets：高质量数据集

地址：https://huggingface.co/datasets

结构化数据集的集中地，支持直接从库加载：

```python
from datasets import load_dataset

# 加载 GLUE 的 MRPC 子集
dataset = load_dataset("glue", "mrpc")

# 加载一个图像数据集
dataset = load_dataset("imagenet-1k", split="train")
```

### 3. Spaces：ML Demo 展示平台

地址：https://huggingface.co/spaces

每个 Space 就是一个可交互的 ML demo，用 Gradio 或 Streamlit 构建。

- 你想看一个模型的真实效果？去它的 Space 直接试
- 你想让别人试你的模型？建个 Space 发上去

---

## 如何解读 Model Card

Model Card 是一份描述模型的标准文档。重点关注这几个 section：

### Model description

- **Architecture**：模型结构（如 LLaMA-7B、CLIP ViT-L/14）
- **Developer**：谁训练的（Meta、OpenAI、Stability AI、个人研究员？）
- **Base model**：如果是 fine-tuned，基于哪个模型

### Intended use & limitations

- **Intended use**：这个模型设计的用途是什么（chat、text-generation、image captioning？）
- **Limitations**：模型存在的问题、已知不足——**这比 evaluation 更能帮你判断适不适用**

### Training data

- **Data source**：用了什么数据训练（The Pile、Common Crawl、LAION-5B？）
- **Data size**：数据量级
- **Data preprocessing**：是否做了 dedup、filtering、tokenization

### Evaluation results

评价指标通常包含：

| 指标类型 | 例子 | 说明 |
|---------|------|------|
| 标准 benchmark | GLUE、MMLU、HumanEval | 用于和其他模型横向对比 |
| 任务特定 | BLEU（翻译）、Perplexity（语言模型） | 针对具体任务 |
| Human evaluation | Chatbot Arena Elo | 主观质量评估 |

**关键判断**：Model Card 上的分数是在标准 benchmark 上测的。如果你的任务和数据分布和 benchmark 差异很大，这些分数不一定能反映在你任务上的表现。

---

## 用 transformers 加载和测试模型

### 快速开始

```python
from transformers import pipeline

# 一行代码加载任何模型进行推理
classifier = pipeline(
    "text-classification",
    model="roberta-large-mnli"
)

result = classifier("I love this movie!")
print(result)  # [{'label': 'POSITIVE', 'score': 0.98}]
```

### 加载文本生成模型

```python
from transformers import AutoModelForCausalLM, AutoTokenizer

model_name = "Qwen/Qwen2.5-1.5B-Instruct"

tokenizer = AutoTokenizer.from_pretrained(model_name)
model = AutoModelForCausalLM.from_pretrained(
    model_name,
    torch_dtype="auto",
    device_map="auto"
)

prompt = "Explain the attention mechanism in one paragraph."
inputs = tokenizer(prompt, return_tensors="pt").to(model.device)

outputs = model.generate(**inputs, max_new_tokens=200)
response = tokenizer.decode(outputs[0], skip_special_tokens=True)
print(response)
```

### 加载视觉模型

```python
from transformers import AutoImageProcessor, AutoModelForImageClassification
from PIL import Image

processor = AutoImageProcessor.from_pretrained("google/vit-base-patch16-224")
model = AutoModelForImageClassification.from_pretrained("google/vit-base-patch16-224")

image = Image.open("cat.jpg")
inputs = processor(image, return_tensors="pt")

outputs = model(**inputs)
pred = outputs.logits.argmax(-1).item()
print(model.config.id2label[pred])
```

### 加载模型时常用的参数

```python
model = AutoModelForCausalLM.from_pretrained(
    "meta-llama/Llama-2-7b-chat-hf",
    torch_dtype=torch.float16,    # 半精度节省显存
    device_map="auto",            # 自动分配到可用设备
    load_in_8bit=True,             # 8-bit 量化（显存不够时用）
    trust_remote_code=True,        # 自定义代码的模型需要
)
```

### 模型推理的性能优化

| 场景 | 推荐做法 | 代码 |
|------|---------|------|
| 显存不够 | 量化+半精度 | `load_in_4bit=True` + `torch.float16` |
| 推理太慢 | 用 Flash Attention | `model = model.to_bettertransformer()` |
| 批量推理 | 用 pipeline 的 batch 参数 | `pipeline(model, batch_size=8)` |
| CPU 推理 | 不用改，但会很慢 | 或者用 ONNX 导出加速 |

---

## GPU 不够怎么办

- **用 CPU + 量化**：`load_in_8bit=True` 或 `load_in_4bit=True`
- **Google Colab**：免费 T4，pro 有 A100
- **HF Inference API**：不用本地跑，HTTP 请求调模型
- **Ollama + local models**：适合本地 7B 以下模型

---

## 要点总结

1. **Hugging Face 三大组件**：Models（模型托管）、Datasets（数据集）、Spaces（Demo 展示），覆盖 ML 全流程
2. **Model Card 重点看四块**：基础信息（用途）、训练数据（来源）、评价指标（benchmark）、局限性（适用边界）
3. **transformers 库**用 `pipeline` 一行加载模型，`AutoModel.from_pretrained` 处理所有细节
4. **模型加载参数**：`torch_dtype`、`device_map`、`load_in_8bit/4bit` 是控制资源消耗的三个关键参数
5. **不用在本地跑大模型**：Colab、HF Inference API、Ollama 都是可行的替代方案
