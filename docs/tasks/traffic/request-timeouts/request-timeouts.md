> 未完成撰写的文档，因为版本迭代过快，跟新版本会存在一定差异，后续会进行补充完善。

## 前提条件
- 已安装 [moviereview](https://github.com/apache/dubbo-kubernetes/tree/master/samples/moviereview) 服务

## 超时

把 `reviews` 请求转发到 `reviews-v2`，并把请求超时设置为 `500ms`：

```bash
cat <<EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: reviews-timeout
  namespace: moviereview
spec:
  parentRefs:
  - group: ""
    kind: Service
    name: reviews
    port: 9080
  rules:
  - timeouts:
      request: 500ms
    backendRefs:
    - name: reviews-v2
      port: 9080
EOF
```

## 查看资源信息

```bash
kubectl get httproute -n moviereview
```

TODO 过时内容
## 验证

先把 `dubbod` 的测试入口转发到本地：

```bash
kubectl -n dubbo-system port-forward deploy/dubbod 17171:17171
```

正常超时时间下，请求应该返回 `reviews-v2`：

```bash
grpcurl -plaintext \
  -d '{"url":"xds:///reviews.moviereview.svc.cluster.local:9080","path":"/reviews"}' \
  :17171 proto.XDSTestService/ForwardHTTP | jq -r '.output | join("")'
```

预期输出包含 `v2`：

```text
reviews v2
```

把超时临时改成 `1ms`，验证请求会被取消：

```bash
kubectl -n moviereview patch httproute reviews-timeout --type='merge' -p '
{
  "spec": {
    "rules": [
      {
        "timeouts": {
          "request": "1ms"
        },
        "backendRefs": [
          {
            "name": "reviews-v2",
            "port": 9080
          }
        ]
      }
    ]
  }
}'
```

再次请求：

```bash
grpcurl -plaintext \
  -d '{"url":"xds:///reviews.moviereview.svc.cluster.local:9080","path":"/reviews"}' \
  :17171 proto.XDSTestService/ForwardHTTP
```

预期返回 `context deadline exceeded`：

```text
ERROR:
  Code: Unknown
  Message: Get "http://.../reviews": context deadline exceeded
```

恢复 `500ms` 后，请求应重新成功：

```bash
kubectl -n moviereview patch httproute reviews-timeout --type='merge' -p '
{
  "spec": {
    "rules": [
      {
        "timeouts": {
          "request": "500ms"
        },
        "backendRefs": [
          {
            "name": "reviews-v2",
            "port": 9080
          }
        ]
      }
    ]
  }
}'
```

## 清理资源

```bash
kubectl -n moviereview delete httproute reviews-timeout
```
