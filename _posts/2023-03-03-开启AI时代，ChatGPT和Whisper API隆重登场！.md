---
layout:     post
title:      手把手教你完成 ChatGPT 注册
subtitle:   
date:       2023-02-04
author:     Yezhiwei
header-img: img/WechatIMG38.jpeg
catalog: true
category: java
tags:
    - ChatGPT
---

## 最新动态

ChatGPT 和 Whisper 模型现在已经可以在 OpenAI 的 API 上使用，使开发人员可以获得尖端的语言处理（不仅仅是聊天！）和语音转文本功能。通过一系列系统级别的优化，自去年 12 月以来，OpenAI 团队已经实现了 ChatGPT 的成本降低了 90％；现在正在将这些节省成本传递给 API 用户。开发人员现在可以在 API 中使用 OpenAI 开源的 Whisper large-v2 模型，获得更快速和经济实惠的结果。ChatGPT API 用户可以期待持续的模型改进，并选择专用容量以更深入地控制模型。密切听取了开发人员的反馈，并改进了 API 服务条款，以更好地满足用户的需求。

## ChatGPT and Whisper APIs 的早期用户

### Snap

Snap Inc. 是 Snapchat 的创建者，本周推出了名为 My AI for Snapchat+ 的实验性功能，运行于 ChatGPT API 上。My AI 为 Snapchatters 提供了一个友好、可定制的聊天机器人，提供推荐，并可以在几秒钟内为朋友写出一首俳句。Snapchat 是一个日常通讯和消息传递的平台，拥有 7.5 亿月度活跃用户。

### Quizlet

Quizlet 是一个全球性的学习平台，有超过 6000 万的学生使用它来学习、练习和掌握他们正在学习的知识。在过去的三年里，Quizlet 与 OpenAI 合作，利用 GPT-3 在多个用例中，包括词汇学习和练习测试。随着 ChatGPT API 的推出，Quizlet 推出了 Q-Chat，一个完全自适应的 AI 导师，通过有趣的聊天体验，基于相关学习材料提供自适应问题，与学生互动。

### Instacart

Instacart 正在扩展其应用程序，使顾客可以询问食品并获得富有灵感和可购买的答案。这使用了 ChatGPT 和 Instacart 自己的 AI，结合其来自 75,000 多个零售合作伙伴商店位置的产品数据，帮助顾客发现针对开放式购物目标的想法，例如“如何制作美味的炸鱼卷？”或“什么是我孩子的健康午餐？” Instacart 计划在今年晚些时候推出“Ask Instacart”功能。

### Shop

Shop 是 Shopify 的消费者应用程序，有 1 亿名购物者使用它来寻找并与他们喜欢的产品和品牌互动。ChatGPT API 用于支持 Shop 的新购物助手。当购物者搜索产品时，购物助手会根据他们的请求提供个性化的推荐。Shop的新型AI购物助手将通过扫描数百万个产品来快速找到买家所寻找的商品或帮助他们发现新的商品，从而简化应用内购物流程。

### Speak

Speak 是一款以 AI 为动力的语言学习应用程序，专注于建立通往口语流利的最佳路径。他们是韩国增长最快的英语应用程序，已经使用 Whisper API 来支持一款新的 AI 语音伴侣产品，并快速将其带到全球其他地区。Whisper 针对各个级别的语言学习者提供人类水平的准确性，解锁真正的开放式对话练习和高度准确的反馈。

## ChatGPT API

模型：今天 OpenAI 团队发布的 ChatGPT 模型系列中，gpt-3.5-turbo 是 ChatGPT 产品中使用的同一模型。它的价格为每 1k 令牌 0.002 美元，比现有的 GPT-3.5 模型便宜 10 倍。对于许多非聊天使用案例，它也是最好的模型-已经看到早期测试者从 text-davinci-003 迁移到 gpt-3.5-turbo 时只需要对他们的提示进行少量调整。

API：传统上，GPT 模型会消耗无结构的文本，该文本以“标记”序列的形式呈现给模型。ChatGPT 模型相反，会消耗一系列消息以及元数据。 （对于好奇的人来说：在底层，输入仍然以“标记”序列的形式呈现给模型消耗；模型使用的原始格式是一种称为 Chat Markup Language（“ChatML”）的新格式。）

### 代码示例

#### 请求

```shell
curl https://api.openai.com/v1/chat/completions
  -H "Authorization: Bearer $OPENAI_API_KEY"
  -H "Content-Type: application/json"
  -d '{
  "model": "gpt-3.5-turbo",
  "messages": [{"role": "user", "content": "What is the OpenAI mission?"}]
}'
```

#### 响应

```json
{
  "id": "chatcmpl-6p5FEv1JHictSSnDZsGU4KvbuBsbu",
  "object": "messages",
  "created": 1677693600,
  "model": "gpt-3.5-turbo",
  "choices": [
    {
      "index": 0,
      "finish_reason": "stop",
      "messages": [
        {
          "role": "assistant",
          "content": "OpenAI's mission is to ensure that artificial general intelligence benefits all of humanity."
        }
      ]
    }
  ],
  "usage": {
    "prompt_tokens": 20,
    "completion_tokens": 18,
    "total_tokens": 38
  }
}
```

#### Python 示例

```python
import openai

completion = openai.ChatCompletion.create(
  model="gpt-3.5-turbo", 
  messages=[{"role": "user", "content": "Tell the world about the ChatGPT API in the style of a pirate."}]
)

print(completion)
```

## ChatGPT 升级

OpenAI 团队不断改进 ChatGPT 模型，并希望将这些增强功能也提供给开发人员。使用 gpt-3.5-turbo 模型的开发人员将始终获得 OpenAI 团队推荐的稳定模型，同时仍然具有选择特定模型版本的灵活性。例如，今天OpenAI 团队发布了 gpt-3.5-turbo-0301 版本，它将在至少6月1日之前得到支持，并且 OpenAI 团队将在 4 月将 gpt-3.5-turbo 更新到新的稳定版本。模型页面将提供转换更新。

## 专用实例

OpenAI 团队现在还提供了专用实例，为那些想要更深入地控制特定模型版本和系统性能的用户。默认情况下，请求在与其他用户共享的计算基础设施上运行，他们需要按请求付费。 API 运行在 Azure 上，通过专用实例，开发人员将按时间段付费，以获得为服务他们的请求保留的计算基础设施分配。

开发人员可以完全控制实例的负载（更高的负载可以提高吞吐量，但会使每个请求变慢），选择启用诸如更长上下文限制之类的功能，并能够固定模型快照。

独立实例对于每日运行超过约 4.5 亿令牌的开发人员可能会经济合理。此外，它可以直接针对硬件性能优化开发人员的工作负载，相对于共享基础架构，可以大大降低成本。如果您对独立实例有疑问，请与 OpenAI 团队联系。

## Whisper API

OpenAI 团队在 2022 年 9 月公开了语音转文本模型 Whisper，受到了开发者社区的极高赞誉，但运行起来可能有些困难。现在已经通过 API 提供了 large-v2 模型，使用户可以方便地按需使用，价格为每分钟 $0.006。此外，OpenAI 团队高度优化的服务堆栈可以确保比其他服务更快的性能。

Whisper API 可以通过“转录”（将源语言转录）或“翻译”（将语音转录为英文）终端使用，并且接受多种格式（m4a，mp3，mp4，mpeg，mpga，wav，webm）

### 代码示例

#### 请求

```shell
curl https://api.openai.com/v1/audio/transcriptions \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: multipart/form-data" \
  -F model="whisper-1" \
  -F file="@/path/to/file/openai.mp3"
```

#### 响应

```json
{
  "text": "Imagine the wildest idea that you've ever had, and you're curious about how it might scale to something that's a 100, a 1,000 times bigger..."
}
```

#### Python 示例

```python
import openai

file = open("/path/to/file/openai.mp3", "rb")
transcription = openai.Audio.transcribe("whisper-1", f)

print(transcription)
```

## 开发者关注点

过去六个月里，OpenAI 团队一直在收集 API 用户的反馈，以了解如何更好地为他们服务。已经做出了具体的改变，包括：

- 除非用户选择加入，否则不再使用通过 API 提交的数据用于服务改进（包括模型训练）；
- 为 API 用户实施默认的 30 天数据保留政策，根据用户需求提供更严格的数据保留选项；
- 取消预发布审查（通过改进自动监控解锁）；
- 改进开发者文档；
- 简化服务条款和使用政策，包括关于数据所有权的条款：用户拥有模型的输入和输出。

在过去的两个月中，OpenAI 团队的在线时间未能达自己的期望和用户的期望。OpenAI 的工程团队现在的首要任务是确保生产用例的稳定性—— OpenAI 团队知道确保AI惠及所有人类需要成为一个可靠的服务提供者。请继续监督 OpenAI 团队在未来几个月中改善在线时间！

OpenAI 团队相信，AI 可以为所有人提供令人难以置信的机会和经济赋权，而实现这一点的最佳方式是允许每个人都可以使用它进行开发。希望今天宣布的变化将带来许多应用程序，每个人都可以从中受益。现在开始使用 ChatGPT 和 Whisper 构建下一代应用程序吧！

## 小结

未来 AI 能力会越来越平民化，降低普通开发者使用 AI 的成本。如果你有好的创意和想法，AI 的能力将成为最底层的能力，提供了无限的可能性。只有你想不到的，没有做不到的💪💪

原文地址：https://openai.com/blog/introducing-chatgpt-and-whisper-apis

## 推荐阅读

[不需要编写代码，也能成为Hive SQL面试高手？ChatGPT告诉你...](https://mp.weixin.qq.com/s/jRq8YqeVXrcNItFjK9-p2g)
[ChatGPT 数据仓库实战：Kaggle 酒店入住数据分析与维度建模](https://mp.weixin.qq.com/s/TeWeJBPYKtABUN7HdtNQmQ)
[OpenAI 快速入门（附代码）](https://mp.weixin.qq.com/s/rvVuVpkqqiiumCeYrMN4EQ)

