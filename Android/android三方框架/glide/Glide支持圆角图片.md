最近项目上遇到一个需求图片实现圆角。可能你会说这个很简单啊。直接使用Glide圆角变换不就完成了。还值得专门笔记记录吗？

问题：在使用下面的代码加载圆角图片的时候没有效果

```kotlin
val roundedCorners= RoundedCorners(30)
var options = RequestOptions().transform(roundedCorners) 
Glide.with(this)
.load("http://mserver.wjdev.chinamcloud.cn/cms/mrzd/upload/Image/mrtp/2019/12/08/1_25f95541a8a04f7eb549b6cf33de808e.jpg")
.apply(options)
.into(image_test_glide_circle_radius)
```

经过网上查阅资料发现原来是在使用圆角的时候不能设置ImageView的scaleType为centerCrop。在取消centerCrop属性后确实能正常加载图片了。但是图片却不能按照我们想要的缩放模式进行显示了。纳尼！难道为了一个圆角我们就要放弃图片的显示方式设置。这个明细是不合理的。

那么到底该如何兼容两者呢？通过源码阅读发现在调用into的时候重现添加了一个图片变换。

```java
public ViewTarget<ImageView, TranscodeType> into(@NonNull ImageView view) {
    Util.assertMainThread();
    Preconditions.checkNotNull(view);

    BaseRequestOptions<?> requestOptions = this;
    if (!requestOptions.isTransformationSet()
        && requestOptions.isTransformationAllowed()
        && view.getScaleType() != null) {
      // Clone in this method so that if we use this RequestBuilder to load into a View and then
      // into a different target, we don't retain the transformation applied based on the previous
      // View's scale type.
      switch (view.getScaleType()) {
        case CENTER_CROP:
          requestOptions = requestOptions.clone().optionalCenterCrop();
          break;
        case CENTER_INSIDE:
          requestOptions = requestOptions.clone().optionalCenterInside();
          break;
        case FIT_CENTER:
        case FIT_START:
        case FIT_END:
          requestOptions = requestOptions.clone().optionalFitCenter();
          break;
        case FIT_XY:
          requestOptions = requestOptions.clone().optionalCenterInside();
          break;
        case CENTER:
        case MATRIX:
        default:
          // Do nothing.
      }
    }

    return into(
        glideContext.buildImageViewTarget(view, transcodeClass),
        /*targetListener=*/ null,
        requestOptions,
        Executors.mainThreadExecutor());
  }
```

