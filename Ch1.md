# Qwen-blog学习笔记

## Qwen2模型架构

![框架图](./img/framework.JPG)

图1 Qwen2模型架构图

Qwen2与Decoder-only的Llama2架构类似，都是由Tokenizer分词器、Embedding嵌入层以及由transformer decoder layer变体而来的decoder layer，不同的是Qwen2的decoder layer采用的是pre-norm，transformer的decoder layer采用的是post-norm，Qwen2的MLP层中加入了门控信息，Qwen2不只是在嵌入层添加位置信息，而是在每一个注意力层都使用了Qwen2RotaryEmbedding旋转嵌入，Qwen2使用RMSNorm正则化，transformer使用层正则化。

其中：

- `tokenizer`将文本转为词表里面的数值。
- 数值经过`embedding`得到一一对应的向量。
- post-norm 应对表征坍塌（Representation Collapse）的能力更强，但处理梯度消失略弱。而pre-norm可以更好的应对梯度消失，但处理表征坍塌的能力略弱。
- 旋转位置编码（Rotary Positional Embeddings, RoPE）替代了原有的绝对位置编码，从而增强位置编码的表达能力，增强了模型对序列顺序的理解。
- RMSNorm 是 LayerNorm 的简化版本，通过**移除中心化步骤**降低计算复杂度，同时在多数任务中保持性能接近。其核心优势是高效性，适合对计算资源敏感的大规模模型。
- 各类下游任务，`Casual`,`seqcls`等，基本都是基础模型`model`后面接对应的`Linear`层，还有损失函数不一样。

