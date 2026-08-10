---
layout: fragment
title: 双向认证问题排查记录
tags: [tls, ssl, mtls]
description: 在车联网场景中，客户端和服务端互联时，为了提升整个系统的安全，需要用上常用的数据传输层的加密策略.
keywords: 双向认证, https, mtls
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---


## 尝试建立连接
openssl s_client -connect robotics-map-me-cloud.geely-test.com:8883 -servername robotics-map-me-cloud.geely-test.com


## 尝试携带客户端证书和私钥与服务端建立连接
openssl s_client -connect robotics-map-me-cloud.geely-test.com:8883 -servername robotics-map-me-cloud.geely-test.com -cert EX1H_EU_PROD.pem -key EX1H_EU_PROD.key


## 验证某个证书是否由某个CA签发
openssl verify -CAfile ZK ECU Issuing EU-CA.cer EX1H_EU_PROD.pem

## 连接服务端，使用自签的根CA来校验服务端
openssl s_client -connect robotics-map-me-cloud.geely-test.com:8883 \
  -servername robotics-map-me-cloud.geely-test.com \
  -CAfile server.crt

## 验证证书颁发机构和CA是否对应
openssl verify -CAfile CA_Bundle.pem EX1H_EU_PROD.pem

## 验证双向认证
 openssl s_client -connect robotics-map-me-cloud.geely-test.com:8883 \
    -servername robotics-map-me-cloud.geely-test.com \
    -cert EX1H_EU_PROD.pem \
    -key EX1H_EU_PROD.key \
    -CAfile server.pem

## 中东测试
openssl s_client -connect robotics-map-me-cloud.geely-test.com:8883 \
  -servername robotics-map-me-cloud.geely-test.com \
  -cert full_client_chain.pem \
  -key EX1H_EU_PROD.key \
  -CAfile server.me.test.pem

 openssl s_client -connect robotics-map-me-cloud.geely-test.com:8883 \
    -servername robotics-map-me-cloud.geely-test.com \
    -cert full_client_chain.pem \
    -key EX1H_EU_PROD.key \
    -CAfile root-ca-me-test.pem

openssl s_client -connect robotics-map-me-cloud.geely-test.com:8883 \
  -servername robotics-map-me-cloud.geely-test.com \
  -cert EX1H_EU_PROD.pem \
  -key EX1H_EU_PROD.key \
  -CAfile root-ca-me-test.pem

openssl s_client -connect robotics-map-me-cloud.geely-test.com:443 \
  -servername robotics-map-me-cloud.geely-test.com \
  -cert EX1H_EU_PROD.pem \
  -key EX1H_EU_PROD.key \
  -CAfile root-ca-me-test.pem

## 欧洲测试
openssl s_client -connect robotics-map-eu-cloud.geely-test.com:8883 \
  -servername robotics-map-eu-cloud.geely-test.com \
  -cert full_client_chain.pem \
  -key EX1H_EU_PROD.key \
  -CAfile server.eu.test.pem

openssl s_client -connect robotics-map-eu-cloud.geely-test.com:443 \
    -servername robotics-map-eu-cloud.geely-test.com \
    -cert EX1H_EU_PROD.pem \
    -key EX1H_EU_PROD.key \
    -CAfile root-ca-eu-test.pem

openssl s_client -connect robotics-map-eu-cloud.geely-test.com:8883 \
    -servername robotics-map-eu-cloud.geely-test.com \
    -cert EX1H_EU_PROD.pem \
    -key EX1H_EU_PROD.key \
    -CAfile root-ca-eu-test.pem

## 中东生产

 openssl s_client -connect robotics-map-me-cloud.geely.com:8883 \
    -servername robotics-map-me-cloud.geely.com \
    -cert full_client_chain.pem \
    -key EX1H_EU_PROD.key \
    -CAfile server.eu.prod.pem

openssl s_client -connect robotics-map-me-cloud.geely.com:8883 \
  -servername robotics-map-me-cloud.geely.com \
  -cert EX1H_EU_PROD.pem \
  -key EX1H_EU_PROD.key \
  -CAfile root-ca-me-prod.pem


openssl s_client -connect robotics-map-me-cloud.geely.com:443 \
  -servername robotics-map-me-cloud.geely.com \
  -cert EX1H_EU_PROD.pem \
  -key EX1H_EU_PROD.key \
  -CAfile root-ca-me-prod.pem


## 查看证书的签发机构
openssl x509 -in server-cert.pem -noout -issuer


## 保存服务端返回的证书

 openssl s_client -connect robotics-map-eu-cloud.geely-test.com:8883 \
      -servername robotics-map-eu-cloud.geely-test.com -showcerts 2>/dev/null \
      | awk '/BEGIN CERT/,/END CERT/' > server_returned.pem

 openssl s_client -connect robotics-map-me-cloud.geely-test.com:8883 \
      -servername robotics-map-me-cloud.geely-test.com -showcerts 2>/dev/null \
      | awk '/BEGIN CERT/,/END CERT/' > server_returned_me_test.pem


 openssl s_client -connect robotics-map-me-cloud.geely.com:8883 \
      -servername robotics-map-me-cloud.geely.com -showcerts 2>/dev/null \
      | awk '/BEGIN CERT/,/END CERT/' > server_returned_me_prod.pem
