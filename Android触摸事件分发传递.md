### Android触摸事件分发传递

Android研发过程中回经常遇到ViewGroup和多个View嵌套的问题，例如ViewPager中嵌套Fragment，而Fragment中有个横向滚动的RecyclerView，结果发现横向的RecyclerView直接触发了外侧ViewPager的滑动事件。如果想了解冲突的原因以及找到解决办法，就必须对View的事件传递机制有深入的理解才行。