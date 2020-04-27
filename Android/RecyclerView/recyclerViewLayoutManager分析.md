1. 寻找一个锚点，
2. 调用detachAndScrapAttachedViews将页面元素和recycler暂时分离
3. 根据锚点信息从上到下，或者从左到右进行布局
   1. 先回收不可加的视图
   2. 对视图进行布局，





recyclerView的核心是LayoutManager