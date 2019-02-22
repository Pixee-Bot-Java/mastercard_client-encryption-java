# client-encryption-java

[![](https://travis-ci.org/Mastercard/client-encryption-java.svg?branch=master)](https://travis-ci.org/Mastercard/client-encryption-java)
[![](https://sonarcloud.io/api/project_badges/measure?project=Mastercard_client-encryption-java&metric=alert_status)](https://sonarcloud.io/dashboard?id=Mastercard_client-encryption-java) 
[![](https://img.shields.io/maven-central/v/com.mastercard.developer/client-encryption.svg)](https://search.maven.org/artifact/com.mastercard.developer/client-encryption/)
[![](https://img.shields.io/badge/license-MIT-yellow.svg)](https://github.com/Mastercard/client-encryption-java/blob/master/LICENSE)

## Table of Contents
- [Overview](#overview)
  * [Compatibility](#compatibility)
  * [References](#references)
- [Usage](#usage)
  * [Prerequisites](#prerequisites)
  * [Adding the Library to Your Project](#adding-the-library-to-your-project)
  * [Loading the Encryption Certificate](#loading-the-encryption-certificate) 
  * [Loading the Decryption Key](#loading-the-decryption-key)
  * [Performing Field Level Encryption and Decryption](#performing-field-level-encryption-and-decryption)
  * [Integrating with OpenAPI Generator API Client Libraries](#integrating-with-openapi-generator-api-client-libraries)

## Overview <a name="overview"></a>
Zero dependency library for Mastercard API compliant encryption/decryption.

### Compatibility <a name="compatibility"></a>
Java 7+

### References <a name="references"></a>
* [Encryption of sensitive data guide](https://developer.mastercard.com/page/mdes-token-connect-encryption-of-sensitive-data)

## Usage <a name="usage"></a>
### Prerequisites <a name="prerequisites"></a>
Before using this library, you will need to set up a project in the [Mastercard Developers Portal](https://developer.mastercard.com). 

As part of this set up, you'll receive:
* A public request encryption certificate (aka _Client Encryption Keys_)
* A private response decryption key (aka _Mastercard Encryption Keys_)

### Adding the Library to Your Project <a name="adding-the-library-to-your-project"></a>

#### Maven
```xml
<dependency>
    <groupId>com.mastercard.developer</groupId>
    <artifactId>client-encryption</artifactId>
    <version>${client-encryption-version}</version>
</dependency>
```

#### Gradle
```
dependencies {
	implementation "com.mastercard.developer:client-encryption:$clientEncryptionVersion"
}	
```

#### Other Dependency Managers
See: https://search.maven.org/artifact/com.mastercard.developer/client-encryption

### Loading the Encryption Certificate <a name="loading-the-encryption-certificate"></a>

A `Certificate` object can be created from a PEM file by calling the `EncryptionUtils.loadEncryptionCertificate` method:
```java
Certificate encryptionCertificate = EncryptionUtils.loadEncryptionCertificate("<insert PEM certificate file path>");
```

### Loading the Decryption Key <a name="loading-the-decryption-key"></a>

#### From a PKCS#12 file

A `PrivateKey` key object can be created from a PKCS#12 file by calling the `EncryptionUtils.loadDecryptionKey` method:
```java
PrivateKey decryptionKey = EncryptionUtils.loadDecryptionKey(
						"<insert PKCS#12 key file path>", 
						"<insert key alias>", 
						"<insert key password>");
```

#### From a PKCS#8 (DER) encoded content

A `PrivateKey` object can be created from a PKCS#8 key file by calling the `EncryptionUtils.loadDecryptionKey` method:
```java
PrivateKey decryptionKey = EncryptionUtils.loadDecryptionKey("<insert PKCS#8 file path>");
```

#### From a PEM file

Reading PEM encoded keys requires an additional step:

1. Convert the key using: `openssl pkcs8 -topk8 -inform PEM -outform DER -in key.pem -out key.der -nocrypt`
2. Call `EncryptionUtils.loadDecryptionKey` (see above)

### Performing Field Level Encryption and Decryption <a name="performing-field-level-encryption-and-decryption"></a>
The methods that do all the heavy lifting are `encryptPayload` and `decryptPayload` in the `FieldLevelEncryption` class.

Usage:
```java
String encryptedRequestPayload = FieldLevelEncryption.encryptPayload(requestPayload, config);
String responsePayload = FieldLevelEncryption.decryptPayload(encryptedResponsePayload, config);
```

#### Configuring the Field Level Encryption
Use the `FieldLevelEncryptionConfigBuilder` to create `FieldLevelEncryptionConfig` instances. Example:
```java
FieldLevelEncryptionConfig config = FieldLevelEncryptionConfigBuilder.aFieldLevelEncryptionConfig()
                .withEncryptionCertificate(encryptionCertificate)
                .withDecryptionKey(decryptionKey)
                .withEncryptionPath("$.path.to.foo", "$.path.to.encryptedFoo")
                .withDecryptionPath("$.path.to.encryptedFoo", "$.path.to.foo")
                .withMgf1ParameterSpec(MGF1ParameterSpec.SHA256)
                .withEncryptedValueFieldName("encryptedValue")
                .withEncryptedKeyFieldName("encryptedKey")
                .withIvFieldName("iv")
                .withFieldValueEncoding(FieldValueEncoding.HEX)
                .build();
```

See also: [FieldLevelEncryptionConfig.java](TODO)

#### Performing Encryption

Call `FieldLevelEncryption.encryptPayload` with a JSON request payload and a `FieldLevelEncryptionConfig` instance.

Example using the configuration above:

* Request payload:
```json
{
    "path": {
        "to": {
            "foo": {
                "sensitiveField1": "sensitiveValue1",
                "sensitiveField2": "sensitiveValue2"
            }
        }
    }
}
```

* Encrypted request payload:
```json
{
    "path": {
        "to": {
            "encryptedFoo": {
                "iv": "7f1105fb0c684864a189fb3709ce3d28",
                "encryptedKey": "67f467d1b653d98dddd7eea5ae411a0c6d3c(...)ffd4c09dd42f713b6b39313503be179bae18a51bff2b48f937c8",
                "encryptedValue": "b73aabd267517f79c54e84ff01d8bc09ed72455c2(...)dffb5fa04bf4ca4fff907d67072a75b076e6ce9ade1ff514ed6141",
                "oaepHashingAlgorithm": "SHA256"
            }
        }
    }
}
```

#### Performing Decryption

Call `FieldLevelEncryption.decryptPayload` with a JSON response payload and a `FieldLevelEncryptionConfig` instance.

Encrypted response payload:
```json
{
    "path": {
        "to": {
            "encryptedFoo": {
                "iv": "e5d313c056c411170bf07ac82ede78c9",
                "encryptedKey": "e3a56746c0f9109d18b3a2d91619e6cac86bcc7b09652b76(...)f16d8af7e006440f17677eaaeff36b2479652f5c24ae7bd",
                "encryptedValue": "809a09d78257af5379df0c454dfa9c3e2ecf5787430775ebcf(...)409d8d27ab29803353cded59fe72fd4a7735c69da4080e74f",
                "oaepHashingAlgorithm": "SHA256"
            }
        }
    }
}
```

Response payload:
```json
{
    "path": {
        "to": {
            "foo": {
                "sensitiveField1": "sensitiveValue1",
                "sensitiveField2": "sensitiveValue2"
            }
        }
    }
}
```

### Integrating with OpenAPI Generator API Client Libraries <a name="integrating-with-openapi-generator-api-client-libraries"></a>

[OpenAPI Generator](https://github.com/OpenAPITools/openapi-generator) generates API client libraries from [OpenAPI Specs](https://github.com/OAI/OpenAPI-Specification). 
It provides generators and library templates for supporting multiple languages and frameworks.

The `com.mastercard.developer.interceptors` package will provide you with some interceptor classes you can use when configuring your API client. These classes will take care of encrypting/decrypting request and response payloads.

Library options currently supported for the `java` generator:
+ [okhttp-gson](#okhttp-gson)
+ [retrofit](#retrofit)
+ [retrofit2](#retrofit2)

See also:
* [OpenAPI Generator (maven Plugin)](https://mvnrepository.com/artifact/org.openapitools/openapi-generator-maven-plugin) 
* [CONFIG OPTIONS for java](https://github.com/OpenAPITools/openapi-generator/blob/master/docs/generators/java.md)

#### okhttp-gson <a name="okhttp-gson"></a>
##### OpenAPI Generator Plugin Configuration
```xml
<configuration>
    <inputSpec>${project.basedir}/src/main/resources/openapi-spec.yaml</inputSpec>
    <generatorName>java</generatorName>
    <library>okhttp-gson</library>
    <!-- ... -->
</configuration>
```

##### Usage of the `OkHttp2FieldLevelEncryptionInterceptor`
```java
ApiClient client = new ApiClient();
client.setBasePath("https://sandbox.api.mastercard.com");
List<Interceptor> interceptors = client.getHttpClient().networkInterceptors();
interceptors.add(new OkHttp2FieldLevelEncryptionInterceptor(config));
interceptors.add(new OkHttp2OAuth1Interceptor(consumerKey, signingKey));
ServiceApi serviceApi = new ServiceApi(client);
// ...
```

#### retrofit <a name="retrofit"></a>
##### OpenAPI Generator Plugin Configuration
```xml
<configuration>
    <inputSpec>${project.basedir}/src/main/resources/openapi-spec.yaml</inputSpec>
    <generatorName>java</generatorName>
    <library>retrofit</library>
    <!-- ... -->
</configuration>
```

##### Usage of the `OkHttp2FieldLevelEncryptionInterceptor`
```java
ApiClient client = new ApiClient();
RestAdapter.Builder adapterBuilder = client.getAdapterBuilder();
adapterBuilder.setEndpoint("https://sandbox.api.mastercard.com"); 
List<Interceptor> interceptors = client.getOkClient().networkInterceptors();
interceptors.add(new OkHttp2FieldLevelEncryptionInterceptor(config));
interceptors.add(new OkHttp2OAuth1Interceptor(consumerKey, signingKey));
ServiceApi serviceApi = client.createService(ServiceApi.class);
// ...
```

#### retrofit2 <a name="retrofit2"></a>
##### OpenAPI Generator Plugin Configuration
```xml
<configuration>
    <inputSpec>${project.basedir}/src/main/resources/openapi-spec.yaml</inputSpec>
    <generatorName>java</generatorName>
    <library>retrofit2</library>
    <!-- ... -->
</configuration>
```

##### Usage of the `OkHttpFieldLevelEncryptionInterceptor`
```java
ApiClient client = new ApiClient();
Retrofit.Builder adapterBuilder = client.getAdapterBuilder();
adapterBuilder.baseUrl("https://sandbox.api.mastercard.com"); 
OkHttpClient.Builder okBuilder = client.getOkBuilder();
okBuilder.addNetworkInterceptor(new OkHttpFieldLevelEncryptionInterceptor(config));
okBuilder.addNetworkInterceptor(new OkHttpOAuth1Interceptor(consumerKey, signingKey));
ServiceApi serviceApi = client.createService(ServiceApi.class);
// ...
```