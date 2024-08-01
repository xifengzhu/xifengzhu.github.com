---
layout: post
title: 使用redis存储微信小程序form_id
keywords: "wechat form_id redis zremrangebyscore"
background: '/assets/images/bg-post.jpg'
---


> 当用户在小程序内发生过提交表单行为且该表单声明为要发模板消息的，开发者需要向用户提供服务时，可允许开发者向用户在7天内推送有限条数的模板消息（1次提交表单可下发1条，多次提交下发条数独立，相互不影响）

小程序发送模板消息只能通过form_id(订单除外，订单的pepay_id可以发送三条模板消息)，并且form_id 的有效期只有7天，那如何存储小程序的form_id，并且能保证每次拿到的都是未过期的form_id呢？
下面就是围绕这个问题来给一个解决方案。

## 方案

### 方案一
存储：新建一个 `form_ids` 表，跟用户相关联。即：`User has_many form_ids`

使用：然后通过某种机制清理过期的 `form_id` 记录，并且在使用过一个 `form_id` 记录之后自动删除记录。

### 方案二
存储：在 `User` 上添加一个字段：`form_ids`, 存储数组： `[{created_at: ${form_id}}]`

使用：从中取有效的 `form_id`, 用完就删除该记录并且通过某种机制清理无效的 `form_id`

### 方案二：
存储：存在 `redis` 里面，使用 `redis` 的自动过期机制

使用：直接从`redis` 里面取

## 分析
方案1：需要建表来存储。而且需要处理过期 `form_id`，并且需要做到用完之后销毁。

方案2：使用字段的方式，并且用完之后需要去除使用过的 `form_id`, 更新字段。

方案3：使用 `redis` 不影响现有的表结构，可以跟业务解耦；若使用用户的`id` 作为键，那 `form_ids` 就是一个列表，`redis` 只能队一个键设置过期，并不能对键中的某个值设置过期。

若是能解决 `redis` 过期的问题，那方案3就是一个最合适的方案，而且可以做成一个 `form_id` 收集服务，容易在项目间移植。

## 最终的思路与实现方案

### 思路

使用 `redis` 有序集合 `sorted Set`。每个有序集合 的成员都关联着一个 `score`，这个 `score` 用于把有序集合中的成员按最低到最高排列。如果我们将 `score` 设置成时间戳，让后通过 `zremrangebyscore` 来删除过期数据。这样就通过曲线救国的方式解决方案3的问题。

### 实现代码

```ruby
module Wechat
  class FormId

    @@redis_connection = Redis.new(host: 127.0.0.1, port: 6379, db: 1)

    @@redis = Redis::Namespace.new("wechat_form_id", :redis => @redis_connection)


    def self.save(user_id, form_id)
      @@redis.zadd(user_id, Time.current.to_i, form_id)
    end

    def self.get_one(user_id)
      @@redis.zrangebyscore(user_id, seven_days_ago, Time.current.to_i, limit: [0,1])
    end

    def self.get_all(user_id)
      @@redis.zrangebyscore(user_id, seven_days_ago, Time.current.to_i)
    end

    def self.delete(user_id, form_id)
      @@redis.zrem(user_id, form_id)
    end

    def self.pop(user_id)
      form_id = get_one(user_id)&.first
      delete(user_id, form_id) if form_id
      form_id
    end

    # form_id 有效期为7天
    def self.delete_expire(user_id)
      @@redis.zremrangebyscore(user_id, 0, seven_days_ago)
    end

    def self.seven_days_ago
      (Time.current - 7.days).to_i
    end
  end
end

```

保存 `form_id`: `Wechat::FormId.save(${user_id}, ${form_id})`

取出 `form_id`: 当需要使用 `form_id` 来发模板消息的时候，调用 `Wechat::FormId.pop(${form_id})`即可

处理过期：使用定时任务跑 `Wechat::FormId.delete_expire(${user_id})`