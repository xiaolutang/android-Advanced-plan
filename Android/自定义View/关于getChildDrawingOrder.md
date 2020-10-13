# android 绘制流程之修改子View的绘制顺序

一般而言我们不需要指定View的绘制顺序，它会按照[0~childCount)进行绘制，但是在tv app 中需要对焦点View进行放大高亮显示，但是却被后面的View遮挡住了。

