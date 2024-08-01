---
layout: post
title: Rails grape 项目集成JWT验证
description: "Rails grape 项目集成JWT验证"
keywords: "Rails grape JWT"
categories: "Rails"
background: '/assets/images/bg-post.jpg'
---

## JWT简介

官方描述：

> JSON Web Tokens are an open, industry standard RFC 7519 method for representing claims securely between two parties.
> JWT.IO allows you to decode, verify and generate JWT.
我们来看看 `jwt` 生成的东西

```
aaaaa.bbbbbbb.ccccccc
```

第一部分是 `header`， 第二个部分是 `payload`， 第三部分是 `signature`

#### Header 部分

`header` 部分解密之后为两部分，例如:

```json
{
  "typ": "JWT",
  "alg": "HS256"
}
```

#### Payload部分

`payload` 部分为 `JWT` 声明。 声明有三种类型 - 私人，公开和注册

* 注册声明是其名称被保留但不是强制性使用的索赔。 例如 - iss，sub，aud等

* 私人声明是双方同意的名称，可能与其他公开声明相冲突。 必须谨慎使用。

* 公开声明 - 根据我们的认证要求（如用户名，用户信息等）可以创建的声明。

```json
{
  "iss": "sitepoint.com",
  "name": "Devdatta Kane",
  "admin": true
}
```
这个被 `encode` 之后就变成一串 `hash`，成为`jwt token`第二部分

#### Signature 部分

`Signature` 也可能是最重要的部分, 它是由 `Header`，`Payload`和 `Secret` 生成的一串 `hash`。我们通过具有 `secret` 的 `HMACSHA256` 功能作为服务器端密码运行头部和有效负载的组合字符串。 像这样：

```
require "openssl"
require "base64"
var encodedString = Base64.encode64(header) + "." + Base64.encode64(payload);
hash  = OpenSSL::HMAC.digest("sha256", "secret", encodedString)
```

由于只有服务器知道这个 `secret` ，所以没有人可以篡改 `payload`，并且服务器可以使用 `signature` 来检测任何篡改。

## JWT in Rails with Grape

#### 1. 首先添加 `gem 'jwt'` 到 `Gemfile`, 然后 `bundle install`

#### 2. 然后在 `lib / json_web_token.rb` 中创建一个名为 `JsonWebToken` 的类。 该类将封装JWT令牌的编码和解码逻辑。 像这样：

```ruby
require 'jwt'

class JsonWebToken
  # Encodes and signs JWT Payload with expiration
  def self.encode(payload)
    payload.reverse_merge!(meta)
    JWT.encode(payload, Rails.application.secrets.secret_key_base)
  end

  # Decodes the JWT with the signed secret
  def self.decode(token)
    JWT.decode(token, Rails.application.secrets.secret_key_base)
  end

  # Validates the payload hash for expiration and meta claims
  def self.valid_payload(payload)
    if expired(payload) || payload['client'] != meta[:client]
      return false
    else
      return true
    end
  end

  # Default options to be encoded in the token
  def self.meta
    {
      expired_at: 7.days.from_now.to_i,
      client: 'wechat_app',
    }
  end

  # Validates if the token is expired by exp parameter
  def self.expired(payload)
    Time.at(payload['expired_at']) < Time.now
  end
end
```

#### 3. 为 `grape` 添加一个 `helper` 方法，如下：

```ruby
module Helpers
  module AuthenticationHelper

    # Returns 401 response. To handle malformed / invalid requests.
    def unauthenticated!
      error! '401 Unauthorized', 401
    end

    # Validates the token and user and sets the @current_user scope
    def authenticate!
      if !payload || !JsonWebToken.valid_payload(payload.first)
        return unauthenticated!
      end
      unauthenticated! unless current_user
    end

    private
    # Deconstructs the Authorization header and decodes the JWT token.
    def payload
      token = request.headers['Authorization']
      JsonWebToken.decode(token)
    rescue
      nil
    end

    # Sets the @current_user with the access_token from payload
    def current_user
      if payload[0] && payload[0]['user_id']
        @current_user ||= User.find_by_wechat_open_id(payload[0]['user_id'])
      else
        nil
      end
    end
  end
end
```

#### 4. 在用户登录的时候 调用JWT生成一个`access_token`, 以后每次调用API加上Header `Authorization ${access_token}`

```ruby
user = User.find_by(email: params[:email])
if user && user.valid_password?(params[:password])
  { access_token: JsonWebToken.encode({user_id: user.id}) }
end
```

#### 5. 在需要验证用户的`API` 中 `helpers ::Helpers::AuthenticationHelper` 并且在调用逻辑代码之前 执行 `authenticate!`就行了。