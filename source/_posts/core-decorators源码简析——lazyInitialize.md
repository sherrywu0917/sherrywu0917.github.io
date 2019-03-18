---
title: core-decorators源码简析——lazyInitialize
date: 2018-06-14 20:45:45
tags: [core-decorators, decorator, lazyInitialize, js]
---
### @lazyInitialize
`@lazyInitialize`标记的作用是，只有在属性真正使用的时候才会去初始化。适用于一些在未来可能会被用到、也可能不会被用到的属性上。

### 使用实例
下面是[官网](https://github.com/jayphelps/core-decorators#lazyinitialize)给出的例子，可以帮助我们理解该标记的作用。
``` js
import { lazyInitialize } from 'core-decorators';

function createHugeBuffer() {
  console.log('huge buffer created');
  return new Array(1000000);
}

class Editor {
  @lazyInitialize
  hugeBuffer = createHugeBuffer();
}

var editor = new Editor();
// createHugeBuffer() has not been called yet

editor.hugeBuffer;
// logs 'huge buffer created', now it has been called

editor.hugeBuffer;
// already initialized and equals our buffer, so
// createHugeBuffer() is not called again
```

### 源码
``` js
import { decorate, createDefaultSetter } from './private/utils';
const { defineProperty } = Object;

function handleDescriptor(target, key, descriptor) {
  const { configurable, enumerable, initializer, value } = descriptor;

  return {
    configurable,
    enumerable,

    get() {
      // This happens if someone accesses the
      // property directly on the prototype
      if (this === target) {
        return;
      }

      const ret = initializer ? initializer.call(this) : value;

      defineProperty(this, key, {
        configurable,
        enumerable,
        writable: true,
        value: ret
      });

      return ret;
    },

    set: createDefaultSetter(key)
  };
}

export default function lazyInitialize(...args) {
  return decorate(handleDescriptor, args);
}
```

在第一次调用`editor.hugeBuffer`时，真正的初始化发生在这两步：
``` js
    const ret = initializer ? initializer.call(this) : value;
    defineProperty(this, key, {
        configurable,
        enumerable,
        writable: true,
        value: ret
    });
```
**step1**
- 其中`initializer.call(this)`中的`this`是调用该方法的对象，这儿是`Editor`的实例对象`editor`，通过call方法`initializer`方法的this指向`editor`。
- `initializer`是一个返回`createHugeBuffer()`的函数，`console.log(initializer)`的输出如下所示:（什么情况下initializer不存在，取value值？）
``` js
ƒ initializer() {
    return createHugeBuffer();
}
```

**step2**
> - `defineProperty`重新赋值了key值对应的descriptor对象，将初始化后的值ret赋值给对应的value字段，应该是模拟了***在类构造函数执行时，`initializer`返回的值作为属性的值***，***在类构造函数执行后，`initializer`属性被替换为value属性***。

#### 为什么需要`defineProperty`
如果删除上面get方法的`defineProperty`，则editor.hugeBuffer每次都会调用get()方法，进而执行createHugeBuffer()方法，每次都会去重新初始化，所以在属性初始化完成后，需要对value赋值，替换掉`initializer`。
```js
    //defineProperty(this, key, {
    //  configurable,
    //  enumerable,
    //  writable: true,
    //  value: ret
    //});
```

