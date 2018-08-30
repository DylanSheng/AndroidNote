<h1>View的事件体系</h1>
<h2>3.1 View基础知识</h2>
ViewGroup继承了View，意味着View本身就可以是单个控件也可以是多个控件组成的一组控件。
Button是个View，LinearLayout不但是一个View而且还是一个ViewGroup。

Left, Right, Top, Bottom是坐标。
x,y是左上角的坐标。
translationX, translationY是左上角相对于父容器的偏移量。
View在平移的过程中，top和left并不会发生变化，改变的是x,y,translationX和translationY。
