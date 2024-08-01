---
layout: post
title: 微信小程序用 web socket 和 rails 的 Action Cable 通讯
background: '/assets/images/bg-post.jpg'
---

最近在做一个商品竞拍的需求，考虑到需要使用微信小程序的 web socket跟rails 的 Action Cable。做了以下尝试，将踩坑的过程记录下来。

## 建立 rails Action Cable 服务端

### 链接设置
由于小程序端的请求是无状态的，无法发送 `cookie` 来鉴权， 所有在请求的时候需要带上一个特殊的 `headers['Authorization']`， 后端采用 `jwt` 进行鉴权。 具体代码如下：

```ruby
module ApplicationCable
  class Connection < ActionCable::Connection::Base
    identified_by :current_user

    def connect
      self.current_user = find_verified_user
    end

    private
    def find_verified_user
      token = request.headers['Authorization']
      if token
        # jwt authentication
      else
        reject_unauthorized_connection
      end
  end
end
```

### 设置频道与接收数据处理逻辑

```ruby
class AuctionChannel < ApplicationCable::Channel
  # 当用户成为此频道的订阅者时调用
  def subscribed
    stream_from "auction_channel_#{params[:id]}"
  end

  def unsubscribed
  end

  def receive(data)
    # 接收收据处理逻辑
  end
end
```

## 微信端链接

建立链接是需要注意的是在 `header` 中加上用来做身份鉴定的 `Authorization`

微信端向服务端发送消息需要将 `command` 包装在 `data` 参数中，可使用的 `command` 有 `[subscribe, message]`, 一个用来订阅频道， 一个用来发送消息。


```javascript
const initSocket = async () => {
  # 建立socket链接
  wx.connectSocket({
    url: "ws://localhost:3000/cable",
    header:{
      'content-type': 'application/json',
      'Authorization': ${current_user.token}
    },
    method:"GET"
  });

  # Rails 的 js 代码对 websocket 进行了一些封装，直接用 socket 连到 ws://localhost:3000/cable 是收不到消息的，要客户端发送一个 "subscribe" 到 websocket 才可以
  wx.onSocketOpen(function() {
    const id = JSON.stringify({channel: "AuctionChannel", id: ${id}});
    wx.sendSocketMessage({
      data: JSON.stringify({command: "subscribe", identifier: id: ${id}}),
    })
  });

  # 客户端发送信息到服务端
  wx.sendSocketMessage({
    const id = JSON.stringify({channel: "AuctionChannel", id: ${id}});
    data: JSON.stringify({
      command: 'message',
      data: JSON.stringify({test: 'ddd'}),
      identifier: id
    })
  })

  # 接受到服务端推送信息处理逻辑
  wx.onSocketMessage(function(res) {
    const response = JSON.parse(res.data);
    if(response.identifier && response.message){
      const auctionRecord = response.message;
      console.log(auctionRecord)
    }
  });

  wx.onSocketError(function(res){
    console.log('WebSocket连接打开失败，请检查！')
  });

  wx.onSocketClose(function(res) {
    console.log('WebSocket 已关闭！')
  })
}
```

## 参考
* [Action Cable 概览](https://ruby-china.github.io/rails-guides/action_cable_overview.html)
* [小程序 WebSocket](https://developers.weixin.qq.com/miniprogram/dev/api/network-socket.html)
