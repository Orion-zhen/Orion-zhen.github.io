---
date: "2024-10-19T21:35:02+08:00"
draft: false
title: "使用 LLaMA-Factory 微调大语言模型"
slug: "train-llm-with-llama-factory"
description: 新手友好的训练方式
image: /post/tutorial/train-llm-with-llama-factory-logo.png
categories:
  - 教程
tags:
  - 大模型
  - 微调
  - LLaMA-Factory
license: GNU General Public License v3.0
---

## 写在前面

[LLaMA-Factory](https://github.com/hiyouga/LLaMA-Factory) 是一款新手友好, 支持范围广泛的大模型微调工具. 通过它, 你可以轻松地使用多种方式微调多种架构的大模型

需要指出的是, 微调大模型比部署大模型更加复杂, 对你的能力和硬件资源也提出了更高的要求. 因此, 请确保你:

- 具备大模型的基础知识
- 了解不同是数据集格式, 以及如何处理数据
- 了解微调大模型的基本原理
- 掌握基础的编程语言技能
- 至少拥有一块显存超过 24G 的显卡, 或者愿意租用云服务器
- 准备好充足的时间

## 准备工作

### 安装 LLaMA-Factory

**首先将 LLaMA-Factory 克隆到本地**:

```bash
git clone https://github.com/hiyouga/LLaMA-Factory.git
cd LLaMA-Factory
```

**然后安装依赖**:

```bash
pip install -r requirements.txt -U
```

**最后安装 LLaMA-Factory**:

```bash
pip install -e . -U
```

### 选择训练方式

**训练策略**:

- 继续预训练: continue pretrain, 可以给大模型注入新知识, 但需要**高质量**数据集, 需要**大量**的时间和资源
- 指令监督微调: supervised fine-tune, 可以激发出大模型已有的知识, 但无法注入新知识
- 偏好学习: 可以改变大模型的说话方式

**训练方式**:

- 全量微调: 训练整个模型, 包括所有参数. 可以达到最理想的*效果*. 注意, 这里的*效果*仅指微调的影响的显著程度, 而不是模型的最终性能表现. 不恰当的全量微调可能导致模型性能大幅下降
- LoRA: 使用低秩适配器来微调模型, 相较于全量微调需要的计算资源和时间显著减少, 但是*效果*不如全量微调
- QLoRA: 将原模型量化后再使用 LoRA 微调. 进一步减少计算资源, 稍微增加训练时间, *效果*和 LoRA 相当

如果你不知道如何选择, sft+LoRA 可能是一个不错的开始

### 准备数据集

根据不同的**训练策略**, 你需要准备不同的数据集

一般地, 数据集可以分为两种格式: Alpaca 和 ShareGPT 格式. 其中, Alpaca 格式的数据集主要是单轮对话, 而 ShareGPT 格式的数据集可以是单轮, 也可以是多轮对话

在 LLaMA-Factory 中, 准备数据集需要在 `data/dataset_info.json` 文件中声明. 一般地, 声明一个数据集需要包含以下信息:

```json
"数据集名称": {
  "hf_hub_url": "Hugging Face 的数据集仓库地址 (若指定, 则忽略 script_url 和 file_name)",
  "ms_hub_url": "ModelScope 的数据集仓库地址 (若指定, 则忽略 script_url 和 file_name)",
  "script_url": "包含数据加载脚本的本地文件夹名称 (若指定, 则忽略 file_name)",
  "file_name": "该目录下数据集文件夹或文件的名称 (若上述参数未指定, 则此项必需)",
  "formatting": "数据集格式 (可选, 默认: alpaca, 可以为 alpaca 或 sharegpt)",
  "ranking": "是否为偏好数据集 (可选, 默认: False)",
  "subset": "数据集子集的名称 (可选, 默认: None)",
  "split": "所使用的数据集切分 (可选, 默认: train)",
  "folder": "Hugging Face 仓库的文件夹名称 (可选, 默认: None)",
  "num_samples": "该数据集所使用的样本数量. (可选, 默认: None)",
  "columns(可选)": {
    "prompt": "数据集代表提示词的表头名称 (默认: instruction)",
    "query": "数据集代表请求的表头名称 (默认: input)",
    "response": "数据集代表回答的表头名称 (默认: output)",
    "history": "数据集代表历史对话的表头名称 (默认: None)",
    "messages": "数据集代表消息列表的表头名称 (默认: conversations)",
    "system": "数据集代表系统提示的表头名称 (默认: None)",
    "tools": "数据集代表工具描述的表头名称 (默认: None)",
    "images": "数据集代表图像输入的表头名称 (默认: None)",
    "videos": "数据集代表视频输入的表头名称 (默认: None)",
    "chosen": "数据集代表更优回答的表头名称 (默认: None)",
    "rejected": "数据集代表更差回答的表头名称 (默认: None)",
    "kto_tag": "数据集代表 KTO 标签的表头名称 (默认: None)"
  },
  "tags(可选, 用于 sharegpt 格式)": {
    "role_tag": "消息中代表发送者身份的键名 (默认: from)",
    "content_tag": "消息中代表文本内容的键名 (默认: value)",
    "user_tag": "消息中代表用户的 role_tag (默认: human)",
    "assistant_tag": "消息中代表助手的 role_tag (默认: gpt)",
    "observation_tag": "消息中代表工具返回结果的 role_tag (默认: observation)",
    "function_tag": "消息中代表工具调用的 role_tag (默认: function_call)",
    "system_tag": "消息中代表系统提示的 role_tag (默认: system, 会覆盖 system column)"
  }
}
```

下面根据不同的训练策略讲解数据集的准备:

#### 继续预训练

预训练一般采用 Alpaca 格式

在预训练时，只有 `text` 列中的内容会用于模型学习

```json
[
  {"text": "document"},
  {"text": "document"}
]
```

对于上述格式的数据，`dataset_info.json` 中的*数据集描述*应为：

```json
"数据集名称": {
  "file_name": "data.json",
  "columns": {
    "prompt": "text"
  }
}
```

#### 指令监督微调

**Alpaca 格式**:

在指令监督微调时，`instruction` 列对应的内容会与 `input` 列对应的内容拼接后作为人类指令，即人类指令为 `instruction\ninput`. 而 `output` 列对应的内容为模型回答

如果指定，`system` 列对应的内容将被作为系统提示词

`history` 列是由多个字符串二元组构成的列表，分别代表历史消息中每轮对话的指令和回答. 注意在指令监督微调时，历史消息中的回答内容**也会被用于模型学习**

> 事实上, Alpaca 格式做多轮对话非常麻烦, 建议使用 ShareGPT 格式

```json
[
  {
    "instruction": "人类指令(必填)",
    "input": "人类输入(选填)",
    "output": "模型回答(必填)",
    "system": "系统提示词(选填)",
    "history": [
      ["第一轮指令(选填)", "第一轮回答(选填)"],
      ["第二轮指令(选填)", "第二轮回答(选填)"]
    ]
  }
]
```

对于上述格式的数据，`dataset_info.json` 中的*数据集描述*应为：

```json
"数据集名称": {
  "file_name": "data.json",
  "columns": {
    "prompt": "instruction",
    "query": "input",
    "response": "output",
    "system": "system",
    "history": "history"
  }
}
```

**ShareGPT 格式**:

相比 Alpaca 格式的数据集，ShareGPT 格式支持**更多的角色种类**，例如 human, gpt, observation, function 等等. 它们构成一个对象列表呈现在 `conversations` 列中

注意其中 human 和 observation 必须出现在奇数位置，gpt 和 function 必须出现在偶数位置

```json
[
  {
    "conversations": [
      {
        "from": "human",
        "value": "人类指令"
      },
      {
        "from": "function_call",
        "value": "工具参数"
      },
      {
        "from": "observation",
        "value": "工具结果"
      },
      {
        "from": "gpt",
        "value": "模型回答"
      }
    ],
    "system": "系统提示词(选填)",
    "tools": "工具描述(选填)"
  }
]
```

对于上述格式的数据，`dataset_info.json` 中的*数据集描述*应为：

```json
"数据集名称": {
  "file_name": "data.json",
  "formatting": "sharegpt",
  "columns": {
    "messages": "conversations",
    "system": "system",
    "tools": "tools"
  }
}
```

#### 偏好学习

**Alpaca 格式**:

偏好数据集用于奖励模型训练, DPO 训练, ORPO 训练和 SimPO 训练

它需要在 `chosen` 列中提供更优的回答，并在 `rejected` 列中提供更差的回答

```json
[
  {
    "instruction": "人类指令(必填)",
    "input": "人类输入(选填)",
    "chosen": "优质回答(必填)",
    "rejected": "劣质回答(必填)"
  }
]
```

对于上述格式的数据，`dataset_info.json` 中的*数据集描述*应为：

```json
"数据集名称": {
  "file_name": "data.json",
  "ranking": true,
  "columns": {
    "prompt": "instruction",
    "query": "input",
    "chosen": "chosen",
    "rejected": "rejected"
  }
}
```

**ShareGPT 格式**:

ShareGPT 格式的偏好数据集同样需要在 `chosen` 列中提供更优的消息，并在 `rejected` 列中提供更差的消息

```json
[
  {
    "conversations": [
      {
        "from": "human",
        "value": "人类指令"
      },
      {
        "from": "gpt",
        "value": "模型回答"
      },
      {
        "from": "human",
        "value": "人类指令"
      }
    ],
    "chosen": {
      "from": "gpt",
      "value": "优质回答"
    },
    "rejected": {
      "from": "gpt",
      "value": "劣质回答"
    }
  }
]
```

对于上述格式的数据，`dataset_info.json` 中的*数据集描述*应为：

```json
"数据集名称": {
  "file_name": "data.json",
  "formatting": "sharegpt",
  "ranking": true,
  "columns": {
    "messages": "conversations",
    "chosen": "chosen",
    "rejected": "rejected"
  }
}
```

### 准备模型

请下载**原始模型**, 不要下载量化模型

## 开始微调

现在你已经准备好了数据集和模型. 让我们开始微调吧!

复制一份 `examples/train_lora/llama3_lora_sft.yaml` 然后重命名成你自己的配置文件, 例如我将其重命名为 `examples/train_lora/qwen2_lora_sft.yaml`. 然后开始更改相关配置:

- model_name_or_path: 你之前下载的**原始模型**的路径. 请填写绝对路径
- lora_rank: LoRA 的秩, 为 2 的幂. 一般取 8. 秩越大, 需要的显存越多. 根据[这篇论文](https://arxiv.org/abs/2106.09685), LoRA 的性能在 32 秩时达到最佳, 64 秩时开始下滑. 但是根据[这篇论文](https://arxiv.org/abs/2405.09673), LoRA rank 越大, 越趋近于全量微调, LoRA rank 是用来调节学习程度和遗忘程度的变量
- dataset: 你需要使用的数据集. 如果有多个数据集, 请用逗号分隔
- template: 你需要使用的 prompt 模板. 请根据你选择的模型填写其相应的模板. 例如 Qwen2.5 的模板为 qwen
- cutoff_len: 最大输入长度, 你可以近似理解为上下文. 越长的上下文需要的显存越多, 而过短的上下文可能导致数据集内容被异常截断, 导致模型训练效果不佳. 你需要自行选择
- max_samples: 最大训练样本数量. 一般来说, 越多的训练样本, 训练效果越好, 但也会消耗更多的显存和时间. 配置文件模板给出的值是 1000, 将这一行注释掉即可使用全部样本进行微调
- output_dir: 微调后的模型保存路径
- logging_steps: 训练记录输出间隔, 每隔 `logging_steps` 个 step 输出一次训练记录, 包含 `loss`, `grad_norm`, `learning_rate` 等信息
- save_steps: 训练模型保存间隔, 每隔 `save_steps` 个 step 保存一次模型, 保存在 `output_dir` 目录下
- overwrite_output_dir: 是否覆盖 `output_dir` 目录
- restore_callback_states_from_checkpoint: 是否从上一个保存的检查点恢复微调状态, 适用于训练异常中止后恢复训练. 和 `overwrite_output_dir` 冲突
- save_total_limit: 最多保存的模型数量, 超出数量的模型将被删除
- per_device_train_batch_size: 训练时每个 GPU 的 batch size
- gradient_accumulation_steps: 累积梯度的步数, 即每 `gradient_accumulation_steps` 个 step 进行一次梯度累积, 并更新模型参数. 关于梯度累计和 batch size 的讨论, 可以参考[这个讨论贴](https://discuss.huggingface.co/t/batch-size-vs-gradient-accumulation/5260)
- learning_rate: 学习率, 一般在 `1e-5` 到 `1e-4` 之间, 根据你的训练状况调整
- num_train_epochs: 在数据集上训练的轮数. 当数据集较大时, 设置为 1 即可. 过大的训练轮数可能会导致模型过拟合
- bf16: 是否使用 bf16 精度训练. 当你的 GPU 支持 bf16 时, 建议开启. 例如 NVIDIA 3000 系及以上的显卡支持 bf16
- fp16: 是否使用 fp16 精度训练. 当你的 GPU 不支持 bf16 时, 建议开启

在这里我给出我的一份配置文件以供参考:

```yaml
### model
model_name_or_path: /path/to/qwen2.5-14b

### method
stage: sft
do_train: true
finetuning_type: lora
lora_target: all
lora_rank: 32

### dataset
dataset: nopm_writing, dirty_writing
template: qwen
cutoff_len: 4096
# max_samples: 1000
overwrite_cache: true
preprocessing_num_workers: 16

### output
output_dir: saves/qwen2.5-14b/lora/sft-write
logging_steps: 1
save_steps: 200
plot_loss: true
# overwrite_output_dir: true
restore_callback_states_from_checkpoint: true
save_total_limit: 5

### train
per_device_train_batch_size: 1
gradient_accumulation_steps: 32
max_grad_norm: 1.0
learning_rate: 2.0e-5
num_train_epochs: 1
lr_scheduler_type: cosine
warmup_ratio: 0.1
fp16: true
upcast_layernorm: false
ddp_find_unused_parameters: false
ddp_timeout: 180000000

### eval
val_size: 0.1
per_device_eval_batch_size: 1
eval_strategy: steps
eval_steps: 200
```

好了, 现在你已经准备好了配置文件, 接下来可以启动微调了. 使用如下命令启动微调:

```shell
llama-factory train examples/train_lora/qwen2_lora_sft.yaml
```

命令格式是:

```shell
llama-factory train 配置文件路径
```

如果一切顺利, 你应该在训练过程结束后, 在 `output_dir` 目录下看到微调后的模型文件

## 合并 Lora

复制一份 `examples/merge_lora/llama3_lora_sft.yaml` 然后重命名成你自己的配置文件, 例如我将其重命名为 `examples/merge_lora/qwen2_lora_sft.yaml`. 然后开始更改相关配置:

- model_name_or_path: 你之前微调后的模型的路径. 请填写绝对路径
- adapter_name_or_path: 你之前训练的 LoRA 路径, 即 `output_dir`
- template: 你需要使用的 prompt 模板. 请根据你选择的模型填写其相应的模板. 例如 Qwen2.5 的模板为 qwen
- export_dir: 合并后的模型保存路径
- export_size: 模型分块文件的大小, 单位为 GB, 整数. 一般不要超过 5

好了, 现在你已经准备好了配置文件, 接下来可以启动合并. 使用如下命令启动合并:

```shell
llama-factory export examples/merge_lora/qwen2_lora_sft.yaml
```

命令格式是:

```shell
llama-factory export 配置文件路径
```

如果一切顺利, 你应该在合并过程结束后, 在 `export_dir` 目录下看到合并后的模型文件

## 进阶技巧

人类的 GPU 显存是有极限的, 但是人类对于在本地微调更大模型的欲望是无穷的. 为了在本地微调更大模型, 智慧的人类开发出了多种方法

### QLoRA

QLoRA 是 LoRA 的一个优化, 允许在**量化后**的模型上训练 LoRA. 相比于直接训练 LoRA, QLoRA 的显存占用显著降低, 而训练速度稍微变慢

要使用 QLoRA, 需要安装 bitsandbytes 库:

```shell
pip install bitsandbytes
```

如果是 Windows, 则需要从已经构建好的 whl 文件安装 bitsandbytes:

```shell
pip install https://github.com/jllllll/bitsandbytes-windows-webui/releases/download/wheels/bitsandbytes-0.41.2.post2-py3-none-win_amd64.whl
```

然后, 从 `examples/train_qlora/llama3_lora_sft_oftq.yaml` 复制一份配置文件, 依照前文 LoRA 的相关配置进行更改

QLoRA 特定的配置项仅有两项:

- quantization_bit
- quantization_method

一般保持默认即可

### Unsloth

一个神奇的仓库, 可以显著地加快训练速度并降低显存需求, 但是仅能在单卡上运行

要使用 Unsloth, 需要安装 Unsloth 库:

```shell
pip install "unsloth @ git+https://github.com/unslothai/unsloth.git"
```

如果你的 NVIDIA 显卡是 3000 系及以上, 则可以指定显卡架构安装:

```shell
pip install "unsloth[cu121-ampere-torch240] @ git+https://github.com/unslothai/unsloth.git"
```

要使用 Unsloth, 只需要在你的配置文件中加入这一行:

```yaml
use_unsloth: true
```

### Liger Kernel

Liger Kernel 也可以显著地减少显存占用, 且没有单卡要求. 要使用 Liger Kernel, 只需要在你的配置文件中加入这一行:

```yaml
enable_liger_kernel: true
```

### FSDP+QLoRA

一种分布式训练方法, 要求你至少有两块显卡. 这种方法牺牲了训练时间, 但极大地减少了显存占用, [你甚至可以在两块 24G 的显卡上微调 70B 参数的大模型](https://www.answer.ai/posts/2024-03-06-fsdp-qlora.html)

但是天下没有免费的午餐, FSDP+QLoRA 需要较高的内存大小. 我在使用这个方法微调 32B 参数的 Qwen2.5-32B-Instruct 时, 观察到的最高内存占用可达 88GB.

假设你有两张显卡, 要使用 FSDP+QLoRA, 请复制一份 `examples/extras/fsdp_qlora/llama3_lora_sft.yaml` 配置文件, 按照 LoRA 的配置进行更改, 注意**不要更改**量化位数

然后, 找到 `examples/extras/fsdp_qlora/train.sh`, 将其中的 `examples/extras/fsdp_qlora/llama3_lora_sft.yaml` 改成你的配置文件路径

最后, 使用如下命令启动训练:

```shell
bash examples/extras/fsdp_qlora/train.sh
```
