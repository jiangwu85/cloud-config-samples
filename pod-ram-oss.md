保持极致简洁的核心在于：**只保留业务必须的配置，将云原生环境下的凭证获取完全交给阿里云官方 SDK 自动处理**。

以下是去除了所有冗余代码的极简版本：

### 📄 application.yml
仅保留 OSS 的地域、Bucket 名称和 RAM 角色 ARN（用于明确指定要扮演的角色）。

```yaml
aliyun:
  oss:
    region-id: cn-beijing                   # 替换为你的 Bucket 所在地域
    bucket-name: your-bucket-name           # 替换为你的 Bucket 名称
    role-arn: acs:ram::1257099578784861:role/your-role-name  # 替换为你绑定的 RAM Role ARN
```

---

### ☕ Java 核心代码

#### 1. Maven 依赖 (`pom.xml`)
```xml
<dependency>
    <groupId>com.aliyun.oss</groupId>
    <artifactId>aliyun-sdk-oss</artifactId>
    <version>3.17.4</version>
</dependency>
<dependency>
    <groupId>com.aliyun</groupId>
    <artifactId>credentials-java</artifactId>
    <version>0.3.4</version>
</dependency>
```

#### 2. 配置类 (`OssConfig.java`)
利用 `credentials-java` 库的自动化能力，SDK 会自动从 K8s 注入的环境变量中抓取 OIDC Token 文件路径等信息，我们只需指定类型和 RoleArn 即可。

```java
import com.aliyun.credentials.Client;
import com.aliyun.credentials.models.Config;
import com.aliyun.oss.ClientBuilderConfiguration;
import com.aliyun.oss.OSS;
import com.aliyun.oss.OSSClientBuilder;
import com.aliyun.oss.common.auth.CredentialsProvider;
import com.aliyun.oss.common.auth.DefaultCredentials;
import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class OssConfig {

    @Bean
    @ConfigurationProperties(prefix = "aliyun.oss")
    public OssProperties ossProperties() {
        return new OssProperties();
    }

    @Bean
    public OSS ossClient(OssProperties properties) throws Exception {
        // 极简 RRSA 配置：SDK 会自动读取 K8s 注入的标准环境变量
        Config credConfig = new Config();
        credConfig.setType("oidc_role_arn");
        credConfig.setRoleArn(properties.getRoleArn());
        
        Client credentialClient = new Client(credConfig);

        // 封装为 OSS SDK 所需的动态凭证提供者
        CredentialsProvider credentialsProvider = () -> {
            var credential = credentialClient.getCredential();
            return new DefaultCredentials(
                credential.getAccessKeyId(),
                credential.getAccessKeySecret(),
                credential.getSecurityToken()
            );
        };

        // 初始化 OSS 客户端（推荐开启 V4 签名）
        ClientBuilderConfiguration conf = new ClientBuilderConfiguration();
        conf.setSignatureVersion(com.aliyun.oss.common.comm.SignVersion.V4);
        
        return new OSSClientBuilder().build(properties.getRegionId(), credentialsProvider, conf);
    }

    @Data
    public static class OssProperties {
        private String regionId;
        private String bucketName;
        private String roleArn;
    }
}
```

#### 3. 业务服务类 (`OssService.java`)
直接注入 `OSS` 客户端进行日常操作，代码与使用普通 AccessKey 时完全一致。

```java
import com.aliyun.oss.OSS;
import com.aliyun.oss.model.PutObjectRequest;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;
import org.springframework.web.multipart.MultipartFile;

import java.io.IOException;

@Service
public class OssService {

    @Autowired
    private OSS ossClient;

    @Value("${aliyun.oss.bucket-name}")
    private String bucketName;

    public String uploadFile(MultipartFile file, String objectName) throws IOException {
        try {
            // 执行上传
            ossClient.putObject(new PutObjectRequest(bucketName, objectName, file.getInputStream()));
            
            // 返回文件的访问 URL
            return String.format("https://%s.%s.aliyuncs.com/%s", 
                    bucketName, ossClient.getClientConfiguration().getRegionId(), objectName);
        } catch (Exception e) {
            throw new RuntimeException("文件上传失败", e);
        }
    }
}
```
