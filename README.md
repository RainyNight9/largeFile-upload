# 字节跳动面试官：请你实现一个大文件上传和断点续传

## 大文件上传

### 整体思路

---

#### 前端

前端大文件上传网上的大部分文章已经给出了解决方案，核心是利用 Blob.prototype.slice 方法，和数组的 slice 方法相似，调用的 slice 方法可以返回原文件的某个切片

这样我们就可以根据预先设置好的切片最大数量将文件切分为一个个切片，然后借助 http 的可并发性，同时上传多个切片，这样从原本传一个大文件，变成了同时传多个小的文件切片，可以大大减少上传时间

另外由于是并发，传输到服务端的顺序可能会发生变化，所以我们还需要给每个切片记录顺序

#### 服务端

服务端需要负责接受这些切片，并在接收到所有切片后合并切片

这里又引伸出两个问题

    1.何时合并切片，即切片什么时候传输完成
    2.如何合并切片

第一个问题需要前端进行配合，前端在每个切片中都携带切片最大数量的信息，当服务端接受到这个数量的切片时自动合并，也可以额外发一个请求主动通知服务端进行切片的合并

第二个问题，具体如何合并切片呢？这里可以使用 nodejs 的 读写流（readStream/writeStream），将所有切片的流传输到最终文件的流里

Don't Say so Much,Just Coding!

---

### 前端部分

技术栈：

    1.vue
    2.element-ui

#### 请求逻辑

#### 上传切片

接着实现比较重要的上传功能，上传需要做两件事:

    1.对文件进行切片
    2.将切片传输给服务端

当点击上传按钮时，调用 createFileChunk 将文件切片，切片数量通过文件大小控制，这里设置 10MB，也就是说 100 MB 的文件会被分成 10 个切片

createFileChunk 内使用 while 循环和 slice 方法将切片放入 fileChunkList 数组中返回

在生成文件切片时，需要给每个切片一个标识作为 hash，这里暂时使用文件名 + 下标，这样后端可以知道当前切片是第几个切片，用于之后的合并切片

随后调用 uploadChunks 上传所有的文件切片，将文件切片，切片 hash，以及文件名放入 FormData 中，再调用上一步的 request 函数返回一个 proimise，最后调用 Promise.all 并发上传所有的切片

#### 发送合并请求

这里使用整体思路中提到的第二种合并切片的方式，即前端主动通知服务端进行合并，所以前端还需要额外发请求，服务端接受到这个请求时主动合并切片

---

### 服务端部分

简单使用 http 模块搭建服务端

#### 接受切片

使用 multiparty 包处理前端传来的 FormData

在 multiparty.parse 的回调中，files 参数保存了 FormData 中文件，fields 参数保存了 FormData 中非文件的字段

查看 multiparty 处理后的 chunk 对象，path 是存储临时文件的路径，size 是临时文件大小，在 multiparty 文档中提到可以使用 fs.rename(由于我用的是 fs-extra，它的 rename 方法 windows 平台权限问题，所以换成了 fse.move) 移动临时文件，即移动文件切片

在接受文件切片时，需要先创建存储切片的文件夹，由于前端在发送每个切片时额外携带了唯一值 hash，所以以 hash 作为文件名，将切片从临时路径移动切片文件夹中

#### 合并切片

在接收到前端发送的合并请求后，服务端将文件夹下的所有切片进行合并

由于前端在发送合并请求时会携带文件名，服务端根据文件名可以找到上一步创建的切片文件夹

接着使用 fs.createWriteStream 创建一个可写流，可写流文件名就是切片文件夹名 + 后缀名组合而成

随后遍历整个切片文件夹，将切片通过 fs.createReadStream 创建可读流，传输合并到目标文件中

值得注意的是每次可读流都会传输到可写流的指定位置，这是通过 createWriteStream 的第二个参数 start/end 控制的，目的是能够并发合并多个可读流到可写流中，这样即使流的顺序不同也能传输到正确的位置，所以这里还需要让前端在请求的时候多提供一个 size 参数

其实也可以等上一个切片合并完后再合并下个切片，这样就不需要指定位置，但传输速度会降低，所以使用了并发合并的手段，接着只要保证每次合并完成后删除这个切片，等所有切片都合并完毕后最后删除切片文件夹即可
