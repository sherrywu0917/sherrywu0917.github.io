---
title: css效果
date: 2017-06-30 19:44:13
tags: [css]
---

### 仿古效果
> CSS处理图像效果：仿古效果[https://www.w3cplus.com/css3/vintage-washout.html]

冲洗效果: 通过减轻暗色阴影和改变一些阴影的细节（改变暗色的细节），看上去就是在亮度的范围降低颜色对比度。
- 混合模式：lighten
将lighten混合模式应用于一个重叠元素或者一个伪元素上。你可以在某个元素上使用background-blend-mode:lighten或者使用多个混合模式，也可以在覆盖元素上使用mix-blend-mode:lighten。建议使用多个背景。

- 应用
使用@mixin
```scss
@mixin fade-it($img, $shadow: #536) {
  background: url('#{$img}'), $shadow;
  background-blend-mode: lighten;
}
.apply-base {
  @include fade-it('1.jpg');
}
.apply-unique-shade {
  @include fade-it('2.jpg', #293e78);
}
```
编译后的css
```css
.apply-base {
  background: url('1.jpg'), #536;
  background-blend-mode: lighten;
}
.apply-unique-shade {
  background: url('2.jpg'), #293e78;
  background-blend-mode: lighten;
}
```
