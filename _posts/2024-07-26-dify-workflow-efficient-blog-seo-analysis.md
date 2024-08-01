---
layout: post
title: 基于Dify Workflow打造博客SEO分析工具
subtitle: 学习如何使用Dify Workflow进行博客SEO分析，通过Dify平台的可视化界面轻松创建和管理自动化工作流程，提升网站在搜索引擎结果页面（SERP）中的排名，吸引更多读者和潜在客户。
keywords: Dify Workflow, 大模型
background: '/assets/images/ai-cover.jpg'
---


### 前言

#### SEO重要性

搜索引擎优化（SEO）是提升网站在搜索引擎结果页面（SERP）中排名的关键策略。通过优化内容和技术细节，SEO可以显著提高网站的可见性和流量，从而吸引更多的读者和潜在客户。对于博客来说，良好的SEO实践不仅能增加读者数量，还能提升品牌知名度和影响力。

#### Dify 工作流基本介绍

Dify Workflow是Dify平台中的一个核心功能，允许用户通过可视化界面创建和管理自动化工作流程。用户可以将多个任务和工具集成到一个工作流程中，自动化执行复杂的业务操作。

### 使用Dify Workflow进行SEO分析

对于这个实例中 主要是通过 Workflow 自动化执行关键词研究、内容分析、链接分析等任务，节省时间和精力。

#### 构建dify workflow

##### (1) 创建dify工作流应用

![create-workflow](/assets/images/create-workflow.png)

##### (2) 构建工作流

本应用主要分为三个节点：

**开始**： 接收入参 title, content

**SEO LLM**: 运用大模型对输入的参数 进行大模型(gpt-3.5-turbo)分析，根据prompt 输出相关数据

**输出数据**： 将大模型的输出数据进行返回

![workflow](/assets/images/workflow.png)

##### (3) 配置完成之后，直接发布即可

到此为止我们的flow是可以直接在dify里面执行了。
![dify-result](/assets/images/dify-result.png)

但是怎么集成到 beansmile 的博客系统， 就需要用到dify 提供的另一个能力，提供访问的api能力。

##### (4) 生成基础编排聊天助手API密钥

![dify-app-api-secret](/assets/images/dify-app-api-secret.jpg)
至此，dify flow 准备工作结束，在此小节中我们只需要保存好两个东西：**API密钥**与**API服务器地址**

#### 集成到beansmile 博客系统

基于以上的work flow 的构建之后，dify 提供了一个接口给我们的系统访问。这个时候我们只需要集成dify 的接口就行了。

##### (1) 集成dify 接口代码

于是找到 **公司内部的AI工具** 帮我自动写了一下代码，放在 `rails` 的 lib 下面

```ruby
require 'httparty'
require 'singleton'

module Dify
  class BaseClient
    include HTTParty
    include Singleton

    base_uri 'https://api.dify.ai/v1'

    def initialize
      self.class.headers 'Authorization' => "Bearer #{api_key}"
      self.class.headers 'Content-Type' => 'application/json'
    end

    def run_workflow(inputs: {}, response_mode: 'blocking', user: nil)
      body = {
        inputs: inputs,
        response_mode: response_mode,
        user: user
      }.compact

      response = self.class.post('/workflows/run', body: body.to_json)

      if response.success?
        response.parsed_response
      else
        Rails.logger.error("Dify API request failed: #{response.code} - #{response.body}")
        raise "API request failed: #{response.code} - #{response.message}"
      end
    end

    private

    def api_key
      raise NotImplementedError, "#{self.class} should implement api_key method"
    end
  end

  class Seo < BaseClient
    private
    def api_key
      Rails.application.credentials.dig(Rails.env.to_sym, :dify, :seo_api_secret)
    end
  end
end
```

则调用的伪代码就是：

```ruby
response = Dify::Seo.instance.run_workflow(inputs: {
  title: params[:title],
   content: params[:content],
}, user: current_user.id)
```

这样就是实现了与dify接口的打通。
将生成的代码配置到blog系统，写完博客后就可以应用dify生成的seo信息。

##### (2) 最终在博客系统效果图

![beansmile-blog-result](/assets/images/beansmile-blog-result.png)

以上是beansmile博客系统集成Dify Workflow进行SEO分析的过程和方法。通过Dify Workflow，我们能够自动提取博客内容的关键信息，并生成SEO相关信息，为我们的博客优化带来便利。

### 总结

本文介绍了如何使用Dify Workflow进行SEO分析，通过具体步骤展示了从创建工作流应用到在Beansmile博客系统中集成的全过程。通过Dify Workflow，我们能够自动提取博客内容的关键信息，并自动生成SEO相关信息，如标题、描述和URL Slug，使得博客能够更有效地进行SEO优化。

Dify作为一个中间件工具平台，允许用户通过可视化界面轻松创建和管理LLM（大型语言模型）应用。这使得没有编程经验的用户也能快速搭建自己的助手应用，提升工作效率。例如，通过Dify，市场人员可以轻松创建SEO分析助手，而无需依赖技术团队。对于有编程经验的用户，Dify提供了丰富的API接口，使其可以打造更具定制化和低耦合的LLM能力，实现更复杂的自动化任务。
