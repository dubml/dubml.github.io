> 未完成撰写的文档，因为版本迭代过快，跟新版本会存在一定差异，后续会进行补充完善。

## 前提条件
- 已安装 [moviereview](https://github.com/apache/dubbo-kubernetes/tree/master/samples/moviereview) 服务

## 切分流量

将 `reviews` 的 63% 流量分配到 `v1`，37% 流量分配到 `v2`：

```bash
cat <<EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: reviews-canary
  namespace: moviereview
spec:
  parentRefs:
  - group: ""
    kind: Service
    name: reviews
    port: 9080
  rules:
  - backendRefs:
    - name: reviews-v1
      port: 9080
      weight: 63
    - name: reviews-v2
      port: 9080
      weight: 37
EOF
```

## 查看资源信息

```bash
kubectl get httproute -n moviereview
```

TODO 过时内容
## 验证

先把 17171 转发到本地：

```bash
kubectl -n dubbo-system port-forward deploy/dubbod 17171:17171
```

确认真实请求进入 `reviews v1` 和 `reviews v2`：

```bash
grpcurl -plaintext \
  -d '{"url":"xds:///reviews.moviereview.svc.cluster.local:9080","count":5}' \
  :17171 proto.XDSTestService/ForwardHTTP | jq -r '.output | join("")'
```

预期会连续打印真实响应，例如：

```text
reviews v1
reviews v2
reviews v1
reviews v1
reviews v2
```

也可以刷新 `frontend` 页面验证。页面里的 moviepage 会通过本地 `reviews-grpc-outbound` 调用 `reviews`，连续刷新时版本应只在 `v1` 和 `v2` 之间变化。

如果要观察热更新，再指定请求次数和间隔；这只是采样参数，不是路由配置。先在一个终端持续发请求，另一个终端里更新 `HTTPRoute` 权重。同一个 `ForwardHTTP` 请求会继续使用同一条 xDS stream，后续请求会按新权重切换。

```bash
grpcurl -plaintext \
  -d '{"url":"xds:///reviews.moviereview.svc.cluster.local:9080","count":200,"requestInterval":"200ms"}' \
  :17171 proto.XDSTestService/ForwardHTTP | jq -r '.output | join("")'
```

把流量改成 `v1` 20%、`v2` 80%：

```bash
kubectl -n moviereview patch httproute reviews-canary --type='merge' -p '
{
  "spec": {
    "rules": [
      {
        "backendRefs": [
          {
            "name": "reviews-v1",
            "port": 9080,
            "weight": 20
          },
          {
            "name": "reviews-v2",
            "port": 9080,
            "weight": 80
          }
        ]
      }
    ]
  }
}'
```

## 清理资源

```bash
kubectl -n moviereview delete httproute reviews-canary
```
