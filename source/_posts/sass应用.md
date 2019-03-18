---
title: SASS使用二三事
date: 2017-08-28 18:15:21
tags: [sass, css]
---

### sass管理z-index

z-index的值和上下文有关系，在复杂布局中要跟踪z-index比较困难，z-index的取值范围很广，很容易出错，所以可以使用sass预处理器去统一管理。

#### 定义浮层
首先，我们可以借助sass 3.3引入的Map定义一个数据结构，key代表了不同浮层类型，value即对应的z-index值：
``` scss
    $z-layers: (
        'toast':          4000,
        'modal':          3000,
        'dropdown':       2000,
        'mask':           1000,
        'default':         1,
        'below':          -1,
        'bottomless-pit': -10000
    );
```
#### 定义函数
``` scss
@function z($layer) {
    @return map-get($z-layers, $layer);
}
```
使用了sass的map-get方法，如果$layer参数存在于$z-layers中，会返回对应的value值，如z('toast')，返回对应的值为4000；若不存在于$z-layers中，则返回null。属性为null时，sass的编译不会输出。
也可以借助map-has-key方法检查元素是否存在，对于不存在的元素使用@warn指令输出警告信息，方便我们在开发的时候发现问题。
``` scss
@function z($layer) {
    @if not map-has-key($z-layers, $layer) {
        @warn "No z-index found in $z-layers map for `#{$layer}`. Property omitted.";
    }

    @return map-get($z-layers, $layer);
}
```

#### 使用方法
``` scss
//function方式
.m-mask {
    z-index: z('mask');
}

//mixin方式
.m-mask {
    @include z('mask');
}
```
对于单个属性来说，建议采用function的定义方式，比起mixin方式，使用起来更加清晰、简单。
<!-- more -->

#### 嵌套层级
在层级关系比较多的情况下，单一层级可能无法满足我们的需求，如弹窗里面还有很多的层级关系时，这个时候，我们可以使用嵌套的层级定义方式，针对modal进行再扩展，对modal里面的元素进一步定义层级数值。
``` scss
    $z-layers: (
        'toast':          4000,
        'modal':          (
            "base": 3200,
            "close": 3100,
            "header": 3050,
            "footer": 3000
        ),
        'dropdown':       2000,
        'mask':           1000,
        'default':         1,
        'below':          -1,
        'bottomless-pit': -10000
    );
```
想要定义$z-layers中modal内部的层级关系，可以用嵌套的Map去设置，如上所示。要处理嵌套的层级关系，对应的z函数可以是：
``` scss
@function map-has-nested-keys($map, $keys...) {
  @each $key in $keys {
    @if not map-has-key($map, $key) {
      @return false;
    }
    $map: map-get($map, $key);
  }
  @return true;
}
@function map-deep-get($map, $keys...) {
  @each $key in $keys {
    $map: map-get($map, $key);
  }
  @return $map;
}
@function z($layers...) {
  @if not map-has-nested-keys($z-layers, $layers...) {
    @warn "No layer found for `#{inspect($layers...)}` in $z-layers map. Property omitted.";
  }
  @return map-deep-get($z-layers, $layers...);
}
```
其中map-has-nested-keys方法可以检查元素是否存在于已经定义的$z-layers中：
1. 若$keys只有一个值'toast'，@each只需要循环一次，在循环内，$map被赋值为'toast'对应的z-index值，最后返回true值；
2. 若$keys有两个值'modal'和'base'，@each循环两次，第一次循环，先检查'modal'是否存在于$z-layers中，再将$map赋值为map-get($z-layers, 'modal')，即内部嵌套的modal的map，第二次循环先判断'base'是否存在于$map中，若不存在直接返回false，若存在$map被赋值为'base'对应的z-index值，最后返回true值；
map-deep-get方法用于获得对应的z-index值，思路和map-has-nested-keys方法一致，只是前者返回true/false，后者返回$map值。
具体的调用方式如下所示：
``` scss
.modal {
  position: absolute;
  z-index: z("modal", "base");

  .close-button {
    z-index: z("modal", "close");
  }

  header {
    z-index: z("modal", "header");
  }

  footer {
    z-index: z("modal", "footer");
  }
}
.toast {
  z-index: z("toast");
}
```
#### 另一种z-index管理思路
首先创建一个层级列表，在这个列表中，元素的出现顺序是从低到高，使用sass提供的index方法获取元素在$elements中的顺序，即为该元素的z-index值。
``` scss
$elements: project-covers, sorting-bar, modals, navigation;

.project-cover {
  z-index: index($elements, project-covers);
}
```
输出的z-index为1，与javascript不同，sass的索引值从1开始，就像css一样，css的:nth-child(n)中的n也是从1开始。
``` scss
.project-cover {
  z-index: 1;
}
```
个人觉得，这个方法更简单，适合于轻量级的项目，但灵活性不够好，取值范围受限于index值，对于嵌套的层级关系不友好，扩展性不好，对于较复杂项目更建议用前一种Map的形式来管理。

### sass主题管理
在项目开发中，涉及到不同主题的切换，例如在正文阅读时，有白天、黑夜、蓝色、黄色四种主题，不同主题配色不同，如果直接用css，结构复杂并且很难维护。利用scss提供的变量定义和方法，可以降低开发和维护成本。

#### sass管理颜色
首先，sass可以对整个项目常用的一些颜色进行定义，例如本次项目通用的红色值为#ED6460，则可以在单独的文件_color.scss中定义该色值，其他scss文件通过@import引用。
``` scss
$black: #24211F;
$red: #ED6460;
$blue: #60aaed;

$border-gray: #ededed;
$bg-gray: #f5f5f5;
$btn-gray: #807A73;
```
#### 主题定义
首先，定义一个Map，记录不同的主题和与之关联的颜色，每个主题下细分了不同用途的色值。
``` scss
@import './color.scss';

//主题base设置
$theme: (
    light: (
        bg: #fff,              //背景色
        color: $black,         //正文颜色
        title: #A83A45,        //章节标题颜色
        link: #60AAED,         //底部链接颜色
        border: #ededed,       //正文底部border颜色
        themeBorder: #d8d8d8   //主题切换btn选中状态的border颜色
    ),
    dark: ( …… ),
    blue: ( …… ),
    yellow: ( …… )
);

@each $name, $theme in $theme {
    .theme--#{$name} {
        color: map-get($theme, color);
        background-color: map-get($theme, bg);

        .m-main .content {
            h1, h2 {
                color: map-get($theme, title);
            }
        }

        .link {
            color: map-get($theme, link);
        }
        // ... 其他涉及主题配色的选择器
    }
}
```
通过@each去遍历Map，.theme--#{$name}编译后会生成.theme-light, .theme-dark等，在.theme--#{$name} 选择器内部，可以定义该主题下不同元素的样式，具体颜色可以通过map-get方法获得。
对于button这种，在不同主题下颜色、边框、背景、active状态等都需要改变，可以单独定义。
``` scss
$btn: (
    light: (
        color: $gray1, border: $gray0, bg: transparent,
        color-active: #fff, border-active: $red, bg-active: $red
    ),
    dark: ( …… ),
    blue: ( …… ),
    yellow: ( …… ),
);

@each $name, $theme in $btn {
    .btn--#{$name} {
        color: map-get($theme, color);
        border-color: map-get($theme, border);
        background-color: map-get($theme, bg);
    }

    .btn--#{$name}:active, .btn--#{$name}-active {
        color: map-get($theme, color-active);
        border-color: map-get($theme, border-active);
        background-color: map-get($theme, bg-active);
    }
}
```
将主题颜色与其他不变的样式分离出来进行管理，所有的主题颜色维护在_theme.scss中，从而极大地提高了代码的可维护性。在切换主题的时候，只需要更换相应的类名，尤其在结合react开发时，theme变化时只需要重新setState一下就会重新渲染页面，十分方便。

### 定义通用样式
可以通过sass提供的@mixin, @function等方式定义通用样式，如可以将实现单行\多行文字截断效果的一组样式封装，用@mixin定义line-ellipsis方法，参数为行数和行高。传入行高是为了兼容不支持多行截断的浏览器，计算得出最大高度，防止样式错乱。其中行数的默认值为1，行高的默认值为1.5，可以使用@if,@else去判断行数，根据$num值去返回样式。
``` scss
@mixin line-ellipsis($num: 1, $lineH: 1.5){
     @if $num > 1 {
        display: -webkit-box;
        -webkit-box-orient: vertical;
        -webkit-line-clamp: $num;
        text-overflow: ellipsis;
        line-height: $lineH;
        max-height: $lineH * $num;
        overflow: hidden;
     } @else {
        white-space: nowrap;
        text-overflow: ellipsis;
        line-height: $lineH;
        overflow: hidden;
     }
}
```
在使用的时候，通过@include方法引用，如下所示：
``` scss
    h3 {
        @include line-ellipsis();  //默认单行截断
    }

    .desc {
        @include line-ellipsis(3, rem(38));  //3行截断
    }
```
desc这个类是多行截断，其中行高是rem(38)，调用了sass定义的px转rem的函数，是将38像素转为rem值。借助sass，可以方便地定义将像素转为rem。
``` scss
@function rem($px, $base-font-size: 75px) {
  @if (unitless($px)) {     //unitless(75) => true; unitless(75px) => false
    @return rem($px + 0px);
  } @else if (unit($px) == rem) {  //unit(75px) => px; unitless(1rem) => rem
    @return $px;
  }
  @return ($px / $base-font-size) * 1rem;
}
```
在rem()函数中设置了两个参数$px和$base-font-size，并且给$base-font-size设置了默认值75px。rem布局使用了淘宝的lib.flexible方案，所以默认值为75px。而且在rem()函数中使用了unitless去判断$px是否携带单位，若没带为true，否则为false。若$px没带单位，则通过+0px的方式带上px单位。若$px带单位，用unit获取$px带的单位，若是rem，则直接返回，其他的，与$base-font-size相除得到rem值。
除此之外，可以结合项目需求定义更多的方法，如通用的button样式、适配浏览器分辨率、兼容性等等。


参考链接：
[Module: Sass::Script::Functions](http://sass-lang.com/documentation/Sass/Script/Functions.html)
[A Better Solution for Managing z-index with Sass](http://www.smashingmagazine.com/2014/06/12/sassy-z-index-management-for-complex-layouts/)
[Friendlier colour names with Sass maps](http://erskinedesign.com/blog/friendlier-colour-names-sass-maps/#solving-issue-1-the-naming-of-things)
