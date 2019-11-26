---
layout: post
title: 用 JavaScript 获取 FLV 信息
date: 2018-12-24 19:20
disqus: y
---

**这是一篇絮絮叨叨自言自语的笔记，请不要抱有过度期待，观看时请保持室内明亮**  

### FLV 官方规格书和一些说明文章  

[Video File Format Specification Version 10](https://www.adobe.com/content/dam/acom/en/devnet/flv/video_file_format_spec_v10.pdf)  
[视音频编解码学习工程：FLV封装格式分析器](https://blog.csdn.net/leixiaohua1020/article/details/17934487)  
[一张图看懂FLV文件格式](https://blog.ibaoger.com/2017/06/04/flv-file-format/)  

### 一个例子
[本帖示例工程](https://github.com/sanlengjingvv/media-info.js)里有一个 example.flv  
用二进制表示`xxd -b example.flv`  
第一个字节是`01000110`，表示为 8 位 unsigned int 是 `70`，在 [ASCII](http://www.asciitable.com/) 中代表字符 `F`。  
第二个字节是`01000110`，表示为 8 位 unsigned int 是 `76`，在 [ASCII](http://www.asciitable.com/) 中代表字符 `L`。  
第三个字节是`01010110`，表示为 8 位 unsigned int 是 `86`，在 [ASCII](http://www.asciitable.com/) 中代表字符 `V`。  
第四个字节是`00000001`，表示为 8 位 unsigned int 是 `1`，表示 FLV 格式的版本号。  
第五个字节是`00000101`，前五位是 `00000`，在 FLV 版本 1 中是保留位没有作用，必须是 0。第六位是 `1`，表示为 boolean 是 `true`，代表 FLV 文件存在音频 tag。第七位是 `0`，保留位，必须是 0。第八位是 `1`，表示为 boolean 是 `true`，代表 FLV 文件存在视频 tag。  
第六到九个字节是 `00000000000000000000000000001001`，表示 32 位 unsigned int 是 `9`，这是 FLV header 最后一部分，代表 FLV header 一共有 9 字节。  

第十到第十三个字节是 `00000000000000000000000000000000` 代表前一个 tag 的长度，因为没有前一个 tag ，这个部分总是 0。  

第十四个字节是 `00010010`，表示为 8 位 unsigned int 是 `18`，代表这个 tag 的类型是 script 。  
第十五到第十七个字节是 `000000000000000101110100`，表示为 24 位 unsigned int 是 `372`，代表这个 tag 的 data 的长度。  
第十八到第二十个字节是 `000000000000000000000000`，第二十一个字节是 `00000000`，代表相对于第一个 tag 时间戳，单位毫秒，貌似是这一个 tag 播放的开始时间？(并没有搞明白)，第一个 tag，所以是零。  
第二十二到二十四个字节是 `000000000000000000000000`，表示为 24 位 unsigned int 是 `0`，代表 StreamID，总是 0，不是道有什么用。  

第 25 到第 396 个字节是这个 tag 的 data，396 = 25 + 372 - 1。  

第 397 到第 400 个字节表示为 32 位 unsigned int 是 `383`，，代表前一个 tag 的 长度，因为 tag header 总是 11 个字节，所以这里 383 = 11 + 372。  

第 401 个字节表示为 8 位 unsigned int 是 `9`，从这里开始是第二个 tag ，9 代表这个 tag 的类型是视频。  
第 412 个字节是 `00010111`，前四位是 `0001`，表示为十进制是 `1`，代表这个 tag 的 data 是关键帧(keyframe)。后四位是 `0111`，表示为十进制是 `7`，代表这个 tag 的 data 是 AVC 编码。  
...
第 465 个字节表示为 8 位 unsigned int 是 `8`，从这里开始是第三个 tag ，8 代表这个 tag 的类型是音频。  
第 476 个字节是 `10101111`，前四位是 `1010`，表示为十进制是 `10`，代表这个 tag 的 data 是 AAC 编码。第五位第六位是 `11`，表示为十进制是 `3`，代表音频采样率是 44kHz。第七位是 `1`，表示音频采样精度是 16Bit。第八位是 `1`，代表音频是立体声(sndStereo)。  

回到 Script tag。FLV 的 Script data 格式是 [Action Message Format](https://www.adobe.com/content/dam/acom/en/devnet/pdf/amf0-file-format-specification.pdf)。  
第 25 个字节是 `00000010`，表示为 8 位 unsigned int 是 `2`，代表第一个 packet 是 String 类型。  
第 26 到第 27 个字节是 `0000000000001010`，表示为 16 位 unsigned int 是 `10`，代表第一个 packet 的长度是 10 字节。  
第 28 到第 37 共 10 个字节表示为 8 位 unsigned int 是 `[111, 110, 77, 101, 116, 97, 68, 97, 116, 97]`，在 [ASCII](http://www.asciitable.com/) 中代表字符 `onMetaData`。  

第 38 个字节表示为 8 位 unsigned int 是 `8`，代表第二个 packet 是 ECMA Array 类型。  
第 39 到 42 个字节表示为 8 位 unsigned int 是 `16`，代表第二个 packet 包含了 16 个键值对，键是 String 类型。  
第 43 到 44 个字节表示为 16 位 unsigned int 是 `8`，代表键的长度。  
第 45 到 52 共 8 个字节表示为 ASCII 是 `duration`。代表视频时长，单位是秒。  
第 53 个字节表示为 8 位 unsigned int 是 `0`，代表第一个键值对的值是 DOUBLE 类型。  
第 54 到第 61 个字节表示为 64 位浮点数是 2.378。`ffmpeg -i example.flv -hide_banner` 给出的 duration 是 00:00:02.38，`mediainfo example.flv` 给出的 duration 是 2 s 192 ms，为啥不一样呢？  
第 62 到 63 个字节表示为 16 位 unsigned int 是 `5`，代表键第二个的长度。  
第 64 到 68 共 5 个字节表示为 ASCII 是 `width`。代表视频分辨率宽度。  
第 69 个字节表示为 8 位 unsigned int 是 `0`，代表第一个键值对的值是 DOUBLE 类型。  
第 70 到第 77 个字节表示为 64 位浮点数是 640。  
...  
第 240 到 245 共 6 个字节是第十个键名 `stereo`。  
第 246 个字节表示为 8 位 unsigned int 是 `1`，代表第十个键值对的值是 Boolean 类型。  
第 247 个字节表示为 8 位 unsigned int 是 `1`，是 `true`，代表 FLV 是立体声(stereo)。  
...  
第 394 到 396 个字节表示为 8 位 unsigned int 是 `[0, 0, 9]`，是 ECMA Array 的结束标记。到这里也是 Script data 的结尾。  


### JavaScript 处理二进制数据  

安装 http-server 作为本地服务器  

```bash
npm install http-server -g
http-server -a 127.0.0.1 -p 9999
```

浏览器打开，http://127.0.0.1:9999/index.html，打开开发者工具，点 Run 按钮，会在 console 中输出 FLV 信息。  

用浏览器中用 `getReader()` 读取二进制数据  

```javascript
fetch(req).then(response => {
    const reader = response.body.getReader()
    reader.read().then(result =>  {
        const chunk = result.value
        parse(chunk)
    })
})
```
console 中的输出是：`Uint8Array(26192)`，文件就是 26192 字节。  

```bash
ls -l example.flv 
```

> -rw-r--r--@ 1 nikoniko  staff  26192 12 23 14:35 example.flv

[ArrayBuffer 使用教程](http://es6.ruanyifeng.com/?search=getReader&x=12&y=6#docs/arraybuffer)  

下面的代码将 Uint8Array 转换成二进制字符串，从这里抄的 [用 ArrayBuffer 来截取二进制数据中的字符](https://github.com/yangkean/blog/issues/5)，文章里有说明  

```javascript
Array.from(uint8Array, (item) => item.toString(2).padStart(8, '0')).join('')
```

### FLV 文件完整性校验  

FLV 文件最后 4 个字节是 PreviousTagSize，所以每个 PreviousTagSize 的值加上每个 PreviousTagSize 本身固定的 4 个字节，在加上 FLV header 固定的 9 个字节，结果应该等于文件大小  

```javascript
let sumPreTagSizes = chunk => {
    let sum = 0

    let preTagEnd = chunk.length

    while (preTagEnd >= (9 + 4)) {
        let preTag = chunk.slice(preTagEnd - 4, preTagEnd)
        let preTagSize = new DataView(preTag.buffer).getUint32()

        sum = sum + preTagSize
        sum = sum + 4  // PreviousTagSize 本身占 4 字节

        preTagEnd = preTagEnd - 4 - preTagSize
    }

    return sum
}
```
