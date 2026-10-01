> 未完成撰写的文档，因为版本迭代过快，跟新版本会存在一定差异，后续会进行补充完善。

## 前提条件
- 已安装 [moviereview](https://github.com/apache/dubbo-kubernetes/tree/master/samples/moviereview) 服务
- `retry` 字段在 Gateway API v1.4.1 中属于 Extended/Experimental 字段。集群必须安装 experimental CRD，standard CRD 会删除或拒绝该字段：
  ```bash
  kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.1/experimental-install.yaml
  ```

## 重试
在连接失败、重置以及 HTTP `500`、`502`、`503`、`504` 时最多重试 3 次。每次上游调用最多等待 1 秒，初次请求和全部重试合计不能超过 5 秒；第一次重试至少等待 100 毫秒，后续使用指数退避，最大为 1 秒。

```bash
cat <<EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: reviews-retry
  namespace: moviereview
spec:
  parentRefs:
  - group: ""
    kind: Service
    name: reviews
    port: 9080
  rules:
  - retry:
      attempts: 3
      codes:
      - 500
      - 502
      - 503
      - 504
      backoff: 100ms
    timeouts:
      request: 5s
      backendRequest: 1s
    backendRefs:
    - name: reviews-v1
      port: 9080
      weight: 50
    - name: reviews-v2
      port: 9080
      weight: 50
EOF
```

## 查看资源信息

```bash
kubectl get httproute -n moviereview
```

TODO 过时内容
## 验证

先把 dubbod 的测试入口转发到本地：

```shell
kubectl -n dubbo-system port-forward deploy/dubbod 17171:17171
```

从另一个终端发送真实请求：

```shell
grpcurl -plaintext \
  -d '{"url":"xds:///reviews.moviereview.svc.cluster.local:9080","path":"/reviews","count":20}' \
  :17171 proto.XDSTestService/ForwardHTTP | jq -r '.output | join("")'
```

正常情况下输出会来自 `reviews-v1` 和 `reviews-v2`。验证重试时，让一个测试后端返回配置中的 `503`，或让首选测试端点拒绝连接；Inherent outbound 客户端会在总请求超时内选择下一个端点。把返回码改成未配置的 `501` 时，不会触发状态码重试。

## 清理资源

```bash
kubectl -n moviereview delete httproute reviews-retry
```
