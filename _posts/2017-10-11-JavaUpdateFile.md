---
layout: post
title:  "Java 使用POST请求方式上传文件"
date:   2017-10-11 18:25:01 +0800
tag: Java
---


```java
	// 换行符
	final String newLine = "\r\n";
	final String boundaryPrefix = "--";
	// 定义数据分隔线
	String BOUNDARY = "----WebKitFormBoundary7MA4YWxkTrZu0gW";
	// 服务器的域名
	URL url = new URL(uri);
	HttpURLConnection conn = (HttpURLConnection) url.openConnection();
	// 设置为POST情
	conn.setRequestMethod("POST");
	// 发送POST请求必须设置如下两行
	conn.setDoOutput(true);
	conn.setDoInput(true);
	conn.setUseCaches(false);
	// 设置请求头参数
	conn.setRequestProperty("connection", "Keep-Alive");
	conn.setRequestProperty("Charsert", "UTF-8");
	conn.setRequestProperty("Content-Type", "multipart/form-data; boundary=" + BOUNDARY);

	OutputStream out = new DataOutputStream(conn.getOutputStream());

	// 上传文件
	File file = new File(geoFilePath);
	String fName = file.getName();
	StringBuilder sb = new StringBuilder();
	sb.append(boundaryPrefix);
	sb.append(BOUNDARY);
	sb.append(newLine);
	// 文件参数,photo参数名可以随意修改
	sb.append("Content-Disposition: form-data;name=\"file\";filename=\"" + fName + "\"" + newLine);
	sb.append("Content-Type: multipart/form-data");
	// 参数头设置完以后需要两个换行，然后才是参数内容
	sb.append(newLine);
	sb.append(newLine);

	// 将参数头的数据写入到输出流中
	out.write(sb.toString().getBytes());

	// 数据输入流,用于读取文件数据
	DataInputStream in = new DataInputStream(new FileInputStream(file));
	byte[] bufferOut = new byte[1024];
	int bytes = 0;
	// 每次读1KB数据,并且将文件数据写入到输出流中
	while ((bytes = in.read(bufferOut)) != -1) {
		out.write(bufferOut, 0, bytes);
	}
	// 最后添加换行
	out.write(newLine.getBytes());
	in.close();

	// 定义最后数据分隔线，即--加上BOUNDARY再加上--。
	byte[] end_data = (newLine + boundaryPrefix + BOUNDARY + boundaryPrefix + newLine).getBytes();
	// 写上结尾标识
	out.write(end_data);
	out.flush();
	out.close();
	// 定义BufferedReader输入流来读取URL的响应
	BufferedReader reader = new BufferedReader(new InputStreamReader(conn.getInputStream()));
	String line = null;
	while ((line = reader.readLine()) != null) {
		System.out.println(line);
	}
```