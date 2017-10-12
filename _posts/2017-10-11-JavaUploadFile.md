---
layout: post
title:  "Java 使用POST请求方式上传文件"
date:   2017-10-11 18:25:01 +0800
tag: Java
---


Java以form表单的形式对指定的api发送POST请求上传文件。实现代码如下：
>pom文件配置：

```xml
<dependency>
	<groupId>org.apache.httpcomponents</groupId>
	<artifactId>httpclient</artifactId>
	<version>4.5.1</version>
</dependency>
<dependency>
    <groupId>org.apache.httpcomponents</groupId>
    <artifactId>httpmime</artifactId>
    <version>4.5.2</version>
</dependency>
```
>实现：

```java
public void sendFile(String fpath, String postUrl) {
	try {
		// 定义请求url
		HttpClient httpClient = HttpClientBuilder.create().build();
		HttpPost httpPost = new HttpPost(postUrl);
		File f = new File(fpath);
		FileBody fileBody = new FileBody(f);
		StringBody stringBody = new StringBody("application/text", Charset.defaultCharset());

		// 以浏览器兼容模式访问,否则就算指定编码格式,中文文件名上传也会乱码
		HttpEntity reqEntity = MultipartEntityBuilder.create().setMode(HttpMultipartMode.BROWSER_COMPATIBLE)
				.addPart("file", fileBody).setCharset(CharsetUtils.get("UTF-8")).build();

		httpPost.setEntity(reqEntity);
		HttpResponse response = httpClient.execute(httpPost);
		System.out.println(response.getStatusLine().getStatusCode());
		if (HttpStatus.SC_OK == response.getStatusLine().getStatusCode()) {

			HttpEntity entitys = response.getEntity();
			if (entitys != null) {
				System.out.println(EntityUtils.toString(entitys));
			}
		}
		httpClient.getConnectionManager().shutdown();
	} catch (Exception e) {
		System.out.println(e.toString());
	}
}
```
