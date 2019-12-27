# TKE Ingress Controller 证书管理

## 证书相关Annotation
* kubernetes.io/ingress.http-rules
* kubernetes.io/ingress.https-rules
* kubernetes.io/ingress.rule-mix
* qcloud_cert_id（只读）

## Kubectl操作指南

### 配置证书并创建一个Https服务
* 创建Secret资源，配置证书ID为 `XczRzegn`（注意进行Base64编码）

```yaml
apiVersion: v1
data:
  qcloud_cert_id: WGN6UnplZ24=
kind: Secret
metadata:
  name: tencent-com-cert
  namespace: default
type: Opaque
```

* 或者创建时使用`stringData`进行声明，避免手工进行Base64编码

```yaml
apiVersion: v1
stringData:
  qcloud_cert_id: XczRzegn
kind: Secret
metadata:
  name: tencent-com-cert
  namespace: default
type: Opaque
```

* 创建Ingress资源，指定后端Service:`sample-service`:`80`，指定secretName:`tencent-com-cert`

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: qcloud
    qcloud_cert_id: XczRzegn
  name: sample-ingress
  namespace: default
spec:
  rules:
  - http:
      paths:
      - backend:
          serviceName: sample-service
          servicePort: 80
        path: /
  tls:
  - secretName: tencent-com-cert
```

### 修改证书
* 修改Secret资源，将`qcloud_cert_id`的值设置更新为新的证书ID。
* 如同创建Secret证书资源时一样。注意证书ID需要进行Base64编码，或者修改时指定stringData避免手动进行Base64。

## 控制台操作指南

* 控制台创建的Ingress开启Https服务，会先创建同名的Secret资源用于存放证书ID，然后在Ingress中使用与监听这个Secret。
* 在控制台修改证书时，会修改当前Ingress对应的证书资源。注意如果用户多个Ingress配置使用同一个Secret资源，那么这些Ingress对应CLB的证书会一起变更。

## 混合规则配置
TKE Ingress Controller 支持混合配置Http/Https规则

1. `kubernetes.io/ingress.rule-mix`设置为True时开启混合规则。
    * 当Ingress模板未配置TLS时，没有提供证书资源。所有规则都将以Http服务暴露，该注解将不会生效。
2. 将Ingress中的每一条规则与`kubernetes.io/ingress.http-rules`、`kubernetes.io/ingress.https-rules`进行匹配并添加到对应规则集中，如果Ingress中的规则未匹配时，默认添加到Https规则集中。
3. 匹配时校验 Host、Path、ServiceName、ServicePort（Host默认为`VIP`、Path默认为`/`）
    * 注意IPv6的CLB没有提供默认域名的功能。（链接到IPv6相关）

### 示例
* 开启混合规则，同时配置后端服务暴露Http/Https服务。

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.http-rules: '[{"host":"www.tencent.com","path":"/","backend":{"serviceName":"sample-service","servicePort":"80"}}]'
    kubernetes.io/ingress.https-rules: '[{"host":"www.tencent.com","path":"/","backend":{"serviceName":"sample-service","servicePort":"80"}}]'
    kubernetes.io/ingress.rule-mix: "true"
    kubernetes.io/ingress.class: qcloud
    qcloud_cert_id: XczRzegn
  name: sample-ingress
  namespace: default
spec:
  rules:
  - host: www.tencent.com
    http:
      paths:
      - backend:
          serviceName: sample-service
          servicePort: 80
        path: /
  tls:
  - secretName: tencent-com-cert
```

## 注意事项
* Ingress Annotation中的`qcloud_cert_id`是只读的，便于快速了解当前Ingress对应的证书ID。
* Secret证书资源必须和Ingress资源放在同一个Namespace下
* 由于控制台默认会创建同名Secret证书资源，所以当Secret资源已存在时，Ingress将无法创建。
* 默认情况下，TKE Ingress不会复用Secret资源。但是Secret证书资源允许被复用于Ingress，必须注意更新Secret会使得所有Ingress的证书得到更新。

## FAQ
* 我是否能修改Ingress中的tls.secretName，指向另一个Secret资源。
    * 可以，新的Secret证书资源中指定的证书将很快得到同步到Ingress对应的CLB上。
* 我需要在哪里获取证书Id
    * CLB控制台-证书管理（https://console.cloud.tencent.com/clb/cert）

## 相关参考
* Kubenetes Secret参考：https://kubernetes.io/zh/docs/concepts/configuration/secret/
