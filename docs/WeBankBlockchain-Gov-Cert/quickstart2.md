
# cert-mgr使用

## 功能介绍
cert-mgr用于证书托管，适合B2B2C场景。

证书管理中，证书相关的私钥由单独的私钥表保存，还包含了证书表和请求表，生成的证书会保存在证书表中，子证书的请求会保存在请求表中。

证书签名算法目前支持：

*   SHA256WITHRSA
*   SHA256WITHECDSA
*   SM3WITHSM2

## 前置依赖

在使用本组件前，请确认系统环境已安装相关依赖软件，清单如下：

| 依赖软件 | 说明 |备注|
| --- | --- | --- |
| Java | JDK[1.8] | |
| Git | 下载的安装包使用Git | |

如果您还未安装这些依赖，请参考[附录](appendix.md)。

## 部署教程

目前支持从源码进行部署。

### 获取源码

通过git下载源码：

```
https://github.com/WeBankBlockchain/Gov-Cert.git
```

进入目录：
```
cd Gov-Cert/cert-mgr
```

### 编译源码

方式一：如果服务器已安装Gradle
```
gradle build -x test
```

方式二：如果服务器未安装Gradle，使用gradlew编译
```
chmod +x ./gradlew && ./gradlew build -x test
```

### 导入jar包

cert-mgr编译之后在根目录下会生成dist文件夹，文件夹中包含cert-mgr.jar。可以将cert-mgr.jar导入到自己的项目中，例如拷贝到libs目录下，然后进行依赖配置。gradle推荐依赖配置如下，然后再对自己的项目进行编译。

```
repositories {
    mavenCentral()
    mavenLocal()
    maven {
        url "http://maven.aliyun.com/nexus/content/groups/public/"
    }
}

dependencies {
    compile 'org.springframework.boot:spring-boot-starter'
    compile 'org.springframework.boot:spring-boot-starter-data-jpa'

    testCompile('org.springframework.boot:spring-boot-starter-test') {
        exclude group: 'org.junit.vintage', module: 'junit-vintage-engine'
        //exclude group: 'junit', module: 'junit'
    }
    compile 'org.springframework.boot:spring-boot-starter-jta-atomikos'
    compile ('org.projectlombok:lombok:1.18.8')
    compile ('org.projectlombok:lombok:1.18.8')
    annotationProcessor 'org.projectlombok:lombok:1.18.8'
    compile "org.apache.commons:commons-lang3:3.6"
    compile "commons-io:commons-io:2.6"

    compile "com.fasterxml.jackson.core:jackson-core:2.9.6"
    compile "com.fasterxml.jackson.core:jackson-databind:2.9.6"
    compile "com.fasterxml.jackson.core:jackson-annotations:2.9.6"

    compile 'com.lhalcyon:bip32:1.0.0'
    //compile 'io.github.novacrypto:BIP44:0.0.3'

    compile group: 'org.bouncycastle', name: 'bcprov-jdk15on', version: '1.60'
    compile group: 'org.bouncycastle', name: 'bcpkix-jdk15on', version: '1.60'
    compile 'org.web3j:core:3.4.0'
    compile 'com.lambdaworks:scrypt:1.4.0'
    compile 'commons-codec:commons-codec:1.9'

    compile 'mysql:mysql-connector-java'
    compile fileTree(dir:'libs',include:['*.jar'])
}

```


## 使用详解

cert-mgr使用了SpringBoot自动装配功能，所以只要您按照上文添加了SpringBoot依赖，就可以自动装配所需的Bean。

### 配置

请参考下面的模板，配置application.properties。
```
## 加密后的私钥存储url
spring.datasource.url=jdbc:mysql://[ip]:[port]/pkey_mgr?autoReconnect=true&characterEncoding=utf8&useSSL=false&serverTimezone=GMT%2b8
spring.datasource.username=
spring.datasource.password=

## spring jpa config
spring.jpa.properties.hibernate.hbm2ddl.auto=update
spring.jpa.properties.hibernate.show_sql=true
spring.jpa.database-platform=org.hibernate.dialect.MySQL5InnoDBDialect
```

### 建表

如果在上述配置中指定了**spring.jpa.properties.hibernate.hbm2ddl.auto=update**，则jpa会帮助用户自动建立数据表。

如果不希望自动建立数据表，请先关闭jpa建表开关：
```
spring.jpa.properties.hibernate.hbm2ddl.auto=validate
```
然后按下面方式手动建表。

1） 在数据源运行下述建表语句：

```
 
 //证书管理
 -- Create syntax for TABLE 'cert_keys_info'
 drop table if exists cert_keys_info;
 CREATE TABLE `cert_keys_info` (
   `pk_id` bigint(20) unsigned NOT NULL AUTO_INCREMENT,
   `user_id` varchar(255) NOT NULL,
   `key_alg` varchar(8) NOT NULL,
   `key_pem` longtext NOT NULL,
   `creat_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
   `update_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
   PRIMARY KEY (`pk_id`)
 ) ENGINE=InnoDB DEFAULT CHARSET=utf8;
 
 -- Create syntax for TABLE 'cert_info'
 drop table if exists cert_info;
 CREATE TABLE `cert_info` (
   `pk_id` bigint(20) unsigned NOT NULL AUTO_INCREMENT,
   `user_id` varchar(255) NOT NULL,
   `subject_pub_key` longtext NOT NULL,
   `cert_content` longtext NOT NULL,
   `issuer_key_id` bigint(20) NOT NULL,
   `subject_key_id` bigint(20) NOT NULL,
   `parent_cert_id` bigint(20),
   `serial_number` varchar(255) NOT NULL,
   `issuer_org` varchar(255) NOT NULL,
   `issuer_cn` varchar(255) NOT NULL,
   `subject_org` varchar(255) NOT NULL,
   `subject_cn` varchar(255) NOT NULL,
   `is_ca_cert` int(4) NOT NULL,
   `creat_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
   `update_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
   PRIMARY KEY (`pk_id`),
   UNIQUE KEY (`parent_cert_id`,`serial_number`)
 ) ENGINE=InnoDB DEFAULT CHARSET=utf8;
 
 -- Create syntax for TABLE 'cert_request_info'
 drop table if exists cert_request_info;
 CREATE TABLE `cert_request_info` (
   `pk_id` bigint(20) unsigned NOT NULL AUTO_INCREMENT,
   `parent_cert_id` bigint(20),
   `subject_key_id` bigint(20) NOT NULL,
   `user_id` varchar(255) NOT NULL,
   `cert_request_content` longtext NOT NULL,
   `subject_org` varchar(255) NOT NULL,
   `subject_cn` varchar(255) NOT NULL,
   `creat_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
   `update_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
   PRIMARY KEY (`pk_id`),
   UNIQUE KEY (`parent_cert_id`,`subject_key_id`)
 ) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

### 接口使用

CertManagerService类是证书管理的统一入口，覆盖证书管理的全生命周期，包含如下功能：

*   createRootCert : 生成根证书
*   createRootCertByHexPriKey ：私钥Hex格式作为入参生成证书
*   createCertRequest：生成请求
*   createCertRequestByHexPriKey ：私钥Hex格式作为入参生成请求
*   createChildCert：生成子证书
*   resetCertificate：证书重置
*   queryCertList：证书列表查询
*   queryCertRequestList：请求列表查询
*   queryCertKeyList：证书私钥列表查询
*   queryCertInfoByCertId：根据id查询证书
*   queryCertRequestByCsrId：根据id证书请求
*   exportCertToFile：证书导出

#### createRootCert

生成根证书，提供了多种封装接口，可按需使用

```
    @Test
    public void testCreateRootCert0() throws Exception{
        X500NameInfo issuer = X500NameInfo.builder()
                .commonName("chain")
                .organizationName("fisco-bcos")
                .organizationalUnitName("chain")
                .build();
        String userId = "wangyue";
        String cert = certManagerService.createRootCert(userId,issuer);
        System.out.println(cert);
    }

    @Test
    public void testCreateRootCert1() throws Exception{
        X500NameInfo issuer = X500NameInfo.builder()
                .commonName("chain")
                .organizationName("fisco-bcos")
                .organizationalUnitName("chain")
                .build();
        String userId = "wangyue";
        Date beginDate = new Date();
        Date endDate = new Date(beginDate.getTime() + CertConstants.DEFAULT_VALIDITY);
        String cert = certManagerService.createRootCert(userId,issuer,beginDate,endDate);
        System.out.println(cert);
    }

    @Test
    public void testCreateRootCert3() throws Exception{
        X500NameInfo issuer = X500NameInfo.builder()
                .commonName("chain")
                .organizationName("fisco-bcos")
                .organizationalUnitName("chain")
                .build();
        String userId = "wangyue";
        Date beginDate = new Date();
        Date endDate = new Date(beginDate.getTime() + CertConstants.DEFAULT_VALIDITY);
        KeyUsage keyUsage = new KeyUsage(KeyUsage.dataEncipherment);
        String cert = certManagerService.createRootCert(userId,1,issuer,keyUsage,beginDate,endDate);
        System.out.println(cert);
    }

    @Test
    public void testCreateRootCert4() throws Exception{
        X500NameInfo issuer = X500NameInfo.builder()
                .commonName("chain")
                .organizationName("fisco-bcos")
                .organizationalUnitName("chain")
                .build();
        String userId = "wangyue";
        Date beginDate = new Date();
        Date endDate = new Date(beginDate.getTime() + CertConstants.DEFAULT_VALIDITY);
        String pemPriKey = "此处填入私钥";

        String str = certManagerService.createRootCert(userId,pemPriKey,KeyAlgorithmEnums.RSA,issuer,beginDate,endDate);
        System.out.println(str);
    }
```

执行过后，会生成根证书并保存

**涉及参数说明**：

- userId: 用户id

- issuer: 签发者信息

- beginDate：证书生效时间

- endDate：证书失效时间

- keyUsage：证书用途

- certKeyId：证书签名私钥id


#### createRootCertByHexPriKey

私钥Hex格式作为入参生成根证书

```
    @Test
    public void testCreateRootCertByHexPriKey() throws Exception{
        X500NameInfo issuer = X500NameInfo.builder()
                .commonName("chain")
                .organizationName("fisco-bcos")
                .organizationalUnitName("chain")
                .build();
        String userId = "wangyue";
        Date beginDate = new Date();
        Date endDate = new Date(beginDate.getTime() + CertConstants.DEFAULT_VALIDITY);
        KeyPair keyPair = KeyUtils.generateKeyPair();
        String hexPriKey = Numeric.toHexString(keyPair.getPrivate().getEncoded());
        String cert = certManagerService.createRootCertByHexPriKey(userId,hexPriKey,KeyAlgorithmEnums.RSA,issuer,beginDate,endDate);
        System.out.println(cert);
    }
```
执行过后，会生成根证书并保存

**涉及参数说明**：

- userId: 用户id

- issuer: 签发者信息

- beginDate：证书生效时间

- endDate：证书失效时间

- hexPriKey：证书签名私钥Hex格式


#### createCertRequest

生成用于生成子证书的请求，提供了两个封装接口，可按需使用

```
    @Test
    public void testCreateCertRequest0() throws Exception{
        X500NameInfo subject = X500NameInfo.builder()
                .commonName("agancy")
                .organizationName("fisco-bcos")
                .organizationalUnitName("agancy")
                .build();
        String userId = "wangyue1";
        String csr;
        csr = certManagerService.createCertRequest(userId,1, subject);
        System.out.println(csr);
    }

    @Test
    public void testCreateCertRequest1() throws Exception{
        X500NameInfo subject = X500NameInfo.builder()
                .commonName("agancy")
                .organizationName("fisco-bcos")
                .organizationalUnitName("agancy")
                .build();
        String userId = "wangyue1";
        KeyPairGenerator keyPairGenerator = KeyPairGenerator.getInstance("ECDSA", "BC");
        ECGenParameterSpec ecGenParameterSpec = new ECGenParameterSpec("secp256k1");
        keyPairGenerator.initialize(ecGenParameterSpec, new SecureRandom());
        KeyPair keyPair = keyPairGenerator.generateKeyPair();
        PrivateKey privateKey = keyPair.getPrivate();

        String csr = certManagerService.createCertRequest(userId, CertUtils.readPEMAsString(privateKey),
                KeyAlgorithmEnums.ECDSA,1,subject);
        System.out.println(csr);
    }

```
执行过后，会生成请求并保存

**涉及参数说明**：

- userId: 用户id

- subject: 请求方信息

- issuerCertId: 签发证书id

- privateKey：请求签名私钥串

- certKeyId：请求签名私钥id


#### createCertRequestByHexPriKey

私钥Hex格式作为入参生成请求

```
    @Test
    public void testCreateCertRequestByHexPriKey() throws Exception{
        X500NameInfo subject = X500NameInfo.builder()
                .commonName("agancy")
                .organizationName("fisco-bcos")
                .organizationalUnitName("agancy")
                .build();
        String userId = "wangyue";
        String hexPriKey = "3500db68433dda968ef7bfe5a0ed6926b8e85aabcd2caa54f8327ca07ac73526";
        String cert = certManagerService.createCertRequestByHexPriKey(userId,hexPriKey,KeyAlgorithmEnums.ECDSA,3,subject);
        System.out.println(cert);
    }

```
执行过后，会生成请求并保存

**涉及参数说明**：

- userId: 用户id

- subject: 请求者信息

- issuerCertId: 签发证书id

- keyAlg: 密钥算法

- hexPriKey：证书签名私钥Hex格式


#### createChildCert

生成子证书

```
    @Test
    public void testCreateChildCert() throws Exception{
        String userId = "wangyue1";
        String child;
        child = certManagerService.createChildCert(userId,4);
        System.out.println(child);
    }
```
执行过后，会生成子证书并保存

**涉及参数说明**：

- userId: 用户id

- csrId: 请求id


#### resetCertificate

证书重置

```
    @Test
    public void testResetCertificate() throws Exception{
        String userId = "wangyue1";
        Date beginDate = new Date();
        Date endDate = new Date(beginDate.getTime() + CertConstants.DEFAULT_VALIDITY);
        String root = certManagerService.resetCertificate(userId,9,
                new KeyUsage(KeyUsage.dataEncipherment),
                beginDate,endDate);
        System.out.println(root);
    }
```

执行过后，会重置证书并保存

**涉及参数说明**：

- userId: 用户id

- certId: 重置证书id

- keyUsage：证书用途

- beginDate：证书生效时间

- endDate：证书失效时间


#### queryCertList

证书列表查询，多条件联合查询

```
    @Test
    public void testQueryCertList() {
        String userId = "wangyue";
        List<CertVO> list = certManagerService.queryCertList(
                userId,null,null,null,null,null);
        System.out.println();
    }
```

执行过后，会得到证书列表

**涉及参数说明**：

- userId: 用户id

- issuerKeyId: 签发私钥id

- pCertId：签发证书id

- issuerOrg：签发机构名

- issuerCN：签发者公共名称

- isCACert：是否ca机构


#### queryCertRequestList

证书请求查询，多条件联合查询

```
    @Test
    public void testQueryCertRequestList() {
        String userId = "wangyue";
        List<CertRequestInfo> list = certManagerService.queryCertRequestList(
                userId,null,null,null,null);
        System.out.println();
    }
```

执行过后，会得到证书请求列表

**涉及参数说明**：

- userId: 用户id

- subjectKeyId: 请求签名私钥id

- pCertId：签发证书id

- subjectOrg：申请机构名

- subjectCN：申请者公共名称


#### queryCertKeyList

证书私钥查询，会返回私钥列表，但不返回私钥明文

```
    @Test
    public void testQueryCertKeyList() {
        String userId = "wangyue";
        List<CertKeyVO> list = certManagerService.queryCertKeyList(userId);
        System.out.println();
    }
```

执行过后，会得到证书私钥列表

**涉及参数说明**：

- userId: 用户id


#### queryCertInfoByCertId

根据id查询证书

```
 @Test
    public void testQueryCertInfoByCertId() {
        CertVO certInfo = certManagerService.queryCertInfoByCertId(1L);
        System.out.println();
    }    
```

执行过后，会得到证书

**涉及参数说明**：

- certId: 证书id

#### queryCertRequestByCsrId

根据id查询证书请求

```
    @Test
    public void testQueryCertRequestByCsrId() {
        CertRequestVO keyRequestVO = certManagerService.queryCertRequestByCsrId(1L);
        System.out.println();
    }  
```

执行过后，会得到证书请求

**涉及参数说明**：

- csrId: 证书请求id


#### exportCertToFile

证书导出

```
    @Test
    public void testExportCertToFile() throws Exception {
        certManagerService.exportCertToFile(1L,"src/ca.crt");
        System.out.println();
    }
```

执行过后，证书导出到执行文件目录

**涉及参数说明**：

- certId: 证书id

- filePath: 证书导出路径







