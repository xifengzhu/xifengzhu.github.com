---
layout: post
title: "iPhone直接上传图片总是命名为image.jpg的解决方法"
description: "iPhone直接上传图片总是命名为image.jpg的解决方法"
keywords: "CarrierWave canvasResize 上传图片"
categories: "ruby"
background: '/assets/images/bg-post.jpg'

---

经常会遇到一种场景，那就是上传图片，如果是在移动设备上还可以调用到摄像头。

#### 问题
但是在iphone上调用iphone摄像头生成图片的图片文件名都为`image.jp`,如果此时用户连续两次调用摄像头更新图片，那就会出现**图片已经存在的问题**。

> It is correct that the iphone(ipad, etc, i'll just call it iphone from now on) strips exif data. This is also not a bug on the iphone but actually a feature.
> One of the main reasons android users don't like the iphone and iphone users don't like the androids, is because the iphone is very limited (in terms of freedom to change, alter, etc). You can not just run downloaded apps, have limited access to settings, etc.
> This is because the apple strategy is to create a fail-safe product. "If you can not do strange things, strange things will not happen".It tries to protect the user in every way imaginable. It also protects the user when uploading images. In the exif there may be data that can hurt the users privacy. Things like GPS coordinates, but even a timestamp can hurt a user (imagine you uploading a beach picture with a timestamp from a moment you reported in sick with the boss).
> So basically it is a safety meassure to strip all exif data. Myself and a lot of other people do not agree with this strategy, but there is nothing we can do about it unfortunately.
#### 解决方案
首先，iphone摄像头照片（Exchange Image File）是不能为之命名的，所在这个只能通过其他的变通方式解决这个问题

##### 方案1:
将拍照所得的照片 转成 base64 再上传到服务端(比如使用： https://github.com/gokercebeci/canvasResize)

##### 方案2
根据需求确定下图片是否能被覆盖
如果使用CarrierWave, 就会有如下方法

```ruby
  def store_dir
    "uploads/#{model.class.to_s.underscore}/#{mounted_as}/#{model.id}"
  end
```
我们可以看到，其实存储的路径中带有model的Id作为区分，如果用户是对同一对象进行覆盖那肯定是没有问题的。

如果如果需要留下用户的图片，不想覆盖，也可以通过

```ruby
  def filename
    "something.jpg" if original_filename
  end
```
进行重命名文件（比如加上时间戳之类的），这样就不会有冲突啦