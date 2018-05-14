---
title: 写个Vue插件【urlEncode】
date: 2018-05-14
tags: 
  - JS
  - Vue
---

**src/utils/Common.js**

```js
export default {
  install(Vue, options) {
    Vue.prototype.urlEncode = function(param, key, encode) {
      if (param === null) {
        return '';
      }
      var paramStr = '';
      var t = typeof (param);
      if (t === 'string' || t === 'number' || t === 'boolean') {
        paramStr += '&' + key + '=' + ((encode == null || encode) ? encodeURIComponent(param) : param);
      } else {
        for (let i in param) {
          let k = key == null ? i : key + (param instanceof Array ? '[' + i + ']' : '.' + i);
          paramStr += this.urlEncode(param[i], k, encode);
        }
      }
      return paramStr;
    };
  }
};

```

在 **main.js** 引入并且全局注册

```js
import Common from './utils/Common.js'
Vue.use(Common);
```

接下来就可以直接在其他组件中使用

```js
let data = {
	nickName: 'funsoul',
	age: 18
}

this.urlEncode(data);

// &nickName=funsoul&age=18
```
