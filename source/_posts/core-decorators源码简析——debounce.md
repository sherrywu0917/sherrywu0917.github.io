---
title: core-decorators源码简析——debounce
date: 2018-06-13 20:37:05
tags: [core-decorators, decorator, decorators源码简析——debounce, js]
---
### @debounce
防抖动函数：当调用函数n秒后，才会执行该动作，若在这n秒内又调用该函数则将取消前一次并重新计算执行时间。

### 使用实例
``` js
import { debounce } from 'core-decorators';

class Editor {

  content = '';

  @debounce(500)
  updateContent(content) {
    this.content = content;
  }

  bindEvent() {
    document.getElementById('editor').oninput = e => {
      this.updateContent(e.target.value)
    }
  }
}
```
``` html
<textarea id="editor"></textarea>
```
例如在监听id为editor的编辑器输入时，每次输入都会触发`oninput`事件，调用`this.updateContent`方法。为了防止频繁地更新content，可以对`updateContent`装饰debounce方法，在输入的过程中不会去更新`this.content`，直到输入完成500毫秒后，再去一次性更新content内容。

### 源码
``` js
import { metaFor } from './private/utils';

const DEFAULT_TIMEOUT = 300;

function handleDescriptor(target, key, descriptor, [wait = DEFAULT_TIMEOUT, immediate = false]) {
  const callback = descriptor.value;

  if (typeof callback !== 'function') {
    throw new SyntaxError('Only functions can be debounced');
  }

  return {
    ...descriptor,
    value() {
      //返回this的Symbol('__core_decorators__')属性: 为Meta实例
      const { debounceTimeoutIds } = metaFor(this);
      //已有的防抖函数调用
      const timeout = debounceTimeoutIds[key];
      //immediate为true且当前无timeout对象，多次调用的时候只在第一次调用时才去执行
      const callNow = immediate && !timeout;
      const args = arguments;

      clearTimeout(timeout);
      // 每次都覆盖之前的防抖函数，执行最后一次传入的callback函数
      debounceTimeoutIds[key] = setTimeout(() => {
        delete debounceTimeoutIds[key];
        if (!immediate) {
          callback.apply(this, args);
        }
      }, wait);

      if (callNow) { //先执行再等待
        callback.apply(this, args);
      }
    }
  };
}
```

#### 调用参数
在调用的时候支持传入wait和options.immediate两个参数：
- wait: 等待执行的时间，单位ms，默认值300
- {immediate}: 绑定的函数是否先执行，默认值false，等待wait时间后再执行；传递的参数为true时先执行

#### Meta对象
``` js
    const { debounceTimeoutIds } = metaFor(this);
    //这个时候只有debounceTimeoutIds被初始化了哦，因为用了lazyInitialize
    console.log(metaFor(this))
```
`metaFor`方法给当前实例增加了`Symbol('__core_decorators__')`属性，值为一个Meta对象。
Meta类如下所示，包括debounce和throttle用到的一些属性：
``` js
class Meta {
  @lazyInitialize
  debounceTimeoutIds = {};

  @lazyInitialize
  throttleTimeoutIds = {};

  @lazyInitialize
  throttlePreviousTimestamps = {};

  @lazyInitialize
  throttleTrailingArgs = null;

  @lazyInitialize
  profileLastRan = null;
}
```
使用了`@lazyInitialize`标记，可以使得每次只初始化被用到的属性，像debounce的时候只用到debounceTimeoutIds属性。console输出Meta对象，结构如图所示，可以看到只初始化了debounceTimeoutIds属性：
<img src="/image/debounce_meta.png" width="360px">

#### debounceTimeout存储
> 当调用被`@debounce`标记的方法时：
1 . setTimeout会延迟执行该函数，返回一个timeout对象；
2 . 返回的timeout对象会被存储在`debounceTimeoutIds[key]`中（以`updateContent`方法为例，`debounceTimeoutIds[key]`这里的key值就是`updateContent`）；
3 . 重复调用该方法会重新创建timeout对象，并覆盖之前的`debounceTimeoutIds[key]`（这里每次调用`updateContent`都会重新给`debounceTimeoutIds['updateContent']`赋值）；

``` js
setTimeout(() => {
    delete debounceTimeoutIds[key];
    if (!immediate) {
      callback.apply(this, args);
    }
  }, wait);
```
执行的结果就是：**每次都重新等待wait时间，再去执行callback方法，callback即是key值对应的方法**。



