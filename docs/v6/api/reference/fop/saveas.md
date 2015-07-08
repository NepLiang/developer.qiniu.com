---
layout: docs
title: 处理结果另存
order: 185
---

<a id="saveas"></a>
# 处理结果另存

- [描述](#saveas-description)
- [接口规格](#saveas-specification)
- [签名算法](#saveas-sign-algorithm)
  - [算法描述](#describe-algorithm)
  - [算法实例](#case-algorithm)
- [附注](#saveas-remarks)
- [示例](#saveas-samples)
- [内部参考资源](#saveas-internal-resources)

<a id="saveas-description"></a>
## 描述

七牛云存储的云处理API接口均支持如下串行处理规格:

```
<DownloadUrl>?<fop1>|<fop2>|<fop3>|<fopN>
```

可以看到每一步的输出都是下一步的输入，而最后一步的输出即为最终下载到的资源内容。  
每调用一次这类串行化接口，就必须实时地按序执行所有接口，有可能引起不必要的计算耗时。  

我们提供名为`saveas`的云处理操作，将云处理结果作为资源保存到指定空间内，并赋以指定Key。保存成功后，下一次可直接通过指定Key来访问该资源，以达到提升下载速度的效果。  

<a id="saveas-specification"></a>
## 接口规格

```
saveas/<EncodedEntryURI>/sign/<Sign>
```

参数名称             | 必填  | 类型   | 说明                                                          
:------------------  | :---- | :----- | :---------------------------------------------------------------
`<EncodedEntryURI>`  | 是    | string | 以[EncodedEntryURI格式][encodedEntryURIHref]组织的目标Bucket与Key
`/sign/<Sign>`       | 是    | string | 请求签名部分，算法见下方 `在pfop操作中不用指定该参数`

<a id="saveas-sign-algorithm"></a>
## 签名算法

<a id="describe-algorithm"></a>
### 算法描述

1. 在下载URL（不含Scheme部分，即去除`http://`）后附加`saveas`接口（不含签名部分）：  

	```
    NewURL = URL + "|saveas/<EncodedEntryURI>"
	```

2. 使用`SecretKey`对新的下载URL进行HMAC1-SHA1签名：  

	```
    Sign = hmac_sha1(SecretKey, NewURL)
	```

3. 对签名进行URL安全的Base64编码：

	```
    EncodedSign = urlsafe_base64_encode(Sign)
	```

4. 在新的下载URL后拼接签名参数：

	```
    FinalURL = NewURL + "/sign/<AccessKey>:<EncodedSign>"
	```
<a id="case-algorithm"></a>
### 算法实例

生成saveas请求的完整Go代码如下：  

```{go}
func makeSaveasUrl(URL, accessKey string, secretKey []byte, saveBucket, saveKey string) string {

	encodedEntryURI := base64.URLEncoding.EncodeToString([]byte(saveBucket+":"+saveKey))

	URL += "|saveas/" + encodedEntryURI

	h := hmac.New(sha1.New, secretKey)

	// 签名内容不包括Scheme，仅含如下部分：
    // <Domain>/<Path>?<Query>

	u, _ := url.Parse(URL)
	io.WriteString(h, u.Host + u.RequestURI())

	d := h.Sum(nil)
	sign := accessKey + ":" + base64.URLEncoding.EncodeToString(d)

	return URL + "/sign/" + sign

}
```

<a id="saveas-remarks"></a>
## 附注

- `urlsafe_base64_encode()` 函数按照标准的 [RFC 4648](http://www.ietf.org/rfc/rfc4648.txt) 实现，开发者可以参考 [github.com/qiniu](https://github.com/qiniu) 上各SDK的样例代码。
- 此处签名内容不包含Scheme部分，与DownloadToken签名不一样。
- 当要持久化保存的fop耗时较长时候，saveas请求会返回CDN超时，但是只要保证发送的saveas请求合法，七牛服务器还是会对请求做正确处理。
- 大多数使用场景下，建议使用[触发持久化处理][pfopHref]接口来实现处理结果持久存储，避免使用同步操作的`saveas`接口，提升访问速度。
- c# http库发送请求的时候会把 | 转成 %7c，由于签名签的是 | ，导致报400错误，可以在发送请求的时候url中把 | 改为 %7c 或者签名的时候签 %7c 两种方法解决

<a id="saveas-samples"></a>
## 示例

1. 原资源是一个图片：  

	```
    http://t-test.qiniudn.com/Ship.jpg
	```

2. 将图片做缩略处理：  

	```
    http://t-test.qiniudn.com/Ship.jpg?imageView/2/w/200/h/200
	```

3. 对上述云处理结果进行持久化保存：  

	```
    另存操作的目标空间与资源名
    entryURI        = "t-test:Ship-thumb-200.jpg"

    编码结果
    encodedEntryURI = "dC10ZXN0OlNoaXAtdGh1bWItMjAwLmpwZw=="

    需要签名的部分
    signingStr = "t-test.qiniudn.com/Ship.jpg?imageView/2/w/200/h/200|saveas/dC10ZXN0OlNoaXAtdGh1bWItMjAwLmpwZw=="

    签名结果
    sign       = "iguImegxd6hbwF8J6ij2dlLIgycyU4thjg-xmu9q:38kMkgw3We96NWSgUHJz9C72noQ="
	```

4. 最终得到的完整下载URL：  

	```
    http://t-test.qiniudn.com/Ship.jpg?imageView/2/w/200/h/200|saveas/dC10ZXN0OlNoaXAtdGh1bWItMjAwLmpwZw==/sign/iguImegxd6hbwF8J6ij2dlLIgycyU4thjg-xmu9q:38kMkgw3We96NWSgUHJz9C72noQ=
	```

5. 保存转码后资源可通过如下URL访问：  

	```
    http://t-test.qiniudn.com/Ship-thumb-200.jpg
	```

<a id="saveas-internal-resources"></a>
## 内部参考资源

- [EncodedEntryURI格式][encodedEntryURIHref]
- [触发持久化处理][pfopHref]

[encodedEntryURIHref]:          http://developer.qiniu.com/docs/v6/api/reference/data-formats.html#data-format-encoded-entry-uri "EncodedEntryURI格式"
[pfopHref]:                     http://developer.qiniu.com/docs/v6/api/reference/fop/pfop/pfop.html                                     "触发持久化处理"
