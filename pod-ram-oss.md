这个方案的核心是：Java 代码完全不知道 RRSA 的存在，所有配置都通过 Kubernetes 环境变量注入，由阿里云 SDK 自动完成身份认证。
📄 1. Spring Boot 配置文件 (application.yml)
这个文件只包含你的业务配置，不包含任何关于 RAM 角色、OIDC 提供商等敏感信息。
yaml

编辑



aliyun:
  oss:
    # OSS Bucket 所在地域
    region-id: ${ALIYUN_OSS_REGION_ID:cn-beijing}
    # OSS Bucket 名称
    bucket-name: ${ALIYUN_OSS_BUCKET_NAME:your-bucket-name}
💻 2. Spring Boot Java 代码
这段代码非常干净，它不关心凭证从何而来，只负责使用凭证。SDK 会自动从环境变量中读取 RRSA 配置并完成认证。
java

编辑



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

    @Value("${aliyun.oss.region-id}")
    private String regionId;

    @Value("${aliyun.oss.bucket-name}")
    private String bucketName;

    public void uploadFile(MultipartFile file, String objectName) throws Exception {
        // 1. 初始化一个空的 Config
        // SDK 会自动检测 ALIBABA_CLOUD_ 开头的环境变量来获取 RRSA 凭证
        Config credentialConfig = new Config();
        Client credentialClient = new Client(credentialConfig);

        // 2. 为 OSS SDK 创建凭证提供者
        CredentialsProvider credentialsProvider = new CredentialsProviderSupplier(credentialClient);

        // 3. 创建 OSS 客户端并执行上传
        try (OSSClient client = OSSClient.newBuilder()
                .credentialsProvider(credentialsProvider)
                .region(regionId)
                .overrideConfiguration(ClientOverrideConfiguration.create())
                .build()) {

            InputStream inputStream = new FileInputStream(convertToFile(file));
            PutObjectRequest putObjectRequest = PutObjectRequest.builder()
                    .bucket(bucketName)
                    .key(objectName)
                    .body(inputStream)
                    .build();

            client.putObject(putObjectRequest);
            System.out.println("文件上传成功：" + objectName);
        }
    }

    private File convertToFile(MultipartFile multipartFile) throws Exception {
        File convFile = new File(multipartFile.getOriginalFilename());
        multipartFile.transferTo(convFile);
        return convFile;
    }
}
☸️ 3. Kubernetes Deployment 配置 (关键部分)
这是整个方案的“指挥中心”。你需要将信任策略中的信息，通过环境变量的方式注入到 Pod 中。
yaml

编辑



apiVersion: apps/v1
kind: Deployment
metadata:
  name: springboot-oss-app
  namespace: backend-dev
spec:
  template:
    spec:
      serviceAccountName: default # 必须与信任策略中的 serviceaccount 名称一致
      containers:
      - name: springboot-app
        image: your-springboot-image:latest
        env:
        # --- RRSA 核心配置 (从这里开始) ---
        
        # 1. 指定 OIDC Token 文件的路径 (必须与下方 volumeMounts 的路径一致)
        - name: ALIBABA_CLOUD_OIDC_TOKEN_FILE
          value: "/var/run/secrets/oidc/token"
        
        # 2. 从你的信任策略中获取的 RAM 角色 ARN
        - name: ALIBABA_CLOUD_ROLE_ARN
          value: "acs:ram::1257099578784861:role/你的RAM角色名称"
        
        # 3. 从你的信任策略中获取的 OIDC 提供商 ARN
        - name: ALIBABA_CLOUD_OIDC_PROVIDER_ARN
          value: "acs:ram::1257099578784861:oidc-provider/ack-rrsa-c22741d64a9dc4b51886803b90aa95408"
        
        # 4. 自定义会话名称
        - name: ALIBABA_CLOUD_ROLE_SESSION_NAME
          value: "springboot-oss-app"
        
        # --- 业务配置 ---
        - name: ALIYUN_OSS_REGION_ID
          value: "cn-beijing"
        - name: ALIYUN_OSS_BUCKET_NAME
          value: "your-bucket-name"
          
        volumeMounts:
        # 挂载 OIDC Token，路径必须与 ALIBABA_CLOUD_OIDC_TOKEN_FILE 的值一致
        - name: oidc-token
          mountPath: /var/run/secrets/oidc
          readOnly: true
      volumes:
      - name: oidc-token
        projected:
          sources:
          - serviceAccountToken:
              path: token
              expirationSeconds: 3600
              # audience 必须与信任策略中的 oidc:aud 一致
              audience: sts.aliyuncs.com
