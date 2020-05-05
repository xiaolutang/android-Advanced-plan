# 为什么Gldie加载内存占用较小？

Glide的图片加载到内存大部分是通过Downsampler来进行的。而这里面的处理图片主要还是我们常见的通过控制BitmapFactory.Options来达到加载到内存的图片更小的目的。

具体的操作过程如下：

1. 设置BitmapFactory.Options的  inJustDecodeBounds 为true获取图片的原始宽高
2. 根据目标图片的宽高和原始图片的宽高计算出采样率
3. 设置BitmapFactory.Options的  inJustDecodeBounds 为false加载图片

# Glide使用需要注意的地方。

注意

1. 在Fragment中使用的时候最好传递Fragment给Glide#with方法，方便内存回收
2. 在为非View对象加载图片的时候最好调用override方法，设定加载图片的宽高，以便节约内存、
3. 在使用资源文件的时候为了减少内存使用，最好使用Glide进行加载。

