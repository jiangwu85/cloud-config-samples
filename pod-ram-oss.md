结合之前的沟通记录，这里为你梳理一份在 Kubernetes (ACK) 环境下，Spring Boot 应用通过 RRSA 机制安全访问 OSS 的完整配置步骤和代码示例。

### 🛠️ 第一步：阿里云控制台基础配置

1. **开启 RRSA 功能**：
   登录阿里云容器服务 ACK 控制台，进入你的集群详情页。在“基本信息”页面的“安全与审计”页签下，确保 **RRSA OIDC** 功能已开启。
2. **获取 OIDC 信息**：
   开启后，将鼠标悬浮在“RRSA OIDC 已开启”上，记录下弹出的 **提供商 URL**（OIDC Issuer）和 **提供商 ARN**（OIDC Provider ARN），后续配置会用到。
3. **创建 RAM 角色并配置信任策略**：
   * 在 RAM 访问控制控制台，创建一个用于访问 OSS 的 RAM 角色（例如命名为 `oss-app-role`），并为其授予 OSS 的相关权限（如 `AliyunOSSFullAccess`）。
   * 修改该角色的**信任策略**，允许 ACK 集群通过 OIDC 扮演该角色。模板如下（请替换占位符）：
```json
{
  "Statement": [
    {
      "Action": "sts:AssumeRole",
      "Effect": "Allow",
      "Principal": {
        "Federated": [
          "acs:ram::你的阿里云账号ID:oidc-provider/ack-rrsa-你的集群ID" 
        ]
      },
      "Condition": {
        "StringEquals": {
          "oidc:aud": "sts.aliyuncs.com",
          "oidc:iss": "你的OIDC提供商URL",
          "oidc:sub": "system:serviceaccount:你的命名空间:oss-app-sa"
        }
      }
    }
  ],
  "Version": "1"
}
```

### ☸️ 第二步：Kubernetes YAML 资源配置

在你的 K8s 部署文件中，需要创建 ServiceAccount 并通过注解绑定 RAM 角色，同时在 Pod 中挂载 OIDC Token。以下是完整的 `deployment.yaml` 示例：

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: oss-app-sa
  namespace: default # 👉 替换为你的实际命名空间
  annotations:
    # 👇 核心配置：指定该 SA 绑定的 RAM 角色 ARN
    aliyun.com/role-arn: "acs:ram::你的阿里云账号ID:role/oss-app-role" 
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: springboot-oss-app
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: springboot-oss-app
  template:
    metadata:
      labels:
        app: springboot-oss-app
    spec:
      serviceAccountName: oss-app-sa # 👈 绑定上面创建的 ServiceAccount
      containers:
      - name: springboot-app
        image: your-springboot-image:latest # 👉 替换为你的 Spring Boot 镜像地址
        env:
        # 👇 将 OSS 的配置通过环境变量注入到容器中
        - name: ALIYUN_OSS_ENDPOINT
          value: "oss-cn-hangzhou.aliyuncs.com"
        - name: ALIYUN_OSS_BUCKET_NAME
          value: "your-bucket-name"
        volumeMounts:
        # 👇 挂载 OIDC Token，Spring Boot 代码将读取此路径下的 token 文件
        - name: oidc-token
          mountPath: /var/run/secrets/oidc
          readOnly: true
      volumes:
      - name: oidc-token
        projected:
          sources:
          - serviceAccountToken:
              path: token
              expirationSeconds: 3600 # Token 有效期（秒）
              audience: sts.aliyuncs.com
```

### 💻 第三步：Spring Boot 应用程序实现

在 Spring Boot 应用中，推荐使用阿里云官方的 `credentials-java` 凭据插件配合最新的 OSS Java SDK V2。SDK 会自动读取挂载的 Token 文件、调用 STS 接口换取临时凭证，并完成过期自动刷新，极大简化业务代码。

**1. 引入 Maven 依赖：**
```xml
<!-- 官方凭据管理插件 -->
<dependency>
    <groupId>com.aliyun</groupId>
    <artifactId>credentials-java</artifactId>
    <version>0.3.4</version>
</dependency>
<!-- 阿里云 OSS Java SDK V2 -->
<dependency>
    <groupId>com.aliyun.sdk</groupId>
    <artifactId>alibabacloud-oss-v2</artifactId>
    <version>1.0.2</version> <!-- 建议使用最新稳定版 -->
</dependency>
```

**2. 核心业务代码示例：**
```java
import com.aliyun.credentials.Client;
import com.aliyun.credentials.models.Config;
import com.aliyun.sdk.service.oss2.OSSClient;
import com.aliyun.sdk.service.oss2.credentials.CredentialsProvider;
import com.aliyun.sdk.service.oss2.credentials.CredentialsProviderSupplier;
import com.aliyun.sdk.service.oss2.models.PutObjectRequest;
import darabonba.core.client.ClientOverrideConfiguration;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;
import org.springframework.web.multipart.MultipartFile;

import java.io.File;
import java.io.FileInputStream;
import java.io.InputStream;

@Service
public class OssService {

    @Value("${ALIYUN_OSS_ENDPOINT}")
    private String endpoint;
    
    @Value("${ALIYUN_OSS_BUCKET_NAME}")
    private String bucketName;

    public void uploadFile(MultipartFile file, String objectName) throws Exception {
        // 1. 配置 RRSA 凭据提供者
        Config credentialConfig = new Config();
        credentialConfig.setType("oidc_role_arn"); // 固定为 oidc_role_arn
        credentialConfig.setRoleArn("acs:ram::你的阿里云账号ID:role/oss-app-role"); // RAM 角色 ARN
        credentialConfig.setOidcProviderArn("acs:ram::你的阿里云账号ID:oidc-provider/ack-rrsa-xxx"); // 第一步获取的提供商 ARN
        credentialConfig.setOidcTokenFilePath("/var/run/secrets/oidc/token"); // Pod 内挂载的 Token 路径
        credentialConfig.setRoleSessionName("springboot-oss-session"); // 自定义会话名称
        
        // 2. 创建凭据客户端
        Client credentialClient = new Client(credentialConfig);

        // 3. 将凭据客户端适配给 OSS SDK V2
        CredentialsProvider credentialsProvider = new CredentialsProviderSupplier(credentialClient);

        // 4. 创建 OSS 客户端并执行上传
        try (OSSClient client = OSSClient.newBuilder()
                .credentialsProvider(credentialsProvider)
                .region("cn-hangzhou") // 填写 Bucket 所在的地域 ID
                .overrideConfiguration(ClientOverrideConfiguration.create())
                .build()) {
            
            // 构建上传请求
            InputStream inputStream = new FileInputStream(convertToFile(file));
            PutObjectRequest putObjectRequest = PutObjectRequest.builder()
                    .bucket(bucketName)
                    .key(objectName)
                    .body(inputStream)
                    .build();
            
            // 执行上传
            client.putObject(putObjectRequest);
            System.out.println("文件上传成功：" + objectName);
        }
    }

    // 辅助方法：将 MultipartFile 转换为本地临时 File（根据实际需求优化）
    private File convertToFile(MultipartFile multipartFile) throws Exception {
        File convFile = new File(multipartFile.getOriginalFilename());
        multipartFile.transferTo(convFile);
        return convFile;
    }
}
```

按照以上三个步骤进行配置，你的 Spring Boot 应用就能在 K8s 环境中完全脱离 AccessKey，通过最安全的 RRSA 机制实现对 OSS 的上传下载操作。
