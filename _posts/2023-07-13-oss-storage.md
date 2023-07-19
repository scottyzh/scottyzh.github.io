---
layout: post
title: 资源迁移OSS方案记录
categories: storage
description: SpringBoot+OSS方案
keywords: OSS  
---

第一次实习，接受的任务就是将公司的视频资源迁移到OSS服务器上，这里记录一下一些迁移过程。



# 搭建流程

在阿里云上购买oss，并获取具有该Bucket访问权限的AccessKey ID和AccessKey Secret信息。

## 数据迁移方案一

第一次尝试，不修改后端代码，直接将oss直接挂载映射到本地服务器。发现这种方法并不起作用，由于oss最终传输资源都要经由服务器再下发给用户，传输速度依然受限于服务器带宽，并没有起到加速的作用。

操作流程如下：

（1）  下载安装包。

以下载CentOS 7.0 (x64)版本为例：

sudo wget http://gosspublic.alicdn.com/ossfs/ossfs_1.80.6_centos7.0_x86_64.rpm

（2）  安装ossfs。

（3）  CentOS系统/Anolis系统

以CentOS 7.0(x64)版本为例，安装命令如下：

sudo yum install ossfs_1.80.6_centos7.0_x86_64.rpm

（4）  配置账号访问信息。

将Bucket名称以及具有该Bucket访问权限的AccessKey ID和AccessKey Secret信息存放在/etc/passwd-ossfs文件中。文件的权限建议设置为640。

sudo **echo** BucketName:yourAccessKeyId:yourAccessKeySecret > /etc/passwd-ossfs

sudo **chmod** 640 /etc/passwd-ossfs

BucketName、yourAccessKeyId、yourAccessKeySecret请按需替换为您实际的Bucket名称、AccessKey ID和AccessKey Secret，例如：

sudo **echo** bucket-test:LTAIbZcdmQ****:MOk8x0y1coh7A5e2MZEUz**** > /etc/passwd-ossfs

sudo **chmod** 640 /etc/passwd-ossfs

（5）将Bucket挂载到指定目录。

由于我们需要将视频文件进行挂载，此时的挂载目录是/data/docker/videoSystem_v，且BucketName是video-bucket01。

示例如下：

**mkdir** /tmp/ossfs

ossfs video-bucket01 /data/docker/videoSystem_v/videoRes -o url=http://oss-cn-hangzhou.aliyuncs.com

 

（6）由于系统是跑在docker里面，数据是存储到容器里，因此需要把容器里的数据与oss映射的路径做绑定，这里通过创建docker数据卷做绑定：

 docker volume create --driver local \

  --opt type=none \

  --opt device=/data/docker/videoSystem_v/videoRes \

  --opt o=bind \

  oss_volume_name

 

（7）启动docker对应镜像，并作为数据卷的绑定：

docker run -p 8181:8181 -v oss_volume_name:/videoSystem_workdir/videoRes --name videoSystem --restart unless-stopped -d videosystem:v4.0

 

（8）Bucket卸载指令

sudo fusermount -u /data/docker/videoSystem_v/videoRes



​    测试结果：60mb的视频上传，需要花费6分钟才能上传结束。如果不采用OSS映射方案，一分钟就能结束，上传反而更花费时间，这里推测是因为上传需要先上传到服务器本地内存，然后再从服务器本地上传到OSS服务器，并且中间存在映射关系，服务器自身有其他的业务压力，使得耗费时间更长。并且，上传的1080p视频，从用户网站打开播放，不能流畅播放，跑的依然是原先的服务器带宽，如果直接使用OSS上的视频url链接，绕过服务器，则能直接流畅播放1080p的视频。

​    总结：该方案并不能实现预期的效果，因此下一方案将从后端代码层面进行修改。



## 数据迁移方案二

方案二从代码端进行更改，并且更改上传的模式。

后端代码修改方案：

1、上传方式由后端上传改到前端，该方案进行sts认证，需要在OSS服务器端开启，并且要开启跨域访问，新增一些自定义权限认证，后端只负责授权即可，代码：

```java
   /********************************
     *  @function  : 前端获取oss的sts认证
     *  @parameter :
     *  @return    : AjaxResult
     *  @date      : 2023.06.25 15:37
     ********************************/
    @ResponseBody
    @GetMapping("/getSts")
    public AjaxResult getSts(HttpServletRequest req) throws com.aliyuncs.exceptions.ClientException {
        AssumeRoleResponse.Credentials credentials = null;
        String endpoint = "sts.cn-hangzhou.aliyuncs.com";
        String accessKeyId = "";
        String accessKeySecret = "";
        String roleArn = "";
        String roleSessionName = "AliyunDMSRol--ePolicy";
        String bucket = "video-bucket01";
        String region = "oss-cn-hangzhou";
        String policy = "{\n" +
                "    \"Statement\": [\n" +
                "        {\n" +
                "            \"Action\": [\n" +
                "                \"oss:GetObject\",\n" +
                "                \"oss:PutObject\",\n" +
                "                \"oss:DeleteObject\",\n" +
                "                \"oss:ListParts\",\n" +
                "                \"oss:AbortMultipartUpload\",\n" +
                "                \"oss:ListObjects\"\n" +
                "            ],\n" +
                "            \"Effect\": \"Allow\",\n" +
                "            \"Resource\": [\n" +
                "                \"acs:oss:*:*:video-bucket01/*\",\n" +
                "                \"acs:oss:*:*:video-bucket01\"\n" +
                "            ]\n" +
                "        }\n" +
                "    ],\n" +
                "    \"Version\": \"1\"\n" +
                "}";
        // 设置临时访问凭证的有效时间为3600秒。
        Long durationSeconds = 3600L;
        Map<String, Object> map = new HashMap<>();
        try {
            // 添加endpoint（直接使用STS endpoint，前两个参数留空，无需添加region ID）
            DefaultProfile.addEndpoint("", "", "Sts", endpoint);
            // 构造default profile（参数留空，无需添加region ID）
            IClientProfile profile = DefaultProfile.getProfile("", accessKeyId, accessKeySecret);
            // 用profile构造client
            DefaultAcsClient client = new DefaultAcsClient(profile);
            final AssumeRoleRequest request = new AssumeRoleRequest();
            request.setMethod(MethodType.POST);
            request.setRoleArn(roleArn);
            request.setRoleSessionName(roleSessionName);
            request.setPolicy(policy); // Optional
            request.setProtocol(ProtocolType.HTTPS); // 必须使用HTTPS协议访问STS服务);
            final AssumeRoleResponse response = client.getAcsResponse(request);
            credentials = response.getCredentials();
            map = ObjectToMapUtil.objectToMap(credentials);
            map.put("bucket", bucket);
            map.put("region", region);
            CallBack callback = new CallBack();
           
            //回调地址
            callback.setUrl("http://y3bq3i.natappfree.cc/clt3D/videoUpload/callBack");
           //设置返回的回调参数 其中后面两个是自定义参数。
      callback.setBody("fileName=${object}&size=${size}&mimeType=${mimeType}&videoName=${x:videoName}&definition=${x:definition}");
            callback.setContentType("application/x-www-form-urlencoded");
//            String callbackData = BinaryUtil.toBase64String(JSONUtil.parse(callback).toString().getBytes("utf-8"));
            map.put("callback", callback);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return AjaxResult.success(map);
    }
```

2、上传成功后OSS调用回调接口，返回上传的文件的数据：

```java
 /********************************
     *  @function  : 回调接口，将数据写入数据库以及返回成功值给OSS服务器
     *  @parameter :
     *  @return    : HashMap<String, String>
     *  @date      : 2023.06.25 15:37
     ********************************/
    @ResponseBody
    @PostMapping("/callBack")
    public AjaxResult callBack(HttpServletRequest request) {
        String fileName = request.getParameter("fileName");
        String videoName = request.getParameter("videoName");
        String definition = request.getParameter("definition");
        String size = request.getParameter("size");
        String mimeType = request.getParameter("mimeType");
        String ossUrl = "https://video-bucket01.oss-cn-hangzhou.aliyuncs.com";
        if (fileName.equals("") || size.equals("")) {
            return AjaxResult.failed("上传失败");
        }
        String[] mimeTypeArray = mimeType.split("/");
        //获取文件类型 video 或者是 image
        String fileType = mimeTypeArray[mimeTypeArray.length - 1];
        //得到后缀
        String[] fileNameArray = fileName.split("/");
        String fileUUID = fileNameArray[2];
        String fileDir;
//        String parent3DDir = ResourceUtil.getStorageParent3DDir();
        Video3D newVideo3D = new Video3D().setMark(fileUUID)
                .setType(0)
                .setVideoName(videoName)
                .setUploadTime(new Date());
        Integer fileSize = Integer.parseInt(size) / 1024000;
        switch (definition) {
            case "360p":     // 判断属于哪种清晰度,决定保存的文件名
                fileDir = "video360p";
                newVideo3D.setVideo360pFormat(fileType);
                newVideo3D.setVideo360pSize(fileSize + "MB");
                newVideo3D.setVideo360pUrl(ossUrl + "/videoRes/share3D/" + fileUUID + "/" + fileName);
                break;
            case "480p":
                fileDir = "video480p";
                newVideo3D.setVideo480pFormat(fileType);
                newVideo3D.setVideo480pSize(fileSize + "MB");
                newVideo3D.setVideo480pUrl(ossUrl + "/videoRes/share3D/" + fileUUID + "/" + fileName);
                break;
            case "720p":
                fileDir = "video720p";
                newVideo3D.setVideo720pFormat(fileType);
                newVideo3D.setVideo720pSize(fileSize + "MB");
                newVideo3D.setVideo720pUrl(ossUrl + "/videoRes/share3D/" + fileUUID + "/" + fileName);
                break;
            case "1080p":
                fileDir = "video1080p";
                newVideo3D.setVideo1080pFormat(fileType);
                newVideo3D.setVideo1080pSize(fileSize + "MB");
                newVideo3D.setVideo1080pUrl(ossUrl + "/" + fileName);
                break;
            default:
                break;
        }

        //QR图也要处理
        // 从微信小程序服务器获取视频对应的QR码
        try {
            newVideo3D.setQrUrl(getQr(getUsableToken().getAccessToken(), fileUUID));
        } catch (Exception e) {
            e.printStackTrace();
        }

        //把上传的文件数据入库 判断是照片还是视频
        try {
            video3DService.save(newVideo3D);
        } catch (Exception e) {
            e.printStackTrace();
            return AjaxResult.failed("上传失败");
        }

        return AjaxResult.success("上传成功");
    }
```

3、删除OSS的资源接口修改：

```java
try {
  
    String prefix = "videoRes/share3D/" + mark + "/";
    // 列举所有包含指定前缀的文件并删除。
    String nextMarker = null;
    ObjectListing objectListing = null;
    String bucketName = "video-bucket01";
    do {
        ListObjectsRequest listObjectsRequest = new ListObjectsRequest(bucketName)
                .withPrefix(prefix)
                .withMarker(nextMarker);
        objectListing = ossClient.listObjects(listObjectsRequest);
        if (objectListing.getObjectSummaries().size() > 0) {
            List<String> keys = new ArrayList<String>();
            for (OSSObjectSummary s : objectListing.getObjectSummaries()) {
                System.out.println("key name: " + s.getKey());
                keys.add(s.getKey());
            }
            DeleteObjectsRequest deleteObjectsRequest = new DeleteObjectsRequest(bucketName).withKeys(keys).withEncodingType("url");
            DeleteObjectsResult deleteObjectsResult = ossClient.deleteObjects(deleteObjectsRequest);
            List<String> deletedObjects = deleteObjectsResult.getDeletedObjects();
            for (String obj : deletedObjects) {
                    String deleteObj = URLDecoder.decode(obj, "UTF-8");
                    System.out.println(deleteObj);
            }
        }
        nextMarker = objectListing.getNextMarker();
    } while (objectListing.isTruncated());
} catch (Exception e) {
    e.printStackTrace();
    return AjaxResult.failed();
}
```



