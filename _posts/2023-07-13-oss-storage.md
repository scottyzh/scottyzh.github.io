---
layout: post
title: 资源迁移OSS方案记录
categories: storage
description: SpringBoot+OSS方案
keywords: OSS  
---

视频资源迁移到OSS服务器上，记录一下迁移过程。



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

```
sudo yum install ossfs_1.80.6_centos7.0_x86_64.rpm
```

（4）  配置账号访问信息。

将Bucket名称以及具有该Bucket访问权限的AccessKey ID和AccessKey Secret信息存放在/etc/passwd-ossfs文件中。文件的权限建议设置为640。

```
sudo **echo** BucketName:yourAccessKeyId:yourAccessKeySecret > /etc/passwd-ossfs

sudo **chmod** 640 /etc/passwd-ossfs

BucketName、yourAccessKeyId、yourAccessKeySecret请按需替换为您实际的Bucket名称、AccessKey ID和AccessKey Secret，例如：

sudo **echo** bucket-test:LTAIbZcdmQ****:MOk8x0y1coh7A5e2MZEUz**** > /etc/passwd-ossfs

sudo **chmod** 640 /etc/passwd-ossfs
```

（5）将Bucket挂载到指定目录。

由于我们需要将视频文件进行挂载，此时的挂载目录是/data/docker/videoSystem_v，且BucketName是video-bucket01。

示例如下：

```
**mkdir** /tmp/ossfs

ossfs video-bucket01 /data/docker/videoSystem_v/videoRes -o url=http://oss-cn-hangzhou.aliyuncs.com
```

 

（6）由于系统是跑在docker里面，数据是存储到容器里，因此需要把容器里的数据与oss映射的路径做绑定，这里通过创建docker数据卷做绑定：

```
 docker volume create --driver local \

  --opt type=none \

  --opt device=/data/docker/videoSystem_v/videoRes \

  --opt o=bind \

  oss_volume_name
```

 

（7）启动docker对应镜像，并作为数据卷的绑定：

```
docker run -p 8181:8181 -v oss_volume_name:/videoSystem_workdir/videoRes --name videoSystem --restart unless-stopped -d videosystem:v4.0
```

 

（8）Bucket卸载指令

```
sudo fusermount -u /data/docker/videoSystem_v/videoRes
```



​    测试结果：60mb的视频上传，需要花费6分钟才能上传结束。如果不采用OSS映射方案，一分钟就能结束，上传反而更花费时间，这里推测是因为上传需要先上传到服务器本地内存，然后再从服务器本地上传到OSS服务器，并且中间存在映射关系，服务器自身有其他的业务压力，使得耗费时间更长。并且，上传的1080p视频，从用户网站打开播放，不能流畅播放，跑的依然是原先的服务器带宽，如果直接使用OSS上的视频url链接，绕过服务器，则能直接流畅播放1080p的视频。

​    总结：该方案并不能实现预期的效果，因此下一方案将从后端代码层面进行修改。



## 数据迁移方案二

方案二从代码端进行更改，并且更改上传的模式——由前端进行上传。

### 配置OSS

1、本套方案采用前端上传文件的方式，需要前端向请求sts权限认证，为实现这套流程，需要在OSS控制台申请RAM用户，并且配置相关的请求sts认证权限。具体流程参考：

[使用STS临时访问凭证访问OSS (aliyun.com)](https://help.aliyun.com/document_detail/100624.html)

2、保存RAM用户的访问密钥（AccessKey ID 和 AccessKey Secret）。

3、将bucket的权限设置为公共读，其文件的url是永久有效的，方便我们存到数据库以及后续通过url进行文件的下载和播放。



### 后端代码修改

新增AliyunConfig类：

```java
@Component
@ConfigurationProperties(prefix = "aliyun")
@Data
public class AliyunConfig {

    // aliyun endpoint
    private String endpoint;

    // aliyun accessKeyId
    private String accessKeyId;

    // aliyun accessKeySecret
    private String accessKeySecret;

    // aliyun bucketName
    private String bucketName;

    // aliyun urlPrefix
    private String urlPrefix;

    // aliyun stsEndPoint
    private String stsEndPoint;

    // 用户权限角色
    private String roleArn;

    // OSS所在区域
    private String region;

    // OSS回调地址
    private String callBackUrl;

    // 3D存储路径
    private String StorageDirXA;

    // XR存储路径
    private String StorageDirXB;

    @Bean
    public OSS oSSClient() {
        return new OSSClient(endpoint, accessKeyId, accessKeySecret);
    }
}
```

在applicantion.yml新增配置信息：

```
aliyun:
  endpoint: oss-cn-shenzhen.aliyuncs.com
  accessKeyId: LTAI5t**********B9vBbD
  accessKeySecret: 0FI3UB*****3KxnZdKNpHKWcY
  bucketName: ******
  urlPrefix: https:********.com
  stsEndPoint: sts.cn-shenzhen.aliyuncs.com
  roleArn: acs:ram::10*****2863:role/ramosstest
  region: oss-cn-shenzhen
  callBackUrl: https://**********
  StorageDir3d: videoRes/shareXA
  StorageDirXr: videoRes/shareXB
```

回调上传

```java
@ResponseBody
@GetMapping("/getSts")
public AjaxResult getSts(HttpServletRequest req) throws com.aliyuncs.exceptions.ClientException {
    AssumeRoleResponse.Credentials credentials = null;
    String roleSessionName = "AliyunDMSRol--ePolicy";
    // OSS服务器上设置的角色拥有所有权限，可以在下面policy添加需要的权限，按需要给予相关权限
    String policy = "{\n" +
            "    \"Statement\": [\n" +
            "        {\n" +
            "            \"Action\": [\n" +
            "                \"oss:GetObject\",\n" +
            "                \"oss:PutObject\",\n" +
            // 前端暂时不需要删除权限
            //"                \"oss:DeleteObject\",\n" +
            "                \"oss:ListParts\",\n" +
            "                \"oss:AbortMultipartUpload\",\n" +
            "                \"oss:ListObjects\"\n" +
            // 下面的oss:* 拥有所有权限 慎用
            //"                \"oss:*\"\n" +
            "            ],\n" +
            "            \"Effect\": \"Allow\",\n" +
            "            \"Resource\": [\n" +
            "                \"acs:oss:*:*:colorlight-video/*\",\n" +
            "                \"acs:oss:*:*:colorlight-video\"\n" +
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
        DefaultProfile.addEndpoint("", "", "Sts", aliyunConfig.getStsEndPoint());
        // 构造default profile（参数留空，无需添加region ID）
        IClientProfile profile = DefaultProfile.getProfile("", aliyunConfig.getAccessKeyId(), aliyunConfig.getAccessKeySecret());
        // 用profile构造client
        DefaultAcsClient client = new DefaultAcsClient(profile);
        final AssumeRoleRequest request = new AssumeRoleRequest();
        request.setMethod(MethodType.POST);
        request.setRoleArn(aliyunConfig.getRoleArn());
        request.setRoleSessionName(roleSessionName);
        request.setPolicy(policy); // Optional
        request.setProtocol(ProtocolType.HTTPS); // 必须使用HTTPS协议访问STS服务);
        final AssumeRoleResponse response = client.getAcsResponse(request);
        credentials = response.getCredentials();
        map = ObjectToMapUtil.objectToMap(credentials);
        map.put("bucket", aliyunConfig.getBucketName());
        map.put("region",aliyunConfig.getRegion());
        CallBack callback = new CallBack();
        callback.setUrl(aliyunConfig.getCallBackUrl() + "/clt3D/videoUpload/callBack");
        callback.setBody("{fileName:${object},size:${size},mimeType:${mimeType},videoName:${x:videoName},definition:${x:definition}}");
        callback.setContentType("application/json");
        map.put("callback", callback);
    } catch (Exception e) {
        e.printStackTrace();
    }
    return AjaxResult.success(map);
}

```

上传成功后OSS调用回调接口，返回上传的文件的数据：

```java
 @ResponseBody
@PostMapping("/callBack")
@Log(title = "新增视频", module = "clt3D", businessType = BusinessTypeEnum.INSERT)
public AjaxResult callBack(HttpServletRequest request, HttpServletResponse response) {
    Video3D newVideo3D;
    String fileUUID;
    try {
        AliyunCallbackUtil aliyunCallbackUtil = new AliyunCallbackUtil();
        String ossCallBackBody = aliyunCallbackUtil.GetPostBody(request.getInputStream(), Integer.parseInt(request.getHeader("content-length")));
        JSONObject jsonBody = JSON.parseObject(ossCallBackBody);
        String fileName = jsonBody.getString("fileName");
        String videoOriginalFilename = jsonBody.getString("videoName");
        String videoName = NameUtil.format3DVideoName(ResourceUtil.getFileNamePrefix(videoOriginalFilename));
        String definition = jsonBody.getString("definition");
        String size = jsonBody.getString("size");
        String ossUrl = aliyunConfig.getUrlPrefix();
        String fileDir;
        boolean zipFlag = false;
        boolean verifyResult = aliyunCallbackUtil.VerifyOSSCallbackRequest(request, ossCallBackBody);
        // 回调签名验证是不是OSS发来的签名
        if (!verifyResult) {
            response.setStatus(HttpServletResponse.SC_BAD_REQUEST);
            return AjaxResult.failed("OSS回调签名验证失败");
        }
        String[] videoNameArray = videoOriginalFilename.split("\\.");
        //获取文件类型 video 或者是 image
        String fileType = videoNameArray[videoNameArray.length - 1];
        //拿到文件名
        String[] fileNameArray = fileName.split("/");
        String realName = fileNameArray[3];
        String[] realNameArray = realName.split("_");
        fileUUID = realNameArray[0];
        // 判断上传的是不是压缩视频
        if (realNameArray[1].equals("zip")) {
            Video3D video3D = video3DService.getOne(new QueryWrapper<Video3D>().eq("mark", fileUUID));
            video3D.setVideoCompressionUrl(ossUrl + "/" + fileName);
            boolean updateFlag = video3DService.updateById(video3D);
            if (!updateFlag) {
                return AjaxResult.failed("压缩文件上传失败");
            } else {
                return AjaxResult.success("上传成功");
            }
        }else {
            newVideo3D = new Video3D().setMark(fileUUID)
                    .setType(0)
                    .setVideoName(videoName)
                    .setUploadTime(new Date());
            String fileSize = FileUtil.getFileSizeDescription(Integer.parseInt(size));
            switch (definition) {
                case "360p":     // 判断属于哪种清晰度,决定保存的文件名
                    fileDir = "video360p";
                    newVideo3D.setVideo360pFormat(fileType);
                    newVideo3D.setVideo360pSize(fileSize);
                    newVideo3D.setVideo360pUrl(ossUrl + "/" + fileName);
                    break;
                case "480p":
                    fileDir = "video480p";
                    newVideo3D.setVideo480pFormat(fileType);
                    newVideo3D.setVideo480pSize(fileSize);
                    newVideo3D.setVideo480pUrl(ossUrl + "/" + fileName);
                    break;
                case "720p":
                    fileDir = "video720p";
                    newVideo3D.setVideo720pFormat(fileType);
                    newVideo3D.setVideo720pSize(fileSize);
                    newVideo3D.setVideo720pUrl(ossUrl + "/" + fileName);
                    break;
                case "1080p":
                    fileDir = "video1080p";
                    newVideo3D.setVideo1080pFormat(fileType);
                    newVideo3D.setVideo1080pSize(fileSize);
                    newVideo3D.setVideo1080pUrl(ossUrl + "/" + fileName);
                    break;
                default:
                    break;
            }
        }
    } catch (Exception e) {
        e.printStackTrace();
        return AjaxResult.failed("上传失败");
    }

    // 从微信小程序服务器获取视频对应的QR码
    try {
        newVideo3D.setQrUrl(getQr(getUsableToken().getAccessToken(), fileUUID));
    } catch (Exception e) {
        e.printStackTrace();
    }

    //把上传的文件数据入库
    try {
        video3DService.save(newVideo3D);
    } catch (Exception e) {
        e.printStackTrace();
        return AjaxResult.failed("上传失败");
    }
    return AjaxResult.success("上传成功");
}

```

回调中用到的AliyunCallbackUtil ，使用到了OSS调用回调接口传来的公钥和签名，采用RSA加密方式进行认证

```java
public class AliyunCallbackUtil extends HttpServlet {

   private static final long serialVersionUID = 5522372203700422672L;

   /********************************
    *  @method    : executeGet
    *  @function  : 生成公钥
    *  @parameter : [url]
    *  @return    : java.lang.String
    *  @date      : 2023/7/21 14:24
    ********************************/
   public String executeGet(String url) {
      BufferedReader in = null;
      String content = null;
      try {
         // 定义HttpClient
         @SuppressWarnings("resource")
            DefaultHttpClient client = new DefaultHttpClient();
         // 实例化HTTP方法
         HttpGet request = new HttpGet();
         request.setURI(new URI(url));
         HttpResponse response = client.execute(request);
         in = new BufferedReader(new InputStreamReader(response.getEntity().getContent()));
         StringBuffer sb = new StringBuffer("");
         String line = "";
         String NL = System.getProperty("line.separator");
         while ((line = in.readLine()) != null) {
            sb.append(line + NL);
         }
         in.close();
         content = sb.toString();
      } catch (Exception e) {
      } finally {
         if (in != null) {
            try {
               in.close();// 最后要关闭BufferedReader
            } catch (Exception e) {
               e.printStackTrace();
            }
         }
         return content;
      }
   }

   /********************************
    *  @method    : GetPostBody
    *  @function  : 获取回调Body内容
    *  @parameter : [is, contentLen]
    *  @return    : java.lang.String
    *  @date      : 2023/7/21 14:24
    ********************************/
   public String GetPostBody(InputStream is, int contentLen) {
      if (contentLen > 0) {
         int readLen = 0;
         int readLengthThisTime = 0;
         byte[] message = new byte[contentLen];
         try {
            while (readLen != contentLen) {
               readLengthThisTime = is.read(message, readLen, contentLen - readLen);
               if (readLengthThisTime == -1) {// Should not happen.
                  break;
               }
               readLen += readLengthThisTime;
            }
            return new String(message);
         } catch (IOException e) {
         }
      }
      return "";
   }


   /********************************
    *  @method    : VerifyOSSCallbackRequest
    *  @function  : 验证回调请求
    *  @parameter : [request, ossCallbackBody]
    *  @return    : boolean
    *  @date      : 2023/7/21 14:24
    ********************************/
   public boolean VerifyOSSCallbackRequest(HttpServletRequest request, String ossCallbackBody) throws NumberFormatException, IOException
   {
      // 整个验证流程是获取autorization 然后生成公钥，用公钥验签 使用请求url+body 与原先autorization作匹配 相同则验签就成功
      boolean ret = false;   
      String autorizationInput = new String(request.getHeader("Authorization"));
      String pubKeyInput = request.getHeader("x-oss-pub-key-url");
      byte[] authorization = BinaryUtil.fromBase64String(autorizationInput);
      byte[] pubKey = BinaryUtil.fromBase64String(pubKeyInput);
      String pubKeyAddr = new String(pubKey);
      if (!pubKeyAddr.startsWith("http://gosspublic.alicdn.com/") && !pubKeyAddr.startsWith("https://gosspublic.alicdn.com/"))
      {
         System.out.println("pub key addr must be oss addrss");
         return false;
      }
      // 生成公钥
      String retString = executeGet(pubKeyAddr);
      retString = retString.replace("-----BEGIN PUBLIC KEY-----", "");
      retString = retString.replace("-----END PUBLIC KEY-----", "");
      String queryString = request.getQueryString();
      String uri = request.getRequestURI();
      String decodeUri = java.net.URLDecoder.decode(uri, "UTF-8");
      String authStr = decodeUri;
      if (queryString != null && !queryString.equals("")) {
         authStr += "?" + queryString;
      }
      authStr += "\n" + ossCallbackBody;
      ret = doCheck(authStr, authorization, retString);
      return ret;
   }

   /********************************
    *  @method    : doCheck
    *  @function  : 检查回调签名是否正确
    *  @parameter : [content, sign, publicKey]
    *  @return    : boolean
    *  @date      : 2023/7/21 14:25
    ********************************/
   public static boolean doCheck(String content, byte[] sign, String publicKey) {
      try {
         KeyFactory keyFactory = KeyFactory.getInstance("RSA");
         byte[] encodedKey = BinaryUtil.fromBase64String(publicKey);
         PublicKey pubKey = keyFactory.generatePublic(new X509EncodedKeySpec(encodedKey));
         java.security.Signature signature = java.security.Signature.getInstance("MD5withRSA");
         signature.initVerify(pubKey);
         signature.update(content.getBytes());
         boolean bverify = signature.verify(sign);
         return bverify;

      } catch (Exception e) {
         e.printStackTrace();
      }
      return false;
   }

```

涉及到文件由后端进行上传的代码部分：

```java
File file = new File(filepath);
FileOutputStream fos = new FileOutputStream(file);
    byte[] b = new byte[1024];
    while ((is.read(b)) != -1) {
       fos.write(b);// 写入数据
    }
    is.close();
    fos.close();// 保存数据
//添加写入oss
   
 //添加写入oss
    try {
        ossClient.putObject(aliyunConfig.getBucketName(), "videoRes/share3D/" + mark + "/" + fileName, file);
    } catch (Exception e) {
        e.printStackTrace();
    }
    //把本地的FILE文件删除
    String qrDir = ResourceUtil.getStorageParentXADir() + mark + "/";
    File qrFile = new File(qrDir);
    FileUtil.delBatchFile(qrFile);
    return aliyunConfig.getUrlPrefix() + "/videoRes/shareXA/" + mark + "/" + fileName;
}
```

涉及文件从后端向OSS获取的代码部分：

```java
File file = new File(fileDir);
    String objectName = "videoRes/shareXA/"
            + mark + "/"
            + mark
            + SystemConstant.QR_FILE_NAME_SUFFIX_XA
            + SystemConstant.QR_IMAGE_FORMAT_DEFAULT_PNG;
    try {
        // 下载Object到本地文件，并保存到指定的本地路径中。如果指定的本地文件存在会覆盖，不存在则新建。
        // 先创建出文件
        if (!file.exists()) {
            //先得到文件的上级目录，并创建上级目录，在创建文件
            file.getParentFile().mkdir();
            try {
                //创建文件
                file.createNewFile();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        ossClient.getObject(new GetObjectRequest(aliyunConfig.getBucketName(), objectName), file);
    } catch (Exception e) {
        e.printStackTrace();
    }
    FileUtil.fileToZip(fileDir, zipOut, newVideoName);
    File imageTargetFile = new File(ResourceUtil.getStorageParent3DDir()
            + mark + "/");
    FileUtil.delBatchFile(imageTargetFile);
}
```

涉及文件删除的代码：

```java
public AjaxResult deleteVideos(@RequestBody JSONObject reqObj) {
    JSONArray marksArray = reqObj.getJSONArray("marks");
    if (marksArray.size() == 0) {
        return AjaxResult.failed(ResultCodeEnum.PARAM_FORMAT_ERROR);
    }
    int count = 0;
    for (int i = 0; i < marksArray.size(); i++) {
        String mark = marksArray.getString(i);
        try {
            if (video3DService.remove(new QueryWrapper<Video3D>().eq("mark", mark))) {
                count++;
            }
            favoritesVideo3DService.remove(new QueryWrapper<FavoritesVideo3D>().eq("mark", mark));
            String prefix = "videoRes/share3D/" + mark + "/";
            // 列举所有包含指定前缀的文件并删除。
            String nextMarker = null;
            ObjectListing objectListing = null;
            do {
                ListObjectsRequest listObjectsRequest = new ListObjectsRequest(aliyunConfig.getBucketName())
                        .withPrefix(prefix)
                        .withMarker(nextMarker);
                objectListing = ossClient.listObjects(listObjectsRequest);
                if (objectListing.getObjectSummaries().size() > 0) {
                    List<String> keys = new ArrayList<String>();
                    for (OSSObjectSummary s : objectListing.getObjectSummaries()) {
                        System.out.println("key name: " + s.getKey());
                        keys.add(s.getKey());
                    }
                    DeleteObjectsRequest deleteObjectsRequest = new DeleteObjectsRequest(aliyunConfig.getBucketName()).withKeys(keys).withEncodingType("url");
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
    }
    return AjaxResult.success("删除成功，共删除" + count + "个视频");
}
```
