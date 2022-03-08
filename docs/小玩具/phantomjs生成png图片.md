## phantomjs生成png图片

# 概念

### 这是个啥？

[github地址](https://github.com/ariya/phantomjs)

[官网地址](http://phantomjs.org/api/)

- "无头"浏览器 
  
  > 不需要在浏览器中打开运行的JS

### 能做啥?

- 爬虫
- 生成PDF，png图片
- 浏览器自动化测试
- 略

### 怎么安装?

> 请自行google

# 爬虫

> 可以替代的解决方案很多，本文不做介绍使用

# 生成png图片

- 创建phantom.js 
  
  ```javascript
  var page = require('webpage').create(),dTop,dLeft,dWidth,dHeight,imgFilePaht,apiUrl;
  var system = require('system');
  dTop = system.args[1];
  dLeft = system.args[2];
  dWidth = system.args[3];
  dHeight = system.args[4];
  imgFilePaht = system.args[5];
  //url 可以是https://github.com/ariya/phantomjs 或者服务器上静态地址 xx/a.html
  apiUrl = system.args[6];
  
  page.open(apiUrl, function (status) {
   console.log(status)
   if (status !== 'success') {
   console.log('上传失败');
   } else {
   //设置图片截图位置
   page.clipRect = { top:dTop, left:dLeft,width: dWidth, height: dHeight};
   window.setTimeout(function () {
   page.render(imgFilePaht);
   phantom.exit();
   }, 100);
   }
  });
  ```

```
- 编写shell脚本
```

```
#!/bin/bash
path=~/server/phantomjs

# $1 $2 $3 $4 : 分别表示 top、width、height、left
# $5: 表示 apiUrl接口路径
# $6: 表示 imgFilePaht路径（图片存放临时位置）

$path/bin/phantomjs $path/bin/phantom.js $1 $2 $3 $4 $5 $paht$6
```

- 第一种调用api接口

```java

@RestController
@RequestMapping("png")
@Slf4j
public class pngController {

@RequestMapping("test")
public void pngTest(HttpServletResponse response){
    String html="";
    if (html != null) {
        PrintWriter out = null;
        try {
            out = response.getWriter();
            out.write(html);
            out.flush();
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            if (out != null) {
                out.close();
            }
        }
    }
}
```

}

- 第二种静态文件地址(filename)



      FileOutputStream fos = null;
        try {
            File file = new File(filename.trim());
            File parent = file.getParentFile();
            if (!parent.exists()) {
                logger.info("文件目录不存在");
                boolean a = parent.mkdirs();
                logger.info("创建文件目录是否成功:" + a);
            }
            if (!file.exists()) {
                logger.info("文件不存在");
                boolean b = file.createNewFile();
                logger.info("创建文件是否成功:" + b);
            }
            file.setWritable(true);
            fos = new FileOutputStream(file);
            FileChannel channel = fos.getChannel();
            ByteBuffer src = Charset.forName("utf8").encode(html);
            // 字节缓冲的容量和limit会随着数据长度变化，不是固定不变的
            logger.info("初始化容量和limit：" + src.capacity() + ","
                    + src.limit());
    
        }catch(Exception ex){
    
        }finally{
            fos.close
        }




## 备注

- phantomjs最新版本停留在2.1.1，github的star可以看出是一个很多人喜欢的项目
  
