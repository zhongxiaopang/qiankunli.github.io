---

layout: post
title: mosn细节
category: 技术
tags: Mesh
keywords: mosn detail

---

## 前言

* TOC
{:toc}

![](/public/upload/mesh/mosn_process.png)

## 多协议机制

![](/public/upload/mesh/mosn_protocol.png)

[MOSN 多协议机制解析](https://mosn.io/zh/blog/posts/multi-protocol-deep-dive/)

![](/public/upload/mesh/mosn_stream_layer.png)

### 多路复用

借鉴了http2 的stream 的理念（所以Stream interface 上有一个方法是`ID()`），Stream 是虚拟的，**在“连接”的层面上看，消息却是乱序收发的“帧”（http2 frame）**，通过StreamId关联，用来实现在一个Connection 之上的“多路复用”。tcp 数据包在网络上流转，os 维护了socket 对象，随着连接创建、关闭而新建和销毁。 frame 数据包在 连接中传输，网络 应用层维护了 stream 对象，随着 request-response 产生、结束而新建和销毁。 

![](/public/upload/network/network_buffer.png)

Stream 的概念并不新鲜，在微服务通信中，一般要为client request 分配一个requestId，并将requestId 暂存在 client cache中，当client 收到response 时， 从response 数据包中提取requestId，进而确定 response 是client cache中哪个request 的响应。**之前这只是一种“技巧” 或“惯例”，Http2将其 正式 称之为多路复用/Stream**。

![](/public/upload/mesh/rpc_network.png)

mosn http2 和 xprotocol 的StreamConnection 中都保存有 requestId 与 Stream 映射。当发现 frame 携带的 requestId 不存在时，则NewStream，否则读取Stream。然后拿着 Stream 对象 执行`stream.receiver.OnReceive`

以http2 和 xprotocol 对比来说，在收到 network 出来的字节数据时，执行Dispatch 方法

```go
    frame, err := sc.protocol.Decode(streamCtx, buf)
    // 如果没有足够数据，则直接返回，若是凑够了一个frame ，则handleFrame
    handleFrame(streamCtx, xframe)
```

对于http2 来说，一个 请求/响应 对应多个frame（至少包含header 和 data 2个frame），一个stream 对应一个请求加 多个响应  即 多个frame。除了数据frame，http2 还支持很多的control frame。

对于xprotocol 来说，对于普通rpc 协议（支持steaming rpc的协议除外）

1. 一个 请求/响应 对应一个frame（或者说一个frame 只区分 request/response），一个stream 对应一个请求 加一个响应 2个frame。
2. 请求和 响应 一般共用一个统一的数据格式，因此可以用一个 frame struct 表示
3. 一般会支持 心跳机制，即心跳frame

### StreamConnection

StreamConnection is a connection runs multiple streams

```
mosn/pkg/stream
    http
        stream.go
            func init() {
                str.Register(protocol.HTTP1, &streamConnFactory{})
            }
            type streamConnFactory struct{}
    http2
        stream.go
            func init() {
                str.Register(protocol.HTTP2, &streamConnFactory{})
            }
            type streamConnFactory struct{}
    xprotocol
        factory.go
            func init() {
                stream.Register(protocol.Xprotocol, &streamConnFactory{})
            }
            type streamConnFactory struct{}
    factory.go
        var streamFactories map[types.ProtocolName]ProtocolStreamFactory
        func init() {
            streamFactories = make(map[types.ProtocolName]ProtocolStreamFactory)
        }
        func Register(prot types.ProtocolName, factory ProtocolStreamFactory) {
            streamFactories[prot] = factory
        }
        type ProtocolStreamFactory interface {
            CreateClientStream(...) types.ClientStreamConnection
            CreateServerStream(...) types.ServerStreamConnection
            CreateBiDirectStream(...) types.ClientStreamConnection
            ProtocolMatch(context context.Context, prot string, magic []byte) error
        }
    stream.go
    types.go
```

stream 包最外层 定义了ProtocolStreamFactory interface
，针对每个协议 都有一个对应的 streamConnFactory 实现（维护在`var streamFactories map[types.ProtocolName]ProtocolStreamFactory`），协议对应的pkg 内启动时自动执行 init 方法，注册到map。最终 实现根据 Protocol 得到 streamConnFactory 进而得到 ServerStreamConnection 实例

```go
// mosn/pkg/stream/factory.go
func CreateServerStreamConnection(context context.Context, prot api.Protocol, connection api.Connection,
	callbacks types.ServerStreamConnectionEventListener) types.ServerStreamConnection {

	if ssc, ok := streamFactories[prot]; ok {
		return ssc.CreateServerStream(context, connection, callbacks)
	}

	return nil
}
```

![](/public/upload/mesh/mosn_StreamConnection.png)

mosn 数据接收时，从`proxy.onData` 收到传上来的数据，执行对应协议的`serverStreamConnection.Dispatch` ==> 根据协议解析数据 ，经过协议解析，收到一个完整的请求时`serverStreamConnection.handleFrame` 会创建一个 Stream，然后逻辑 转给了`StreamReceiveListener.OnReceive`。proxy.downStream 实现了 StreamReceiveListener

![](/public/upload/mesh/mosn_Stream.png)

### 连接池管理

同样应用了工厂模式

```
mosn/pkg/types
    upstream.go
        func init() {
	        ConnPoolFactories = make(map[api.Protocol]bool)
        }
        var ConnPoolFactories map[api.Protocol]bool
        func RegisterConnPoolFactory(protocol api.Protocol, registered bool) {
            ConnPoolFactories[protocol] = registered
        }
mosn/pkg/network
    connpool.go
        func init() {
            ConnNewPoolFactories = make(map[types.ProtocolName]connNewPool)
        }
        var ConnNewPoolFactories map[types.ProtocolName]connNewPool
        func RegisterNewPoolFactory(protocol types.ProtocolName, factory connNewPool) {
            ConnNewPoolFactories[protocol] = factory
        }
mosn/pkg/stream
    http
        connpool.go
            func init() {
                network.RegisterNewPoolFactory(protocol.HTTP1, NewConnPool)
	            types.RegisterConnPoolFactory(protocol.HTTP1, true)
            }
    http2
        connpool.go
            func init() {
                network.RegisterNewPoolFactory(protocol.HTTP2, NewConnPool)
	            types.RegisterConnPoolFactory(protocol.HTTP2, true)
            }
    xprotocol
        factory.go
            func init() {
                network.RegisterNewPoolFactory(protocol.Xprotocol, NewConnPool)
	            types.RegisterConnPoolFactory(protocol.Xprotocol, true)
            }
    factory.go
    stream.go
    types.go
        type Client interface
    client.go
        type client struct
```

![](/public/upload/mesh/mosn_ConnectionPool.png)

### 协议的编解码

工厂模式，各个协议的包在启动时，将自己注册到protocolMap 和 matcherMap 中。

```
mosn/pkg/protocol/xprotocol
    bolt
        protocol.go
            func init() {
	            xprotocol.RegisterProtocol(ProtocolName, &boltProtocol{})
            }
    dubbo
        protocol.go
            func init() {
                xprotocol.RegisterProtocol(ProtocolName, &dubboProtocol{})
            }
    factory.go
        var (
            protocolMap = make(map[types.ProtocolName]XProtocol)
            matcherMap  = make(map[types.ProtocolName]types.ProtocolMatch)
        )
        func RegisterProtocol(name types.ProtocolName, protocol XProtocol) error {
            ...
            protocolMap[name] = protocol
            ...
        }
```

![](/public/upload/mesh/mosn_xprotocol.png)

一个dubbo 协议的config.json 为例

```json
{
    "servers":[
        {
            "listeners":[
                {
                    "filter_chains":[
                        {
                            "filters":[
                                {
                                    "downstream_protocol": "X",
                                    "upstream_protocol": "X",
                                    "extend_config": {
                                        "sub_protocol": "dubbo"
                                    }
                                }
                            ]
                        }
                    ]
                }
            ]
        }
    ]
}
```

bolt、dubbo 都属于 xprotocol ，它们的区别在于 协议格式的不同（“语义”不同），但数据的通信流程 是相同的（“语法”相同）

![](/public/upload/mesh/mosn_frame.png)


## filter扩展机制

[MOSN 源码解析 - filter扩展机制](https://mosn.io/zh/blog/code/mosn-filters/)MOSN 使用了过滤器模式来实现扩展。MOSN 把过滤器相关的代码放在了 pkg/filter 目录下，包括 accept 过程的 filter，network 处理过程的 filter，以及 stream 处理的 filter。其中 accept filters 目前暂不提供扩展（加载、运行写死在代码里面，如要扩展需要修改源码）， steram、network filters 是可以通过定义新包在 pkg/filter 目录下实现扩展。

mosn 的配置文件config.json 中的Listener 配置包含 stream filter 配置

```json
"listeners":[
    {
        "name":"",
        "address":"", ## Listener 监听的地址
        "filter_chains":[],  ##  MOSN 仅支持一个 filter_chain
        "stream_filters":[], ## 一组 stream_filter 配置，目前只在 filter_chain 中配置了 filter 包含 proxy 时生效
    }
]
```
代码中的示例
```
mosn/pkg/filter/stream
    faultinject
        factory.go
            func init() {
                api.RegisterStream(v2.FaultStream, CreateFaultInjectFilterFactory)
            }
            type FilterConfigFactory struct {
                Config *v2.StreamFaultInject
            }
    mixer
        func init() {
            api.RegisterStream(v2.MIXER, CreateMixerFilterFactory)
        }
        type FilterConfigFactory struct {
	        MixerConfig *v2.Mixer
        }
mosn.io/api
    filter_factory.go
        func init() {
            creatorListenerFactory = make(map[string]ListenerFilterFactoryCreator)
            creatorStreamFactory = make(map[string]StreamFilterFactoryCreator)
            creatorNetworkFactory = make(map[string]NetworkFilterFactoryCreator)
        }
        func RegisterStream(filterType string, creator StreamFilterFactoryCreator) {
            creatorStreamFactory[filterType] = creator
        }
```

## 与control plan 的交互

`pkg/xds/v2/adssubscriber.go` 启动发送线程和接收线程

```go
func (adsClient *ADSClient) Start() {
    adsClient.StreamClient = adsClient.AdsConfig.GetStreamClient()
    utils.GoWithRecover(func() {
        adsClient.sendThread()
    }, nil)
    utils.GoWithRecover(func() {
        adsClient.receiveThread()
    }, nil)
}
```go

定时发送请求
```go
func (adsClient *ADSClient) sendThread() {
    refreshDelay := adsClient.AdsConfig.RefreshDelay
    t1 := time.NewTimer(*refreshDelay)
    for {
        select {
        ...
        case <-t1.C:
            err := adsClient.reqClusters(adsClient.StreamClient)
            if err != nil {
                log.DefaultLogger.Infof("[xds] [ads client] send thread request cds fail!auto retry next period")
                adsClient.reconnect()
            }
            t1.Reset(*refreshDelay)
        }
    }
}
```

接收响应

```go
func (adsClient *ADSClient) receiveThread() {
    for {
        select {
    
        default:
            adsClient.StreamClientMutex.RLock()
            sc := adsClient.StreamClient
            adsClient.StreamClientMutex.RUnlock()
            ...
            resp, err := sc.Recv()
            ...
            typeURL := resp.TypeUrl
            HandleTypeURL(typeURL, adsClient, resp)
        }
    }
}
```

处理逻辑是事先注册好的函数

```go
func HandleTypeURL(url string, client *ADSClient, resp *envoy_api_v2.DiscoveryResponse) {
    if f, ok := typeURLHandleFuncs[url]; ok {
        f(client, resp)
    }
}
func init() {
    RegisterTypeURLHandleFunc(EnvoyListener, HandleEnvoyListener)
    RegisterTypeURLHandleFunc(EnvoyCluster, HandleEnvoyCluster)
    RegisterTypeURLHandleFunc(EnvoyClusterLoadAssignment, HandleEnvoyClusterLoadAssignment)
    RegisterTypeURLHandleFunc(EnvoyRouteConfiguration, HandleEnvoyRouteConfiguration)
}
```

以cluster 信息为例 HandleEnvoyCluster

```go
func HandleEnvoyCluster(client *ADSClient, resp *envoy_api_v2.DiscoveryResponse) {
    clusters := client.handleClustersResp(resp)
    ...
    conv.ConvertUpdateClusters(clusters)
    clusterNames := make([]string, 0)
    ...
    for _, cluster := range clusters {
        if cluster.GetType() == envoy_api_v2.Cluster_EDS {
            clusterNames = append(clusterNames, cluster.Name)
        }
    }
    ...
}
```

会触发ClusterManager 更新cluster 

```go
func ConvertUpdateEndpoints(loadAssignments []*envoy_api_v2.ClusterLoadAssignment) error {
    for _, loadAssignment := range loadAssignments {
        clusterName := loadAssignment.ClusterName
        for _, endpoints := range loadAssignment.Endpoints {
            hosts := ConvertEndpointsConfig(&endpoints)
            clusterMngAdapter := clusterAdapter.GetClusterMngAdapterInstance()
            ...
            clusterAdapter.GetClusterMngAdapterInstance().TriggerClusterHostUpdate(clusterName, hosts); 
            ...
            
        }
    }
    return errGlobal
}
```

## 服务发现注册

从一个公司的实际来说，不可能一下子所有的服务都在容器环境内运行。容器环境内的rpc 服务启动时需要将自己的服务信息注册到 registry 上，进而可以被容器环境外的服务访问到。有几种方式

1. 从k8s向registry 同步数据
2. 业务容器的sdk 直接向registry 写入数据
3. 业务容器的sdk 通过 sidecar 向registry 写入数据

```
mosn/pkg/upstream/servicediscovery/dubbod
    init.go
        func init() {
	        Init()
        }
    bootstrap.go
        func Init( /*port string, dubboLogPath string*/ ) {
	        r := chi.NewRouter()
            r.Post("/sub", subscribe)
	        r.Post("/unsub", unsubscribe)
	        r.Post("/pub", publish)
	        r.Post("/unpub", unpublish)
            ...
        }
    pub.go
        func publish(w http.ResponseWriter, r *http.Request) {
	        err = doPubUnPub(req, true)
	        return
        }
```

sidecar 启动时，通过func init 启动一个webserver组件，和业务容器约定一个pub请求，sidecar web server 收到请求之后，将信息写到 registry（比如zk） 上。 

![](/public/upload/mesh/mosn_pub.png)