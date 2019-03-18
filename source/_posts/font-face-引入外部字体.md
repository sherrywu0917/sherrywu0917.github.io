---
title: '@font-face 引入外部字体'
date: 2017-06-15 20:16:44
tags: [font-face, css]
---
字体是在页面中呈现内容不可缺少的重要元素，合适的字体能让内容更能抓住用户的注意力。 我们的追求是在多平台上呈现可预知的一致的文字效果给用户，但限于平台的字体支持，我们在使用一些不常见的字体时畏首畏脚。 长久以来，我们解决这个问题，一般都采用图片替换文字的方法。这个方法虽然简单，但是弊端不少：

1. 图片体积一般比较大，如需半透明处理，体积会进一步增大
2. 工作量增加不少
3. 图片放大后可能会失真

随着CSS3的推广，一个通过@font-face自定义字体的技术进入大家的视线中。 这个技术目前正被大量应用于自定义图标的实现。但是很少用来实现自定义字体，尤其是中文字体。

这是因为中文包含很多汉字，所以字体文件的体积一般都比较大。如果用做自定义字体， 页面会先下载字体文件，然后再呈现页面，这会导致加载缓慢，用户的流量被浪费。

然而在我们使用字体时，基本上只用来呈现有限的文字，下载整个字体文件是多余的。 那我们是否可以精简字体文件，让它只包含指定文字的字体信息，来解决问题？答案是可以的。

webfont-pick就是这样一个工具，它使用起来非常简单：

```
npm install webfont-pick -g
# 更多选项请执行 webfont-pick --help 查看
webfont-pick --font=/Library/Fonts/YuppySC-Regular.otf --text="你好，世界！" -o ~/Desktop/webfont
```
<!-- more -->

执行上述命令后，只包含你好，世界！这六个汉字的自定义字体文件会出现在指定的目录，并且生成了一个示例页面，用来说明如何使用。 有了webfont-pick之后，不管是微软雅黑还是方正呐喊都可以放心的应用到页面中。

另外webfont-pick不只可以通过命令行调用，还可以通过程序调用，详情请参考[项目主页](https://github.com/anhulife/webfont-pick)。

注1: webfont-pick目前只支持解析WOFF, OTF, TTF格式的字体

注2: webfont-pick的想法来源于[ICONFONT.cn](http://www.iconfont.cn/webfont/#!/webfont/index)

注3: webfont-pick的实现参考[grunt-webfont](https://github.com/sapegin/grunt-webfont)

## 开发中遇到的问题

中文字体应用到英文上会有问题，字体不是期望的那样。解决方案：英文单独使用英文字体，在设置font-family的时候引入中英文两种字体。
```scss
@font-face{
    font-family: "MS-Mincho";
    src: url('../src/font/MS-Mincho.eot'); /* IE9*/
    src: url('../src/font/MS-Mincho.eot?#iefix') format('embedded-opentype'), /* IE6-IE8 */
    url('../src/font/MS-Mincho.woff') format('woff'); /* chrome、firefox */
}

@font-face{
    font-family: "ST-Regular";
    src: url('../src/font/st-regular.eot'); /* IE9*/
    src: url('../src/font/st-regular.eot?#iefix') format('embedded-opentype'), /* IE6-IE8 */
    url('../src/font/st-regular.woff') format('woff'); /* chrome、firefox */
}

.banner-title {
    font-size: 64px;
    color: #fff;
    line-height: 60px;
    margin-bottom: 20px;
    font-family: ST-Regular, MS-Mincho, sans-serif;
    // font-family: STSongti-SC-Regular;
    
    .en {
        vertical-align: sub;
        letter-spacing: -28px;
        margin-right: -6px;
    }
}
```