1. Server code:
```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"net"
	"net/http"
	"time"

	"google.golang.org/grpc"
	"google.golang.org/grpc/keepalive"

	"api/helloworld"
)

func main() {
	listen8080, err := net.Listen("tcp", ":8080")
	if err != nil {
		panic(err)
	}

	listen9090, err := net.Listen("tcp", ":9090")
	if err != nil {
		panic(err)
	}

	listen16060, err := net.Listen("tcp", ":16060")
	if err != nil {
		panic(err)
	}

	go func() {
		mux := http.NewServeMux()
		mux.HandleFunc("/api", func(writer http.ResponseWriter, request *http.Request) {
			query := request.URL.Query()
			log.Printf("Received api: %v", query.Encode())
			data, _ := json.Marshal(map[string]string{"message": "api " + query.Get("name")})
			writer.Write(data)
		})
		server := &http.Server{
			Handler: mux,
		}
		err := server.Serve(listen8080)
		if err != nil {
			fmt.Println("8080 error:", err)
		}
	}()

	go func() {
		mux := http.NewServeMux()
		mux.HandleFunc("/actuator", func(writer http.ResponseWriter, request *http.Request) {
			query := request.URL.Query()
			log.Printf("Received actuator: %v", query.Encode())
			data, _ := json.Marshal(map[string]string{"message": "actuator " + query.Get("name")})
			writer.Write(data)
		})
		server := &http.Server{
			Handler: mux,
		}
		err := server.Serve(listen16060)
		if err != nil {
			fmt.Println("16060 error:", err)
		}
	}()

	go func() {
		parameters := keepalive.ServerParameters{
			// 如果一个 client 空闲超过 15s, 发送一个 GOAWAY, 为了防止同一时间发送大量 GOAWAY,
			// 会在 15s 时间间隔上下浮动 15*10%, 即 15+1.5 或者 15-1.5
			MaxConnectionIdle: 15 * time.Second,
			// 如果任意连接存活时间超过 30s, 发送一个GOAWAY
			MaxConnectionAge: 30 * time.Second,
			// 在强制关闭连接之间, 允许有 5s 的时间完成 pending 的 rpc 请求
			MaxConnectionAgeGrace: 5 * time.Second,
			// 如果一个 client 空闲超过 5s, 则发送一个 ping 请求
			Time: 5 * time.Second,
			// 如果 ping 请求 1s 内未收到回复, 则认为该连接已断开
			Timeout: 1 * time.Second,
		}
		enforcementPolicy := keepalive.EnforcementPolicy{
			// 如果客户端两次 ping 的间隔小于 5s，则关闭连接
			MinTime: 5 * time.Second,
			//  即使没有 active stream, 也允许 ping
			PermitWithoutStream: true,
		}
		s := grpc.NewServer(
			grpc.KeepaliveParams(parameters),
			grpc.KeepaliveEnforcementPolicy(enforcementPolicy),
		)
		helloworld.RegisterGreeterServer(s, &Greeter{})
		if err := s.Serve(listen9090); err != nil {
			fmt.Println("9090 error:", err)
		}
	}()

	select {}
}

type Greeter struct {
	helloworld.UnimplementedGreeterServer
}

func (g *Greeter) SayHello(ctx context.Context, request *helloworld.HelloRequest) (*helloworld.HelloReply, error) {
	log.Printf("Received grpc: %v", request.GetName())
	return &helloworld.HelloReply{Message: "grpc " + request.GetName()}, nil
}

```

2. Client code:
```go
// client.go
package main

import (
	"context"
	"log"
	"time"

	"github.com/go-leo/gox/netx/httpx"
	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
	"google.golang.org/grpc/keepalive"

	"api/helloworld"
)

func main() {
	go func() {
		for {
			<-time.After(1 * time.Second)
			textBody, err := httpx.NewRequestBuilder().
				Get().
				URLString("http://helloworld-server:8080/api").
				Query("name", "接口").
				Execute(context.Background(), httpx.PooledClient()).
				TextBody()
			if err != nil {
				log.Println(err)
				continue
			}
			log.Println(textBody)
		}
	}()

	go func() {
		for {
			<-time.After(1 * time.Second)
			textBody, err := httpx.NewRequestBuilder().
				Get().
				URLString("http://helloworld-server:16060/actuator").
				Query("name", "管理").
				Execute(context.Background(), httpx.PooledClient()).
				TextBody()
			if err != nil {
				log.Println(err)
				continue
			}
			log.Println(textBody)
		}
	}()

	go func() {
		clientKeepaliveParameters := keepalive.ClientParameters{
			// 如果没有 activity， 则每隔 10s 发送一个 ping 包
			Time: 10 * time.Second,
			// 如果 ping ack 1s 之内未返回则认为连接已断开
			Timeout: time.Second,
			// 如果没有 active 的 stream， 是否允许发送 ping
			PermitWithoutStream: true,
		}
		conn, err := grpc.Dial("dns:///helloworld-server-headless:9090",
			grpc.WithTransportCredentials(insecure.NewCredentials()),
			grpc.WithDefaultServiceConfig(`{"loadBalancingConfig": [{"round_robin":{}}]}`),
			grpc.WithKeepaliveParams(clientKeepaliveParameters),
		)
		if err != nil {
			log.Fatalf("did not connect: %v", err)
		}
		defer conn.Close()
		c := helloworld.NewGreeterClient(conn)
		for {
			<-time.After(1 * time.Second)
			ctx, _ := context.WithTimeout(context.Background(), time.Second)
			r, err := c.SayHello(ctx, &helloworld.HelloRequest{Name: "微服务"})
			if err != nil {
				log.Fatalf("could not greet: %v", err)
			}
			log.Printf("Greeting: %s", r.GetMessage())
		}
	}()

	select {}
}
```
3. Server Dockerfile
```
FROM xxx
ADD bin/helloworld/server /app
ENTRYPOINT [ "/app" ]
```

4. Client Dockerfile
```
FROM xxx
ADD bin/helloworld/client /app
ENTRYPOINT [ "/app" ]
```

5. Server deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: helloworld
  name: helloworld-server
spec:
  replicas: 3
  selector:
    matchLabels:
      app: helloworld-server
  template:
    metadata:
      labels:
        app: helloworld-server
    spec:
      containers:
        - name: helloworld-server
          command: [
            "/app",
          ]
          image: xxx
          imagePullPolicy: Always
          ports:
            - name: grpc
              containerPort: 9090
              protocol: TCP
            - name: http
              containerPort: 8080
              protocol: TCP
            - name: actuator
              containerPort: 16060
              protocol: TCP
      imagePullSecrets:
        - name: 
```

6. Client deployment.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: helloworld
  name: helloworld-client
spec:
  replicas: 1
  selector:
    matchLabels:
      app: helloworld-client
  template:
    metadata:
      labels:
        app: helloworld-client
    spec:
      containers:
        - name: helloworld-client
          command: [
            "/app",
          ]
          image: xxx
          imagePullPolicy: Always
          ports:
            - name: grpc
              containerPort: 9090
              protocol: TCP
            - name: http
              containerPort: 8080
              protocol: TCP
            - name: actuator
              containerPort: 16060
              protocol: TCP
      imagePullSecrets:
        - name: xxx
```

7. Server K8S Service And Headless Service
```
---
# 配置 service
apiVersion: v1
kind: Service
metadata:
  namespace: helloworld
  name: helloworld-server
  labels:
    app: helloworld-server
spec:
  selector:
    app: helloworld-server
  ports:
    - name: grpc
      port: 9090
      protocol: TCP
      targetPort: 9090
    - name: http
      port: 8080
      protocol: TCP
      targetPort: 8080
    - name: actuator
      port: 16060
      protocol: TCP
      targetPort: 16060

---
# 配置 headless service
apiVersion: v1
kind: Service
metadata:
  namespace: helloworld
  name: helloworld-server-headless
  labels:
    app: helloworld-server-headless
spec:
  clusterIP: None
  selector:
    app: helloworld-server
  ports:
    - name: grpc
      port: 9090
      protocol: TCP
      targetPort: 9090
    - name: http
      port: 8080
      protocol: TCP
      targetPort: 8080
    - name: actuator
      port: 16060
      protocol: TCP
      targetPort: 16060
```

总结：
* grpc keepalive [gRPC 客户端长连接机制实现及 keepalive 分析-如何实现针对 gRPC 客户端的自动重连机制](https://pandaychen.github.io/2020/09/01/GRPC-CLIENT-CONN-LASTING)
* grpc dns resolver 
* grpc load balancer
* k8s headless service