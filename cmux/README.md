### 已知内容
在golang中，无论是http服务还是rpc服务，其服务的流程都如下所示:
```golang
func Serve(l net.Listener) {
    for {
		rw, e := l.Accept() // 获取连接
		if e != nil {
			// handle error
		}
		// handle connection
    }
}
```
### 为什么
最近在重新学习gRPC的过程中，一直在想一个问题，如果一个服务若在只能监听某一端口时，既能对外提供rpc服务，又能对外提供http服务或者其它类型（SSH）的服务。这样既能大大提供io吞吐量，也能大大减少需要暴露给外界或其它内部服务的端口号。
### 怎么做
由于gRPC是基于HTTP2进行传输的，我们知道HTTP2是基于frame来进行传递的，因此我们只需要解析特定到特定的frame，就可以认为这是一个rpc请求，并转发给监听给特点端口的rpc服务去进行服务。其它类型的服务同理。经过调研后，我们发现[cmux](https://github.com/soheilhy/cmux)已经能够提供这类的服务了。
具体代码如下:
```golang
func reuseProtForDifferentServer(l net.Listener) {
    // create a cmux for reuse port
    m := cmux.New(l)
    // create different service with different match function
    grpcListener := m.MatchWithWriters(cmux.HTTP2MatchHeaderFieldSendSettings("content-type", "application/grpc"))
    httpListener := m.Match(cmux.HTTP1Fast())
    
    // create grpc server and bind grpcListener into this server
    grpcServer := grpc.Server()
    go func () {
        if err != grpcServer.Serve(grpcListener); err != nil {
            panic(err)
        }
    }()

    // creare http server and bind httpListener into this server
    httpServer := http.Server{}
    go func () {
        if err != httpServer.Serve(httpListener); err != nil {
            panic(err)
        }
    }

    // start serving cmux for accept connection
    m.serve()
}
```
### 精妙设计
cmux设计如下:
```golang
type cMux struct {
	root        net.Listener        // 监听主端口
	bufLen      int                 // 每个matcherListener缓存队列里的连接数目
	errh        ErrorHandler        // 异常处理函数
	donec       chan struct{}       // 退出通知channel
	sls         []matchersListener  // 不同服务的端口监听器
	readTimeout time.Duration
}
```
在上述cmux的内部设计中，`matchersListener`负责不同类型服务的监听处理，其内部设计如下:
```golang
type matchersListener struct {
	ss []MatchWriter  // 过滤器
	l  muxListener    // 端口监听
}

type MatchWriter func(io.Writer, io.Reader) bool // 根据请求的内容来进行匹配

type muxListener struct {
	net.Listener            // 内嵌Listener接口
	connc chan net.Conn     // 连接的channel
}
```
muxListener通过实行`net.Listener的Accept接口来实行不同类型的服务转发`。
cmux的工作流程如下:
- 在Serve函数内部接受监听主端口的连接
```golang
func (m *cMux) Serve() error {
    // ...
    for {
    	c, err := m.root.Accept()
    	if err != nil {
    		if !m.handleErr(err) {
    			return err
    		}
    		continue
    	}
    	go m.serve(c, m.donec, &wg)
    }
}
```
- 在serve函数内部进行连接转发
```golang
func (m *cMux) serve(c net.Conn, donec <-chan struct{}, wg *sync.WaitGroup) {
    for _, sl := range m.sls {
		for _, s := range sl.ss {  // sl.ss为服务的过滤器
			matched := s(muc.Conn, muc.startSniffing())
			if matched {
				select {
				case sl.l.connc <- muc:  // 若匹配则进行请求转发
				case <-donec:
					_ = c.Close()
				}
				return
			}
		}
	}
}
```
- muxListener重载的Accept函数
```golang
func (l muxListener) Accept() (net.Conn, error) {
	c, ok := <-l.connc // 从channel阻塞读取请求
	if !ok {
		return nil, ErrListenerClosed
	}
	return c, nil
}
```
### 踩坑记录
- grpc的测试过程中，由于gRPC client阻塞的接收`SETTING`，若采用`grpcListener := m.Match(cmux.HTTP2HeaderField("content-type", "application/grpc"))`client会一直阻塞直至超时。。。。
