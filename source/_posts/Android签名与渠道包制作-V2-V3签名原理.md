title: Android签名与渠道包制作-V2/V3签名原理
date: 2021-04-07 19:57:13
tags:
    - 技术相关
    - Android
---
系列文章:

- [Android签名与渠道包制作-V1版本](https://blog.islinjw.cn/2021/04/07/Android%E7%AD%BE%E5%90%8D%E4%B8%8E%E6%B8%A0%E9%81%93%E5%8C%85%E5%88%B6%E4%BD%9C-V1%E7%89%88%E6%9C%AC/)
- [Android签名与渠道包制作-V2/V3签名原理](https://blog.islinjw.cn/2021/04/07/Android%E7%AD%BE%E5%90%8D%E4%B8%8E%E6%B8%A0%E9%81%93%E5%8C%85%E5%88%B6%E4%BD%9C-V2-V3%E7%AD%BE%E5%90%8D%E5%8E%9F%E7%90%86/)




正如[上一篇文章](https://blog.islinjw.cn/2021/04/07/Android%E7%AD%BE%E5%90%8D%E4%B8%8E%E6%B8%A0%E9%81%93%E5%8C%85%E5%88%B6%E4%BD%9C-V1%E7%89%88%E6%9C%AC/)说的,V1版本的签名机制漏洞在于它没有给整个apk包做校验，而且校验的时候需要解压。V2版本的签名机制就是为了解决这两个问题而出现的。

# zip包文件格式

为了了解V2版本的签名原理，我们需要更加深入的了解下zip包的文件格式。由于zip的解析是从后往前的，大体格式如下:

{% img /Android签名与渠道包制作V2V3签名原理/1.png %}

eocd的倒数第三部分[offset of start of central directory with respect to the starting disk number]标记了central directory的偏移:

```
End of central directory record:

        end of central dir signature    4 bytes  (0x06054b50)
        number of this disk             2 bytes
        number of the disk with the
        start of the central directory  2 bytes
        total number of entries in the
        central directory on this disk  2 bytes
        total number of entries in
        the central directory           2 bytes
        size of the central directory   4 bytes
        offset of start of central
        directory with respect to
        the starting disk number        4 bytes
        .ZIP file comment length        2 bytes
        .ZIP file comment       (variable size)
```

## central directory

我们直接以一个例子来说明:

{% img /Android签名与渠道包制作V2V3签名原理/2.png %}

由于zip包是小端序号,所以实际的值应该是0x00149928，这个地址就代表着central directory的起始地址，我们对比central directory的文件结构:

```
 Central directory structure:

    [file header 1]
    .
    .
    . 
    [file header n]
    [digital signature] 

    File header:

      central file header signature   4 bytes  (0x02014b50)
      version made by                 2 bytes
      version needed to extract       2 bytes
      general purpose bit flag        2 bytes
      compression method              2 bytes
      last mod file time              2 bytes
      last mod file date              2 bytes
      crc-32                          4 bytes
      compressed size                 4 bytes
      uncompressed size               4 bytes
      file name length                2 bytes
      extra field length              2 bytes
      file comment length             2 bytes
      disk number start               2 bytes
      internal file attributes        2 bytes
      external file attributes        4 bytes
      relative offset of local header 4 bytes

      file name (variable size)
      extra field (variable size)
      file comment (variable size)

    Digital signature:

      header signature                4 bytes  (0x05054b50)
      size of data                    2 bytes
      signature data (variable size)
```

一堆的文件头，和一个签名。我们在zip包中找到0x00149928这个位置:

{% img /Android签名与渠道包制作V2V3签名原理/3.png %}

根据上面的格式定义将对应的数据列举出来:

| 地址       | 长度                           | 内容                                      | 值                                                           | 小端序实际值                       |
| ---------- | ------------------------------ | ----------------------------------------- | ------------------------------------------------------------ | ---------------------------------- |
| 0x00149928 | 4 bytes                        | central file header signature(0x02014b50) | 0x504B0102                                                   | 0x02014B50                         |
| 0x0014992C | 2 bytes                        | version made by                           | 0x0000                                                       | 0x0000                             |
| 0x0014992E | 2 bytes                        | version needed to extract                 | 0x0000                                                       | 0x0000                             |
| 0x00149930 | 2 bytes                        | general purpose bit flag                  | 0x0000                                                       | 0x0000                             |
| 0x00149932 | 2 bytes                        | compression method                        | 0x0800                                                       | 0x0008                             |
| 0x00149934 | 2 bytes                        | last mod file time                        | 0x0000                                                       | 0x0000                             |
| 0x00149936 | 2 bytes                        | last mod file date                        | 0x0000                                                       | 0x0000                             |
| 0x00149938 | 4 bytes                        | crc-32                                    | 0x39B6CD57                                                   | 0x57CDB639                         |
| 0x0014993C | 4 bytes                        | compressed size                           | 0x12030000                                                   | 0x00000312                         |
| 0x00149940 | 4 bytes                        | uncompressed size                         | 0x98080000                                                   | 0x00000898                         |
| 0x00149944 | 2 bytes                        | file name length                          | 0x1300                                                       | 0x0013                             |
| 0x00149946 | 2 bytes                        | extra field length                        | 0x0000                                                       | 0x0000                             |
| 0x00149948 | 2 bytes                        | file comment length                       | 0x0000                                                       | 0x0000                             |
| 0x0014994A | 2 bytes                        | disk number start                         | 0x0000                                                       | 0x0000                             |
| 0x0014994C | 2 bytes                        | internal file attributes                  | 0x0000                                                       | 0x0000                             |
| 0x0014994E | 4 bytes                        | external file attributes                  | 0x00000000                                                   | 0x00000000                         |
| 0x00149952 | 4 bytes                        | relative offset of local header           | 0x00000000                                                   | 0x00000000                         |
| 0x00149956 | variable size<br/>(0x0013==19) | file name                                 | 0x41 0x6E 0x64 0x72 0x6F 0x69 0x64 0x4D 0x61 0x6E 0x69 0x66 0x65 0x73 0x74 0x2E 0x78 0x6D 0x6C | ASCII码的值为：AndroidManifest.xml |
| -          | variable size(0)               | extra field                               | (空)                                                         | (空)                               |
| -          | variable size(0)               | file comment                              | (空)                                                         | (空)                               |

所以我们找到第一个文件AndroidManifest.xml的[relative offset of local header]为0x00000000，即local file header 1的地址是0x00000000。

## Local file header

0x00000000的内容如下:

{% img /Android签名与渠道包制作V2V3签名原理/4.png %}

Local file header的格式如下:

```
Local file header:

        local file header signature     4 bytes  (0x04034b50)
        version needed to extract       2 bytes
        general purpose bit flag        2 bytes
        compression method              2 bytes
        last mod file time              2 bytes
        last mod file date              2 bytes
        crc-32                          4 bytes
        compressed size                 4 bytes
        uncompressed size               4 bytes
        file name length                2 bytes
        extra field length              2 bytes

        file name (variable size)
        extra field (variable size)
```

根据上面的格式定义将对应的数据列举出来:

| 地址       | 长度                           | 内容                                    | 值                                                           | 小端序实际值                       |
| ---------- | ------------------------------ | --------------------------------------- | ------------------------------------------------------------ | ---------------------------------- |
| 0x00000000 | 4 bytes                        | local file header signature(0x04034b50) | 0x504B0304                                                   | 0x04034B50                         |
| 0x00000004 | 2 bytes                        | version needed to extract               | 0x0000                                                       | 0x0000                             |
| 0x00000006 | 2 bytes                        | general purpose bit flag                | 0x0000                                                       | 0x0000                             |
| 0x00000008 | 2 bytes                        | compression method                      | 0x0800                                                       | 0x0008                             |
| 0x0000000A | 2 bytes                        | last mod file time                      | 0x0000                                                       | 0x0000                             |
| 0x0000000C | 2 bytes                        | last mod file date                      | 0x0000                                                       | 0x0000                             |
| 0x0000001E | 4 bytes                        | crc-32                                  | 0x39B6CD57                                                   | 0x57CDB639                         |
| 0x00000012 | 4 bytes                        | compressed size                         | 0x12030000                                                   | 0x00000312                         |
| 0x00000016 | 4 bytes                        | uncompressed size                       | 0x98080000                                                   | 0x00000898                         |
| 0x0000001A | 2 bytes                        | file name length                        | 0x1300                                                       | 0x0013                             |
| 0x0000001C | 2 bytes                        | extra field length                      | 0x0000                                                       | 0x0000                             |
| 0x0000001E | variable size<br/>(0x0013==19) | file name                               | 0x41 0x6E 0x64 0x72 0x6F 0x69 0x64 0x4D 0x61 0x6E 0x69 0x66 0x65 0x73 0x74 0x2E 0x78 0x6D 0x6C | ASCII码的值为：AndroidManifest.xml |
| -          | variable size(0)               | extra field                             | (空)                                                         | (空)                               |

Local file header后面跟着的就是压缩后的文件数据，这块我们就不再深入了解了。从上面的解析我们可以了解到，zip包的解析其实是从后往前的。

# V2签名原理

了解完zip包的格式之后，就很容易理解V2签名的原理了。V2签名实际上是在apk的[central directory]前面插入一个apk签名块:

{% img /Android签名与渠道包制作V2V3签名原理/5.png %}

也就是说在eocd读取[offset of start of central directory with respect to the starting disk number]这个地址往前读就是APK签名块了。

我们来看看这个APK签名块的格式:

{% img /Android签名与渠道包制作V2V3签名原理/6.png %}

由于是往前读，所以结尾16字节是一个用于识别的魔数(字符串"APK Sig Block 42"),再往前是签名块的长度,继续往前是一系列的带长度前缀的id-value键值对,最前面又是签名块的长度。

我们直接找一个V2签名的apk来分析下:

{% img /Android签名与渠道包制作V2V3签名原理/7.png %}

同样先找到central directory的地址偏移0x00142174:

{% img /Android签名与渠道包制作V2V3签名原理/8.png %}

同样在该地址可以看到0x02014B50这个Central directory的魔数，而往前的16个字节就是字符串”APK Sig Block 42“的ASCII码。继续往前的8个字节则是APK签名块的长度0xFF8。我们用于是我们可以计算出第一个部分的地址:

```
0x00142174 - 0xFF8 - 0x8 = 0x00141174
```

再减去8个字节是因为APK签名块长度不包括第一个部分自身的8个字节。然后我们找到这个地址可以看到值是0x00000000 00000FF8:

{% img /Android签名与渠道包制作V2V3签名原理/9.png %}

根据APK签名块的格式我们知道往后便是第一个id-value键值对。他的长度是0x00000000 0000005F3,而id是0x7109871A。这个id的键值对被命名为"APK 签名方案 v2 分块"，里面保存的就是签名的校验数据。

## 摘要计算

校验数据的话首先要考虑的就是摘要算法，例如V1版本将每个原始文件用sha算法算出摘要之后用MANIFEST.MF一个个保存起来。而V2版本考虑了整个apk的校验，所以它并不去计算每个原始文件的摘要，而是计算整个apk的摘要。

为了加速运算，首先将apk按1m大小分割成若干块，分别计算这些块的摘要，再将这些摘要组合起来计算一次摘要，就得到了整个apk的摘要。并将其放入id为0x7109871A的"APK 签名方案 v2 分块"中:

{% img /Android签名与渠道包制作V2V3签名原理/10.png %}

光讲和看图可能理解还不是特别深入，我们直接干[ApkSignerV2](https://android.googlesource.com/platform/build/+/dd910c5/tools/signapk/src/com/android/signapk/ApkSignerV2.java)的源码:

```java
private static final int CONTENT_DIGESTED_CHUNK_MAX_SIZE_BYTES = 1024 * 1024;

private static Map<Integer, byte[]> computeContentDigests(Set<Integer> digestAlgorithms, ByteBuffer[] contents) throws DigestException {
  // 按1M大小分块,计算分块数量
  int chunkCount = 0;
  for (ByteBuffer input : contents) {
      chunkCount += getChunkCount(input.remaining(), CONTENT_DIGESTED_CHUNK_MAX_SIZE_BYTES);
  }

  // 可能使用多种算法进行摘要计算
  // 每种算法都会计算所有分块的摘要然后组合起来,再计算一次摘要
  // 这里先创建用于组合的buffer
  final Map<Integer, byte[]> digestsOfChunks = new HashMap<>(digestAlgorithms.size());
  for (int digestAlgorithm : digestAlgorithms) {
      // 获取摘要算法计算结果的大小
      int digestOutputSizeBytes = getContentDigestAlgorithmOutputSizeBytes(digestAlgorithm);
      // 前5个字节是0x5a和4个字节的块数量,后面是各个块的摘要直接连接组合
      byte[] concatenationOfChunkCountAndChunkDigests = new byte[5 + chunkCount * digestOutputSizeBytes];
      // 设置第0个字节为0x5a
      concatenationOfChunkCountAndChunkDigests[0] = 0x5a;
      // 设置第1个字节开始的四个字节为块数量
      setUnsignedInt32LittleEngian(chunkCount, concatenationOfChunkCountAndChunkDigests, 1);
      // 将buffer放入map中
      digestsOfChunks.put(digestAlgorithm, concatenationOfChunkCountAndChunkDigests);
  }

  // 各个分块的摘要计算也是类似的
  // 需要在摘要前面添加五个字节: 0x5a + 块长度
  int chunkIndex = 0;
  byte[] chunkContentPrefix = new byte[5];
  chunkContentPrefix[0] = (byte) 0xa5;

  for (ByteBuffer input : contents) {
      while (input.hasRemaining()) {
          // 读取分块
          int chunkSize = Math.min(input.remaining(), CONTENT_DIGESTED_CHUNK_MAX_SIZE_BYTES);
          final ByteBuffer chunk = getByteBuffer(input, chunkSize);

          // 使用各种算法计算分块的摘要
          for (int digestAlgorithm : digestAlgorithms) {
              //创建摘要算法实例
              String jcaAlgorithmName =
                      getContentDigestAlgorithmJcaDigestAlgorithm(digestAlgorithm);
              MessageDigest md;
              try {
                  md = MessageDigest.getInstance(jcaAlgorithmName);
              } catch (NoSuchAlgorithmException e) {
                  throw new DigestException(
                          jcaAlgorithmName + " MessageDigest not supported", e);
              }
              // 这个clear并不会将内容清空,仅仅只是是将内部的指针回到position 0
              chunk.clear();

              //在0x5a后面放入块的大小
              setUnsignedInt32LittleEngian(chunk.remaining(), chunkContentPrefix, 1);

              //计算块的摘要
              md.update(chunkContentPrefix);
              md.update(chunk);

              // 将计算到的分块摘要放入前面为每种算法创建的buffer中组合起来
              byte[] concatenationOfChunkCountAndChunkDigests =
                      digestsOfChunks.get(digestAlgorithm);
              int expectedDigestSizeBytes =
                      getContentDigestAlgorithmOutputSizeBytes(digestAlgorithm);
              int actualDigestSizeBytes =
                      md.digest(
                              concatenationOfChunkCountAndChunkDigests,
                              5 + chunkIndex * expectedDigestSizeBytes,
                              expectedDigestSizeBytes);
              if (actualDigestSizeBytes != expectedDigestSizeBytes) {
                  throw new DigestException(
                          "Unexpected output size of " + md.getAlgorithm()
                                  + " digest: " + actualDigestSizeBytes);
              }
          }
          chunkIndex++;
      }
  }

  // 遍历算法,计算分块摘要组合起来之后的总摘要
  Map<Integer, byte[]> result = new HashMap<>(digestAlgorithms.size());
  for (Map.Entry<Integer, byte[]> entry : digestsOfChunks.entrySet()) {
      int digestAlgorithm = entry.getKey();
      byte[] concatenationOfChunkCountAndChunkDigests = entry.getValue();
      String jcaAlgorithmName = getContentDigestAlgorithmJcaDigestAlgorithm(digestAlgorithm);
      MessageDigest md;
      try {
          md = MessageDigest.getInstance(jcaAlgorithmName);
      } catch (NoSuchAlgorithmException e) {
          throw new DigestException(jcaAlgorithmName + " MessageDigest not supported", e);
      }
      result.put(digestAlgorithm, md.digest(concatenationOfChunkCountAndChunkDigests));
  }
  return result;
}
```

可以从源码看到计算的流程大概有三步:

1. 将整个apk按1M大小分块
2. 用多个摘要算法去计算 "0x5a + 分块长度 + 分块内容" 的摘要
3. 用多个摘要算法计算 "0x5a + 分块数量 + 各个分块摘要" 的总摘要

虽然在签名的时候没有使用并行计算，但是实际上各个分块的摘要是独立的，在需要的时候完全可以使用并发计算去加速优化。

## 摘要签名

为了防止攻击者在修改apk之后同步修改摘要，V2签名还会使用签名私钥对上面计算出来的摘要进行签名:

```java
private static byte[] generateSignerBlock(
      SignerConfig signerConfig,
      Map<Integer, byte[]> contentDigests) throws InvalidKeyException, SignatureException {
  if (signerConfig.certificates.isEmpty()) {
      throw new SignatureException("No certificates configured for signer");
  }
  // 先将公钥保存下来用于
  // 1. 签名之后的验证
  // 2. 写入"APK 签名方案 v2 分块"用于安装时候验证签名
  PublicKey publicKey = signerConfig.certificates.get(0).getPublicKey();
  byte[] encodedPublicKey = encodePublicKey(publicKey);

  // 初始化签名数据
  // 主要是创建<摘要算法id,apk摘要>键值对的列表
  V2SignatureSchemeBlock.SignedData signedData = new V2SignatureSchemeBlock.SignedData();
  try {
      signedData.certificates = encodeCertificates(signerConfig.certificates);
  } catch (CertificateEncodingException e) {
      throw new SignatureException("Failed to encode certificates", e);
  }
  List<Pair<Integer, byte[]>> digests =
          new ArrayList<>(signerConfig.signatureAlgorithms.size());
  for (int signatureAlgorithm : signerConfig.signatureAlgorithms) {
      int contentDigestAlgorithm =
              getSignatureAlgorithmContentDigestAlgorithm(signatureAlgorithm);
      byte[] contentDigest = contentDigests.get(contentDigestAlgorithm);
      if (contentDigest == null) {
          throw new RuntimeException(
                  getContentDigestAlgorithmJcaDigestAlgorithm(contentDigestAlgorithm)
                  + " content digest for "
                  + getSignatureAlgorithmJcaSignatureAlgorithm(signatureAlgorithm)
                  + " not computed");
      }
      digests.add(Pair.create(signatureAlgorithm, contentDigest));
  }
  signedData.digests = digests;

  // 将上面得到的signedData放入signer中用于计算签名
  V2SignatureSchemeBlock.Signer signer = new V2SignatureSchemeBlock.Signer();
  // FORMAT:
  // * length-prefixed sequence of length-prefixed digests:
  //   * uint32: signature algorithm ID
  //   * length-prefixed bytes: digest of contents
  // * length-prefixed sequence of certificates:
  //   * length-prefixed bytes: X.509 certificate (ASN.1 DER encoded).
  // * length-prefixed sequence of length-prefixed additional attributes:
  //   * uint32: ID
  //   * (length - 4) bytes: value
  signer.signedData = encodeAsSequenceOfLengthPrefixedElements(new byte[][] {
      encodeAsSequenceOfLengthPrefixedPairsOfIntAndLengthPrefixedBytes(signedData.digests),
      encodeAsSequenceOfLengthPrefixedElements(signedData.certificates),
      // additional attributes
      new byte[0],
  });

  // 保存公钥
  signer.publicKey = encodedPublicKey;

  // 计算各个摘要算法获取的摘要的签名
  signer.signatures = new ArrayList<>();
  for (int signatureAlgorithm : signerConfig.signatureAlgorithms) {
      Pair<String, ? extends AlgorithmParameterSpec> signatureParams =
              getSignatureAlgorithmJcaSignatureAlgorithm(signatureAlgorithm);
      String jcaSignatureAlgorithm = signatureParams.getFirst();
      AlgorithmParameterSpec jcaSignatureAlgorithmParams = signatureParams.getSecond();
      byte[] signatureBytes;

      // 获取签名算法使用私钥进行签名
      try {
          Signature signature = Signature.getInstance(jcaSignatureAlgorithm);
          signature.initSign(signerConfig.privateKey);
          if (jcaSignatureAlgorithmParams != null) {
              signature.setParameter(jcaSignatureAlgorithmParams);
          }
          signature.update(signer.signedData);
          signatureBytes = signature.sign();
      } catch (InvalidKeyException e) {
          throw new InvalidKeyException("Failed sign using " + jcaSignatureAlgorithm, e);
      } catch (NoSuchAlgorithmException | InvalidAlgorithmParameterException
              | SignatureException e) {
          throw new SignatureException("Failed sign using " + jcaSignatureAlgorithm, e);
      }

      // 使用公钥尝试是否能够正确验证签名
      try {
          Signature signature = Signature.getInstance(jcaSignatureAlgorithm);
          signature.initVerify(publicKey);
          if (jcaSignatureAlgorithmParams != null) {
              signature.setParameter(jcaSignatureAlgorithmParams);
          }
          signature.update(signer.signedData);
          if (!signature.verify(signatureBytes)) {
              throw new SignatureException("Signature did not verify");
          }
      } catch (InvalidKeyException e) {
          throw new InvalidKeyException("Failed to verify generated " + jcaSignatureAlgorithm
                  + " signature using public key from certificate", e);
      } catch (NoSuchAlgorithmException | InvalidAlgorithmParameterException
              | SignatureException e) {
          throw new SignatureException("Failed to verify generated " + jcaSignatureAlgorithm
                  + " signature using public key from certificate", e);
      }

      // 将签名加入签名数据
      signer.signatures.add(Pair.create(signatureAlgorithm, signatureBytes));
  }

  // 生成签名二进制数据
  // FORMAT:
  // * length-prefixed signed data
  // * length-prefixed sequence of length-prefixed signatures:
  //   * uint32: signature algorithm ID
  //   * length-prefixed bytes: signature of signed data
  // * length-prefixed bytes: public key (X.509 SubjectPublicKeyInfo, ASN.1 DER encoded)
  return encodeAsSequenceOfLengthPrefixedElements(
          new byte[][] {
              signer.signedData,
              encodeAsSequenceOfLengthPrefixedPairsOfIntAndLengthPrefixedBytes(
                      signer.signatures),
              signer.publicKey,
          });
}
```

最后会将计算得到的摘要、摘要签名、公钥、算法信息等数据写入刚刚说的的id为0x7109871A的"APK 签名方案 v2 分块"中，于是在安装apk的时候就能使用这些数据去检查apk是否被修改了:

{% img /Android签名与渠道包制作V2V3签名原理/11.png %}

## 防回滚保护

由于需要在Android 7.0之后才支持V2版本的签名，为了兼容低版本的安卓机器，一般情况下我们会同时使用V1和V2版本的签名。但由于V2版本插入apk中间的"APK签名块"是独立于zip格式存在的，攻击者其实可以直接将其直接删掉，使得apk降级回V1。

而高版本的安卓系统为了兼容旧的apk，也会在找不到Apk签名块的情况下使用V1签名去验证。

谷歌为了防止这种恶意操作规定:

同时包含V1和V2签名的CERT.SF文件会加入这样一个属性:

```
X-Android-APK-Signed: 2
```

在Android 7.0之后读取到这个属性的时候就会强制使用V2版本的签名检查机制而不走V1版本的。

# V3签名原理

由于生成签名的时，可以指定一个有效时间，这个时间默认为 25 年，如果过了这个时间可能会出现签名失效不能再安装的情况。

说可能是因为网上[有人](https://blog.fengsq.com/post/ApkSignature.html)实际验证过,有些机器是没有做这个检查的:

> ==但是，我实际测试了下官方模拟器、小米、vivo、华为荣耀，签名已失效依然可以正常安装。== 网上千篇一律都说失效签名无法安装，不知道他们有没有实际测过。咨询了厂商的开发者，目前只收到了vivo的回复，说是因为手机时间可以随意调，所以这个检验没有任何意义，他们废弃掉了，其他厂商不知道是不是也出于这个原因。

但是为了防止的确有公司被收购等这样那样的原因需要更换签名，安卓9.0之后提供了V3版本的签名机制。

V3版本的机制原理是在APK签名块里面新增了一个id为0xF05368C0的键值对，它的格式也和V2版本id为0x7109871A的"APK 签名方案 v2 分块"基本相同，只不过增加了attr块，里面保存了多个level的证书信息。(由于它们的id不一样，所以在V2+V3同时签名的情况下，APK签名块会同时有这两个id的键值对)

我从这位博主的[文章](https://blog.csdn.net/bobby_fu/article/details/103843038)中看到了这附两幅图，能够很形象的解释V2和V3签名间的差异:

{% img /Android签名与渠道包制作V2V3签名原理/12.png %}

在安装的时候会使用旧的证书去验证新证书是否有效。如果当前已经安装的apk的证书在level证书链上，就能逐步完后验证更新的证书的有效性

{% img /Android签名与渠道包制作V2V3签名原理/13.png %}

 证书链验证的核心代码如下：

```java
// frameworks/base/core/java/android/util/apk/ApkSignatureSchemeV3Verifier.java
private static VerifiedProofOfRotation verifyProofOfRotationStruct(
        ByteBuffer porBuf,
        CertificateFactory certFactory)
        throws SecurityException, IOException {
    int levelCount = 0;
    int lastSigAlgorithm = -1;
    X509Certificate lastCert = null;
    List<X509Certificate> certs = new ArrayList<>();
    List<Integer> flagsList = new ArrayList<>();

    // Proof-of-rotation struct:
    // A uint32 version code followed by basically a singly linked list of nodes, called levels
    // here, each of which have the following structure:
    // * length-prefix for the entire level
    //     - length-prefixed signed data (if previous level exists)
    //         * length-prefixed X509 Certificate
    //         * uint32 signature algorithm ID describing how this signed data was signed
    //     - uint32 flags describing how to treat the cert contained in this level
    //     - uint32 signature algorithm ID to use to verify the signature of the next level. The
    //         algorithm here must match the one in the signed data section of the next level.
    //     - length-prefixed signature over the signed data in this level.  The signature here
    //         is verified using the certificate from the previous level.
    // The linking is provided by the certificate of each level signing the one of the next.

    try {

        // get the version code, but don't do anything with it: creator knew about all our flags
        porBuf.getInt();
        HashSet<X509Certificate> certHistorySet = new HashSet<>();
        while (porBuf.hasRemaining()) {
            levelCount++;
            ByteBuffer level = getLengthPrefixedSlice(porBuf);
            ByteBuffer signedData = getLengthPrefixedSlice(level); // 获取当前level证书的信息
            int flags = level.getInt();
            int sigAlgorithm = level.getInt();
            byte[] signature = readLengthPrefixedByteArray(level); // 获取上一level证书为当前level证书生成的签名

            // 使用上一个level的证书去验证下一个level的证书
            if (lastCert != null) {
                // 获取上一个证书的数据
                Pair<String, ? extends AlgorithmParameterSpec> sigAlgParams =
                        getSignatureAlgorithmJcaSignatureAlgorithm(lastSigAlgorithm);
                // 获取上一个证书的公钥
                PublicKey publicKey = lastCert.getPublicKey();
                // 初始化签名信息
                Signature sig = Signature.getInstance(sigAlgParams.first);
                sig.initVerify(publicKey);
                if (sigAlgParams.second != null) {
                    sig.setParameter(sigAlgParams.second);
                }
                // 设置当前level证书的数据
                sig.update(signedData);
                // 使用上一level证书为当前level证书生成的签名去验证当前level证书是否有效
                if (!sig.verify(signature)) {
                    throw new SecurityException("Unable to verify signature of certificate #"
                            + levelCount + " using " + sigAlgParams.first + " when verifying"
                            + " Proof-of-rotation record");
                }
            }
                        
            // 使用证书信息去创建证书，将其赋值给lastCert并将其丢入certs队列
            signedData.rewind();
            byte[] encodedCert = readLengthPrefixedByteArray(signedData);
            int signedSigAlgorithm = signedData.getInt();
            if (lastCert != null && lastSigAlgorithm != signedSigAlgorithm) {
                throw new SecurityException("Signing algorithm ID mismatch for certificate #"
                        + levelCount + " when verifying Proof-of-rotation record");
            }
            lastCert = (X509Certificate)
                    certFactory.generateCertificate(new ByteArrayInputStream(encodedCert));
            lastCert = new VerbatimX509Certificate(lastCert, encodedCert);

            lastSigAlgorithm = sigAlgorithm;
            if (certHistorySet.contains(lastCert)) {
                throw new SecurityException("Encountered duplicate entries in "
                        + "Proof-of-rotation record at certificate #" + levelCount + ".  All "
                        + "signing certificates should be unique");
            }
            certHistorySet.add(lastCert);
            certs.add(lastCert);
            flagsList.add(flags);
        }
    } catch (IOException | BufferUnderflowException e) {
        throw new IOException("Failed to parse Proof-of-rotation record", e);
    } catch (NoSuchAlgorithmException | InvalidKeyException
            | InvalidAlgorithmParameterException | SignatureException e) {
        throw new SecurityException(
                "Failed to verify signature over signed data for certificate #"
                        + levelCount + " when verifying Proof-of-rotation record", e);
    } catch (CertificateException e) {
        throw new SecurityException("Failed to decode certificate #" + levelCount
                + " when verifying Proof-of-rotation record", e);
    }
    return new VerifiedProofOfRotation(certs, flagsList);
}

```

## V3版本校验流程

实际上校验的时候并不需要从证书链中解析出最后的公钥，因为和V2的格式一样，直接可以在签名块中读取到公钥进行校验。所以他的流程前面的部分其实和v2版本是一致的，只不过在校验完成之后会再去验证证书链:

1. 用PublicKey和Signature验证SignerData
2. 用SignerData验证apk
3. 验证当前安装的应用证书是否在证书链中
4. 继续安装

而证书链最新的证书公钥其实就是APK签名块里的PublicKey:

```java
private static VerifiedSigner verifyAdditionalAttributes(ByteBuffer attrs,
        List<X509Certificate> certs, CertificateFactory certFactory) throws IOException {
    X509Certificate[] certChain = certs.toArray(new X509Certificate[certs.size()]);
    VerifiedProofOfRotation por = null;

    while (attrs.hasRemaining()) {
        ByteBuffer attr = getLengthPrefixedSlice(attrs);
        if (attr.remaining() < 4) {
            throw new IOException("Remaining buffer too short to contain additional attribute "
                    + "ID. Remaining: " + attr.remaining());
        }
        int id = attr.getInt();
        switch(id) {
            case PROOF_OF_ROTATION_ATTR_ID:
                if (por != null) {
                    throw new SecurityException("Encountered multiple Proof-of-rotation records"
                            + " when verifying APK Signature Scheme v3 signature");
                }
                por = verifyProofOfRotationStruct(attr, certFactory);
                // 确认证书链最后一个证书的公钥与APK签名块的公钥相等
                try {
                    if (por.certs.size() > 0
                            && !Arrays.equals(por.certs.get(por.certs.size() - 1).getEncoded(),
                                    certChain[0].getEncoded())) {
                        throw new SecurityException("Terminal certificate in Proof-of-rotation"
                                + " record does not match APK signing certificate");
                    }
                } catch (CertificateEncodingException e) {
                    throw new SecurityException("Failed to encode certificate when comparing"
                            + " Proof-of-rotation record and signing certificate", e);
                }

                break;
            default:
                // not the droid we're looking for, move along, move along.
                break;
        }
    }
    return new VerifiedSigner(certChain, por);
}
```

最终用[最新的证书的公钥]+[摘要的签名]去验证[摘要]的有效性，从而验证apk的有效性:

{% img /Android签名与渠道包制作V2V3签名原理/14.png %}

这篇讲述了V2、V3签名机制的原理，由于章节已经很长了，渠道包的制作就放到下一篇。

参考:

[VasDolly实现原理](https://github.com/Tencent/VasDolly/wiki/VasDolly%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86)

[Android V2签名机制以及ApkSignerV2签名源码解析](https://www.jianshu.com/p/dc320629bf9d)

[分析 Android V2 新签名打包机制](https://cloud.tencent.com/developer/article/1006237)

[Android P v3签名新特性](https://blog.csdn.net/bobby_fu/article/details/103843038)

[一次让你搞懂Android应用签名](https://blog.fengsq.com/post/ApkSignature.html)