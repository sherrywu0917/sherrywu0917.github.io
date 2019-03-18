---
title: core-decorators源码简析——throttle
date: 2018-06-25 16:14:08
tags: [core-decorators, decorator, decorators源码简析——debounce, js]
---
### @throttle
节流函数：函数节流能使得连续的函数执行，变为**固定时间段**间断地执行。

### 使用实例
例如在监听窗口滚动事件时，稍微滚动一下，就会触发多次onscroll事件，去更新位置信息。如果onscroll事件是去进行dom操作，频繁地更新dom可能会导致低版本浏览器卡死。所以希望可以间隔一段时间，再去执行回调函数。
应用`throttle`函数，设置wait时间为1000ms，可以保证回调函数至少每隔1000ms执行一次。
``` js
import { throttle } from 'core-decorators';

class Scroller {

  position = '';

  @throttle(1000, {leading: false})
  updatePosition() {
    this.position = window.scrollY;
  }

  bindEvent() {
    window.onscroll = () => {
        this.updatePosition()
    };
  }
}
```

### 源码实现
#### 参数
在调用的时候支持传入wait、options.leading、options.trailing三个参数：
- wait: 等待执行的时间，单位ms，默认值300
- {leading}: 绑定的函数在节流开始前执行，默认值true，触发事件刚开始时执行回调
- {trailing}: 绑定的函数在节流开始后执行，默认值true，触发事件结束时执行回调

#### 具体实现
``` js
function handleDescriptor(target, key, descriptor, [wait = DEFAULT_TIMEOUT, options = {}]) {
  const callback = descriptor.value;

  if (typeof callback !== 'function') {
    throw new SyntaxError('Only functions can be throttled');
  }

  if (options.leading !== false) {
    options.leading = true
  }

  if(options.trailing !== false) {
    options.trailing = true
  }

  return {
    ...descriptor,
    value() {
      // 同debounce，会返回this的Symbol('__core_decorators__')属性: 为Meta实例
      const meta = metaFor(this);
      const { throttleTimeoutIds, throttlePreviousTimestamps } = meta;
      const timeout = throttleTimeoutIds[key];
      // 上次函数执行时的时间戳
      let previous = throttlePreviousTimestamps[key] || 0;
      const now = Date.now();

      // options.trailing为true时，将待执行函数callback的参数赋值给meta.throttleTrailingArgs，保证在setTimeout回调函数中可以取到参数
      if (options.trailing) {
        meta.throttleTrailingArgs = arguments;
      }

      // 第一次调用，并且leading参数为false时，将上次执行时间戳设为当前
      // trailing为true时，leading为false，throttlePreviousTimestamps[key]始终被赋值为0
      if (!previous && options.leading === false) {
        previous = now;
      }

      // leading为false的时候，previous=now, remaining时间始终等于wait，始终不会在节流函数开始时执行回调函数
      // leading为true的时候，第一次调用，previous为0，remaining为负值；再次调用，previous为上次回调执行时间，若距离上次执行时间已经等待了wait毫秒，则remaining<=0
      const remaining = wait - (now - previous);

      //{leading: true} remaining<=0的时候，会在节流开始前执行callback函数
      if (remaining <= 0) {
        clearTimeout(timeout);
        delete throttleTimeoutIds[key];
        // 更新上次执行时间戳
        throttlePreviousTimestamps[key] = now;
        // 执行回调函数
        callback.apply(this, arguments);
      } else if (!timeout && options.trailing) {
        // 当前没有timeout对象，并且trailing为true
        // {trailing: true, leading: false} 设置remaining = wait毫秒后去执行回调函数
        // {trailing: true, leading: true} 第二次调用才会设置remaining=wait - (now - previous)毫秒后去执行回调函数
        throttleTimeoutIds[key] = setTimeout(() => {
          // 更新上次执行时间戳，leading为false，throttlePreviousTimestamps[key]始终被赋值为0
          throttlePreviousTimestamps[key] = options.leading === false ? 0 : Date.now();
          // 删掉timeout对象
          delete throttleTimeoutIds[key];
          // 执行回调函数
          callback.apply(this, meta.throttleTrailingArgs);
          // 防止内存泄露
          meta.throttleTrailingArgs = null;
        }, remaining);
      }
    }
  };
}
```

> 结合了时间戳和定时器setTimeout两种方式来实现throttle函数:
> - leading为true时，主要根据时间戳来记录上次执行时间，每次触发事件时，判断当前时间距离上次执行时间是否已经等待了wait毫秒，如果是，则立即执行
> - trailing为true时，主要通过setTimeout定时器来设定间隔某个时间段才去执行
>   * leading为false，定时**wait毫秒(从当前时间算起)**后去执行
>   * leading为true时，定时**距离上次执行累计wait毫秒**后去执行
> - 每次执行都会记录下当前执行时间，用来控制时间间隔




