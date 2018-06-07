---
title: Android 不重签秒打各种渠道包黑科技
date: 2018-06-05 12:27:03
categories: 
- 渠道包
---

### 前言

我们app经常有这样的需求，就是app经常需要投放到不同渠道。这就需要我们给每个app包中添加一些渠道信息，用数据分析统计、业务功能需求等等。

#### 传统打入渠道信息的弊端

有几种打渠道包信息的方法，如
- 利用 Android Studio 自带 productFlavors 方式，配合第三方平台打包，自动打出一堆渠道包。
- 解压拆包，放入渠道信息。

这几种打包方式都有一个共同缺点，**重签名**，这样在每个渠道打包生成的效率上大打折扣。想象下如果遇到即时的生成，下载包这样的业务场景，肯定是实现不了的。
 
###  秒速打出渠道包

<!-- more -->

由于项目业务原因，事实上我们需要打的渠道包的量是非常大的，几百上千都有，且这些渠道包可能需要即时生成下载。那上面的打包方式明显满足不了我们的需求。

#### V1 签名方式打包

这种方式明显契合需求，有什么优势
- 快速打包，只做在包尾写入渠道信息操作。
- 不需要重签名。

我们的秒速读取渠道信息就是由**V1签名方式实现**的，因此需要在 `gradle` 文件 `signingConfigs`  禁用V2签名方式 `v2SigningEnabled false` 

```gradle
signingConfigs {
        debug {
            v2SigningEnabled false
        }
        release {
            v2SigningEnabled false
        }
    }
```

##### 写入渠道信息

写入渠道信息具体步骤如下：
1.  复制 `apk`
2.  找到 `eocd` 数据块
3.  添加渠道信息
4.  添加渠道信息长度
5.  添加魔数

下面以 `go` 代码为例

> 这里说明下，`go` 里面 ` binary.LittleEndian.PutUint()` 是反序列化输入，写入的字节是反过来的。
> 由于我的apk包本身没有注释， `eocd` 的长度是 `22` 字节

```go
func  (c *Controller) writeChannelInfo(pkgFilePath string) {
	// 找eocd开始位置
	_, err1 = pkgFile.Seek(-22, os.SEEK_END)
	if err1 != nil {
		c.logger.Errorln("Seek error", err1)
		return
	}
	eocdMagicBuff := make([]byte, 4)
	_, err1 = pkgFile.Read(eocdMagicBuff)
	if err1 != nil {
		c.logger.Errorln("Read error", err1)
		return
	}
	//0x06054b50 eocd 的开始 4 个字节的反序
	if binary.LittleEndian.Uint32(eocdMagicBuff) != uint32(0x06054b50) {
		c.logger.Errorln("eocd magic error")
		return
	}
	pkgFile, err1 = os.OpenFile(pkgFilePath, os.O_RDWR, 0644)
	commentMagic := uint32(0xeadf03c8)
	// comment总长度=json数据长度+json数据长度位长度(2)+魔术数长度(4)
	totalLen := len(jsonData) + 2 + 4
	fmt.Println("total len", totalLen)
	totalLenBuff := make([]byte, 2)
	binary.LittleEndian.PutUint16(totalLenBuff, uint16(totalLen))
	jsonLenBuff := make([]byte, 2)
	binary.LittleEndian.PutUint16(jsonLenBuff, uint16(len(jsonData)))
	magicBuff := make([]byte, 4)
	binary.LittleEndian.PutUint32(magicBuff, commentMagic)

	// 从comment长度位置开始写
	_, err1 = pkgFile.Seek(-2, os.SEEK_END)
	if err1 != nil {
		c.logger.Errorln("Seek error", err1)
		return
	}

	// 写入comment总长度
	_, err1 = pkgFile.Write(totalLenBuff)
	if err != nil {
		c.logger.Errorln("Write error", err1)
		return
	}

	// 写入json数据
	_, err1 = pkgFile.Write(jsonData)
	if err != nil {
		c.logger.Errorln("Write error", err1)
		return
	}

	// 写入json长度数据
	_, err1 = pkgFile.Write(jsonLenBuff)
	if err != nil {
		c.logger.Errorln("Write error", err1)
		return
	}

	// 写入magic number
	_, err1 = pkgFile.Write(magicBuff)
	if err != nil {
		c.logger.Errorln("Write error", err1)
		return
	}
	pkgFile.Close()
}
```



##### 读取渠道信息

实际就是在 Apk 文件的注释字段里，添加我们自己的渠道信息，绕过了重签名。读取步骤如下
1. 定位到魔数
2. 向前读两个字节，并确定渠道信息长度 len
3. 继续向前读 len 个字节，读取len 长度信息，就是渠道信息了

```java
public class V1SchemeUtil {
    static final String CONTENT_CHARSET = "UTF-8";
    static final int SHORT_LENGTH = 2;
    static final byte[] V1_MAGIC = ByteUtils.hexStringToBytes("C803DFEA");

    /**
     * read channel from apk
     *
     * @param file Apk file path
     * @return
     * @throws Exception
     */
    public static String readChannel(File file) throws Exception {
        RandomAccessFile raf = null;
        try {
            raf = new RandomAccessFile(file, "r");
            long index = raf.length();
            byte[] buffer = new byte[V1_MAGIC.length];
            index -= V1_MAGIC.length;
            raf.seek(index);
            raf.readFully(buffer);
            // whether magic bytes matched
            if (isV1MagicMatch(buffer)) {
                index -= SHORT_LENGTH;
                raf.seek(index);
                // read channel length field
                int length = readShort(raf);
                if (length > 0) {
                    index -= length;
                    raf.seek(index);
                    // read channel bytes
                    byte[] bytesComment = new byte[length];
                    raf.readFully(bytesComment);
                    return new String(bytesComment, CONTENT_CHARSET);
                } else {
                    throw new Exception("zip channel info not found");
                }
            } else {
                throw new Exception("zip v1 magic not found");
            }
        } finally {
            if (raf != null) {
                raf.close();
            }
        }
    }

    private static short readShort(DataInput input) throws IOException {
        byte[] buf = new byte[SHORT_LENGTH];
        input.readFully(buf);
        ByteBuffer bb = ByteBuffer.wrap(buf).order(ByteOrder.LITTLE_ENDIAN);
        return bb.getShort(0);
    }

    /**
     * check v1 magic
     *
     * @param buffer
     * @return
     */
    private static boolean isV1MagicMatch(byte[] buffer) {
        if (buffer.length != V1_MAGIC.length) {
            return false;
        }
        for (int i = 0; i < V1_MAGIC.length; ++i) {
            if (buffer[i] != V1_MAGIC[i]) {
                return false;
            }
        }
        return true;
    }
}
```

这样就完成了读取渠道信息

##### 具体生成渠道信息包的后apk尾部信息

大家可以下载个16进制查看器，查看包尾信息
> 这里提供个分享链接给大家下载，windows系统的 exe 文件。[https://pan.baidu.com/s/1RzV5F-wtzl9SqO4AjAoUFg](https://pan.baidu.com/s/1RzV5F-wtzl9SqO4AjAoUFg)

![enter image description here](http://p9iqqot9p.bkt.clouddn.com/%E6%B8%A0%E9%81%93%E5%8C%85%E5%B0%BE%E4%BF%A1%E6%81%AF.png)

最后4个字节是我们自己定义的魔数。 

### 这种打包方式的一些缺点。

首先这是基于 V1 打包方式的，并不支持 V2.
使用这种方式打包，在 apk 安装耗时相比优化过更严格的 V2 打包方式 2倍以上

但我们app 非常小， 所以这些缺点并不明显。所以我们采用了这种相对简单，快速的打包方式。

其实我们需要做的是，根据自己实际项目需求，找到合适的渠道打包方式，这是我要给大家的思考。

### 最后

以上就是我们项目实际运用的生成渠道信息的方式。

我发现前人早已造好了轮子了， 并且支持 V1 和 V2 打包方式，有兴趣可以去看下,有3000+ star,  [https://github.com/mcxiaoke/packer-ng-plugin](https://github.com/mcxiaoke/packer-ng-plugin)

### 关于作者

如果文章对你有帮助，大家可以赞赏支持鼓励下作者。
有什么疑问、错误都可以关注下方我的公众号留言。
这是我的个人博客 [http://crizquan.top](http://crizquan.top/) 
这是我的 GitHub 地址 [https://github.com/luomoBM](https://github.com/luomoBM)
![我的个人公众号](http://p9iqqot9p.bkt.clouddn.com/wechatoa.jpg)
