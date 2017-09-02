---
layout: post
title:  "使用AWS Lambda优化图片格式"
date:   2017-09-02 16:35:01 +0800
categories: aws lambda
---

# 当用户上传一个图片文件到S3上面的时候，触发Lambda函数，生成一个thumbnail小图片。

参考链接：<http://docs.aws.amazon.com/lambda/latest/dg/with-s3-example-deployment-pkg.html>  
github链接：<https://github.com/wfhu/aws-Lambda-S3-Thumbnail>


主要的技术点：
* PUT事件触发Lambda函数执行
* 使用 [boto3](https://pypi.python.org/pypi/boto3) package来进行S3相关操作
* 使用 [PIL](http://www.pythonware.com/products/pil/) package来进行图片的处理
* Lambda运行环境支持/tmp目录

常见使用场景：
* 图片自动打水印
* 图片压缩
* 原图上传后，为各种使用场景自动处理生成不同大小的图片


附上修改过的示例代码：

```python
from __future__ import print_function
import boto3
import os
import sys
import uuid
from PIL import Image
import PIL.Image
     
s3_client = boto3.client('s3')
     
def resize_image(image_path, resized_path):
    with Image.open(image_path) as image:
        image.thumbnail(tuple(x / 2 for x in image.size))
        image.save(resized_path)
     
def handler(event, context):
    for record in event['Records']:
        bucket = record['s3']['bucket']['name']
	print("Log bucket name:", bucket)
        key = record['s3']['object']['key'] 
	print("Log key name:", key)
        download_path = '/tmp/{}{}'.format(uuid.uuid4(), key)
	print("Log download_path name:", download_path)
        upload_path = '/tmp/resized-{}'.format(key)
	print("Log upload_path name:", upload_path)
        
	# create source directory
	directory = os.path.dirname(download_path)
	print("Log directory name :", directory)

	if not os.path.exists(directory):
		os.makedirs(directory)

	#create target directory
	targetdirectory = os.path.dirname(upload_path)
	print("Log targetdirectory name :", targetdirectory)

	if not os.path.exists(targetdirectory):
		os.makedirs(targetdirectory)

	with open(download_path, 'wb') as data:
	        s3_client.download_fileobj(bucket, key, data)
        resize_image(download_path, upload_path)
        s3_client.upload_file(upload_path, '{}resized'.format(bucket), key)
```




