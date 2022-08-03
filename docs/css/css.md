
## 不同边距、自号、字体的文段垂直对齐兼容问题(阅读总结)
![](https://raw.githubusercontent.com/captain1023/picGo/master/img/20220802202321.png)
常用垂直居中:
1. 绝对定位,再translate:适合要求不影响父元素的场景
2. flex布局

### 关于字体

![](https://raw.githubusercontent.com/captain1023/picGo/master/img/20220802202445.png)
- HHead/Win Ascent，HHead Acent用在mac系统，而Win Ascent用在windows系统。
- 字体设计之初自带的行间距：typo line gap 与 HHead Line Gap
- 变更font-size大小的时候，其实跟font-size的数值变化完全对应起来的是ascent + descent.字体中的边距就是指descent这一段。这部分距离在字体设计之初就已经定好比例了，不同的字体不一而同

因为 descent这一段在字体设计之初就已经订好了比例,所以使用上面的垂直居中方式都会有如下效果
![](https://raw.githubusercontent.com/captain1023/picGo/master/img/20220802203058.png)
**数字会偏上一点**

#### 常用但有问题的方式
1. 使用flex布局,algin-item:center,字体的line-height = font-size,添加padding适应不同字体的自带边距
问题:padding最小颗粒度是1px,因此达不到精细化控制
```js
// 使用flex 布局， align-items: center，然后将字体的line-height设置得和font-size一样大,之后加padding适应不同字体的自带边距。
<div className="hot-list-item">
    <div
      className={classname('list-item-rank', {
        hot: index < 3,
      })}>
      {index + 1}
    </div>
    <div className="list-item-title">{data.title}</div>
</div>


.hot-list-item {
  display: flex;
  align-items: center;
  padding: 12px 0;

  .list-item-rank {
    box-sizing: border-box;
    width: 36px;
    height: 18px;
    padding-top: 3px;
    text-align: center;
    border-left: 3px solid transparent;
    flex-shrink: 0;
    font-size: 16px;
    line-height: 16px
    color: #999;
    &.hot {
      padding-top: 2px;
      font-size: 17px;
      line-height: 17px;
      color: rgb(255, 94, 94);
    }
  }
  .list-item-title {
    font-family: "PingFang SC"
    font-size: 16px;
    line-height: 24px;
    color: rgb(34, 34, 34);
    .ellipsis();
  }
}
```
2. 1的基础上不使用padding-top,而是通过transform:translateY()来达到精细化控制,之后通过媒体查询适配不同分辨率
问题:解决了更细粒度的控制但是Android和IOS的line-height实现不一致,相同的line-height下,在android和ios所导致的byte-number字体的下边距长度不一样.代码难度很高
#### 略微完善的方案
将需要垂直居中对齐的不同字号和字体文字用一个wrapper包裹起来，之后设置他们align-items: baseline，即以baseline垂直对齐，字号都设置成一样的.之后如果某个字段需要不同大小的字体，再通过transform: scale()来放大缩小达到目的。由于不同字体默认下边距的影响，序号文字所在div的中心并不在文字的中心，可以通过transform-origin来调整至文字中心。为什么不直接设置font-size呢？是因为baseline对齐的情况下，使用font-size缩放字体达不到居中对齐的目的
```html
<div className="hot-list-item">
    <div className="list-item-rank-title-wrapper">
        <div
          className={classname('list-item-rank', {
            hot: index < 3,
          })}>
          {index + 1}
        </div>
        <div className="list-item-title">{data.title}</div>
    </div>
</div>
```
```css
.hot-list-item {
  display: flex;
  align-items: center;
  padding: 4px 0;
  position: relative;
  height: 42px;

  .list-item-rank-title-wrapper {
    display: flex;
    align-items: baseline;
    font-size: 16px;
    height: 16px;
    .ellipsis();

    .list-item-rank {
      box-sizing: border-box;
      font-family: Arial;
      width: 36px;
      text-align: center;
      border-left: 3px solid transparent;
      flex-shrink: 0;
      color: #999;
      transform-origin: 50% 38%;
      &.hot {
        font-family: Arial;
        transform: scale(1.1);
        color: rgb(255, 94, 94);
      }
    }
  
    .list-item-title {
      font-family: "PingFang SC"
      line-height: 17px;
      color: rgb(34, 34, 34);
      .ellipsis();
    }
  }
}
```

总结:
1. 缩放文字大小不止有font-size,还有transform
2. 使用baseline进行对齐能去除字体设计之初自带编剧(Typo Descent)影响
3. transform可以对像素达到0.1px级别控制


## CSS点击穿透问题
问题: 
1. 底部需要一个透明度渐变的蒙层
2. 蒙层会导致其下方元素绑定的点击事件无效

涉及相关css属性:**pointer-events**
```css
pointer-events: auto;           /* 和没有定义pinter-events属性相同,鼠标不会穿透当前层*/
pointer-events: none;           /* 元素永远不会成为鼠标事件的目标,简单来说就是让当前元素的鼠标事件失效*/
pointer-events: visiblePainted; /* SVG only */
pointer-events: visibleFill;    /* SVG only */
pointer-events: visibleStroke;  /* SVG only */
pointer-events: visible;        /* SVG only */
pointer-events: painted;        /* SVG only */
pointer-events: fill;           /* SVG only */
pointer-events: stroke;         /* SVG only */
pointer-events: all;            /* SVG only */
```
**总结**
1. 只是用来禁用鼠标的事件,其他绑定的事件还是会触发的,比如键盘事件
2. 当禁用的元素有父/子元素时,在事件捕获/冒泡节点,事件将在其父/子元素触发.
3. 如果元素有绝对定位,那么它下一层的元素可以被选中,触发相关事件