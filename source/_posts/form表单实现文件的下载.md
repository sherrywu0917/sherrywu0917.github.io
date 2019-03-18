---
title: form表单实现文件的下载
date: 2017-06-15 20:19:22
tags: [下载, 兼容]
---
### h5链接a增加download属性
1. download属性

想到最简单的下载文件的方式是
```html
<a href="large.jpg">下载</a>
```
但是实际效果是在浏览器直接浏览图片，而不是下载图片。
如果我们希望点击“下载”链接下载图片，可以增加一个download属性。
```html
<a href="large.jpg" download>下载</a>
```
通过download属性还可以指定下载图片的文件名，如果后缀一样，可以省略。
```html
<a href="large.jpg" download="large_down.jpg">下载</a>
```
<!-- more -->

2. download的兼容性

如果下载的资源是跨域的，在chrome浏览器下可以正常下载，在firefox浏览器下不支持跨域下载。
要判断是否支持download属性，可以使用下面的代码：
```javascript
var isSupportDownload = 'download' in document.createElement('a');
```

3. 兼容浏览器的下载方法

download属性不支持IE，要兼容IE可以通过JS创建一个iframe去下载，如下所示：
```html
<!DOCTYPE html>
<html lang="en">
    <head>
        <meta charset="utf-8">
    </head>
    <body>
        <a id='trigger' href="javascript:;">下载</a>
        <a id='download' download='pic.jpg' style="display: none;">下载</a>
    </body>
    <script type="text/javascript" src="http://yuedust.yuedu.126.net/js/jquery-1.8.1.min.js"></script>
    <script type="text/javascript">
        function download_pic() {
            var codeurl='https://img4.cache.netease.com/news/2017/6/19/20170619094322c3136.jpg';
            if(browserIsIe()){//假如是ie浏览器
              DownLoadReportIMG(codeurl);
            }else{
              $("#download").attr('href', codeurl);
              document.getElementById("download").click();
            }
        }

        function DownLoadReportIMG(imgPathURL) {
            //如果隐藏IFRAME不存在，则添加
            if (!document.getElementById("IframeReportImg"))
                $('<iframe style="display:none;" id="IframeReportImg" name="IframeReportImg" onload="DoSaveAsIMG();" width="0" height="0" src="about:blank"></iframe>').appendTo("body");

            if (document.all.IframeReportImg.src != imgPathURL) {
                //加载图片
                document.all.IframeReportImg.src = imgPathURL;
            }
            else {
                //图片直接另存为
                DoSaveAsIMG();
            }
        }

        function DoSaveAsIMG() {
            //跨域的话IE会提示没权限
            if (document.all.IframeReportImg.src != "about:blank")
                window.frames.IframeReportImg.document.execCommand("SaveAs");
        }
        //判断是否为ie浏览器
        function browserIsIe() {
            if (!!window.ActiveXObject || "ActiveXObject" in window)
                return true;
            else
                return false;
        }

        document.getElementById("trigger").onclick = function(e) {
            e.preventDefault();
            download_pic();
        };
    </script>
</html>
```

### form表单实现文件的下载
```javascript
    handleExport() {
        const { selectedRowKeys } = this.state;
        let config  = {
           action: UserExportUrl,
           key: 'userIdArray[]'
        }
        let iframe = document.createElement('iframe');
        iframe.style.display = 'none';
        let form = document.createElement('form');
        form.action = config.action;
        form.method = 'post';

        let input = document.createElement('input');
        input.type = 'hidden';
        input.name = config.key;
        input.setAttribute('value', selectedRowKeys.join(','));
        form.appendChild(input);

        document.body.appendChild(iframe);
        iframe.contentWindow.document.body.appendChild(form);
        form.submit();
        // window.open(UserExportUrl + '?userIdArray[]=' + selectedRowKeys.join(','));
    }
```

PS: 移动端H5几乎无法实现保存图片到本地，加download属性和加响应头的方式，微信和浏览器都无法保存图片，手机端的chrome和PC效果一致，真是一股清流。还是提示用户自己长按保存图片吧。