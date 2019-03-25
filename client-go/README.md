# client-go

# 代码结构 (删除不重要的内容)

/home/scott/go/src/k8s.io/client-go/
├── deprecated-dynamic 
├── discovery
├── dynamic
├── examples
├── informers
├── kubernetes
├── listers
├── pkg
├── plugin
├── rest
├── restmapper
├── scale
├── testing
├── third_party
├── tools
├── transport
├── util

## deprecated-dynamic (暂时看来用的比较少)

根据注释该包可为Kubernetes中的任意API对象提供常见的高级操作以及元数据。

## rest **核心库**

/home/scott/go/src/k8s.io/client-go/rest/
├── client.go
├── client_test.go
├── config.go
├── config_test.go
├── fake
│   └── fake.go
├── OWNERS
├── plugin.go
├── plugin_test.go
├── request.go
├── request_test.go
├── transport.go
├── urlbackoff.go
├── urlbackoff_test.go
├── url_utils.go
├── url_utils_test.go
├── watch
│   ├── decoder.go
│   ├── decoder_test.go
│   ├── encoder.go
│   └── encoder_test.go
└── zz_generated.deepcopy.go

### request.go

核心数据结构

```go
type Request struct {
	// required
	client HTTPClient // go 自带http.Client
	verb   string // "GET POST PUT DELETE"

	baseURL     *url.URL
	content     ContentConfig // protobuf || Json
	serializers Serializers // 序列化

	// generic components accessible via method setters
	pathPrefix string   
	subpath    string
	params     url.Values
	headers    http.Header

	// structural elements of the request that are part of the Kubernetes API conventions
	namespace    string
	namespaceSet bool
	resource     string
	resourceName string
	subresource  string
	timeout      time.Duration

	// output
	err  error
	body io.Reader

	// This is only used for per-request timeouts, deadlines, and cancellations.
	ctx context.Context

	backoffMgr BackoffManager
	throttle   flowcontrol.RateLimiter
}

```


## discovery

可以通过该接口，去查看当前kube-apiserver提供的 API组，以及资源列表
