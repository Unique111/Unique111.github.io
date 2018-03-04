---
title: REM——随着窗口大小变化而变化
date: 2017-09-11
categories: ['css']
tags:
---
一般来说，使用rem的时候，都会在html上设定font-size的大小，其他元素的值相对于html的字体大小进行计算。

所以，要实现页面的元素的大小随着窗口的大小变化而变化的效果，只需要在window上绑定resize事件，执行一些计算，去修改html的font-size大小即可。

<!-- more -->

代码如下：
``` javascript
{{#if remLayout}}
<style id="rem-style">
</style>
<script>
/**
* 使用 rem 布局不要使用 text-size-adjust 改变字体效果，
* 在 <meta name="viewport"> 之后使用该代码
* @param {Number} designCSSWidth       设计基准CSS宽度，设备宽度 / dpr
* @param {Number} baseFontSize         基准字体大小，当前 innerWidth 和 designCSSWidth 宽度相同时，html 的 font-size px 值
* @param {Number} maxScale             最大的缩小放大倍数
*/
(function rem(designCSSWidth, baseFontSize, maxScale){
  designCSSWidth = designCSSWidth || 750;
  baseFontSize = baseFontSize || 100;
  maxScale = maxScale || 1.5;
  var w = window;
  var d = document;
  var on = 'addEventListener';
  var html = d.documentElement;
  var maxSize = maxScale * baseFontSize;
  var remStyle = d.getElementById('rem-style');
  function setRem() {
    var isVertical = w.innerWidth < w.innerHeight;
    var width = w.innerWidth;
    if (!isVertical) {
      width = 375;
      remStyle.innerText = 'body { max-width: ' + width + 'px}';
    } else {
      remStyle.innerText = '';
    }
    var size = baseFontSize * width / designCSSWidth;
    html.style.fontSize = Math.min(size, maxSize) + 'px';
  }
  w[on]('resize', setRem, false);
  setRem();
})(375, 50);
</script>
{{/if}}
```
