最近小王项目上遇到一个需求，图片实现圆角。大家可能会想这个问题很简单啊。直接使用Glide圆角变换不就完成了。为什么小王还写了一个笔记呢？

# 问题：在使用下面的代码加载圆角图片的时候没有效果

```kotlin
val roundedCorners= RoundedCorners(30)
var options = RequestOptions().transform(roundedCorners) 
Glide.with(this)
.load("http://mserver.wjdev.chinamcloud.cn/cms/mrzd/upload/Image/mrtp/2019/12/08/1_25f95541a8a04f7eb549b6cf33de808e.jpg")
.apply(options)
.into(image_test_glide_circle_radius)
```

经过网上查阅资料发现原来是在使用圆角的时候不能设置ImageView的scaleType为centerCrop。在取消centerCrop属性后确实能正常加载图片了。但是图片却不能按照我们想要的缩放模式进行显示了。纳尼！难道为了一个圆角我们就要放弃图片的显示方式设置。这个明细是不合理的。

# 原因

设置了ImageView的scaleType为center_crop

# 解决方案

网上看到大多数的处理是直接放弃使用center_crop缩放。显然这个是不合理的。

通过源码阅读发现，当设置了图片的scaleType的时候Glide会默认添加一个图片变换，并且只有最后添加的图片变化会生效。也就是我们前面示例代码中的roundedCorners被center_crop替代了。导致圆角效果失效。

那么该如何解决这个问题呢？Glide提供了一个图片变换类：MultiTransformation<T> 在这个类中会依次调用每一次图片的每一种变换。通过下面的代码即可以解决这个问题

## 方式一：ImageView不自己设置测center_crop而将这个效果交给Glide处理

```kotlin
val roundedCorners= RoundedCorners(30)
var options = RequestOptions().transform(CenterCrop(),roundedCorners)
 Glide.with(this)               .load("http://mserver.wjdev.chinamcloud.cn/cms/mrzd/upload/Image/64522/2020/02/28/1_780793f0fb594ce5bf6a99eefd956d32.jpg")
                .apply(options)
                .into(image_test_glide_circle_radius)
```

## 方式二（不推荐）：Glide和ImageView同时设置center_crop

```kotlin
val roundedCorners= RoundedCorners(30)
var options = RequestOptions().transform(CenterCrop(),roundedCorners)
Glide.with(this)              .load("http://mserver.wjdev.chinamcloud.cn/cms/mrzd/upload/Image/64522/2020/02/28/1_780793f0fb594ce5bf6a99eefd956d32.jpg")
                .override(DisplayUtil.dip2px(this,60f),DisplayUtil.dip2px(this,60f))//
                .apply(options)
                .addListener(object:RequestListener<Drawable>{
                    override fun onLoadFailed(e: GlideException?, model: Any?, target: Target<Drawable>?, isFirstResource: Boolean): Boolean {
                        return false
                    }

                    override fun onResourceReady(resource: Drawable?, model: Any?, target: Target<Drawable>?, dataSource: DataSource?, isFirstResource: Boolean): Boolean {
                        image_test_glide_circle_radius2.setImageDrawable(resource)
                        return true
                    }
                }).submit()
```

在使用方式2的时候要注意：必须调用override方法，原因如下

1. 两者同时处理必须保证待加载的图片和ImageView大小一致，否者图片的center_crop会将圆角裁掉
2. 不使用override可能会导致加载的图片内存过大。

**特别注意：**

在使用MultiTransformation<T>的时候，一定要思考清楚变换的先后关系，比如像上面的例子中如果先进行圆角缩放，在进行center_crop裁剪。很可能将图片的圆角区域直接裁掉导致其失效。所以先进行center_crop变换，在进行圆角处理。

Demo示例：

https://github.com/xiaolutang/androidTool/blob/master/app/src/main/java/com/example/txl/tool/glide/GlideDemoActivity.kt

