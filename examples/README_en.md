#Service deployment
Here we provide two examples:
- Deploy simpleserver
- Deploy by using Go’s HelloWorld’s program TestApp.HelloGo
## Deploy simpleserver
1. Execute: 
   ```cd examples/simple && kubectl apply -f simpleserver.yaml```

   Example description：
   - examples/simple/Dockerfile will create the container image, and cmd/tarscli/Dockerfile will build the base image.
   - `tarscli genconf` in start.sh will be used to generate tars services’ starting configuration. 
   - The file _server_meta.yaml is used to configure services’ metadata. You can refer to the structure of ServerConf in app/genconf/config.go for the field information. The endpoint is `tcp -h ${local_ip} -p ${random_port}` by default and supports automatically filling in IP and random ports.
 
2. To verify if deployment is successfully
   - log into db_tars and execute `select * from t_server_conf\G`. You should see that the simpleserver’s node information is registered. 
   - Enter TarsWeb to check if services are deployed successfully
 
## Deploy by using Go’s HelloWorld’s program TestApp.HelloGo
Follow the instruction to deploy [TarsGo development environment](https://tarscloud.github.io/TarsDocs_en/env/tarsgo.html)
Also refer to the instructions on [creating services with TarsGo](https://tarscloud.github.io/TarsDocs_en/hello-world/tarsgo.html)
 
### Create a service

1. Run bash create_tars_server.sh to generate the required files for creating a service
   ```shell
   sh $GOPATH/src/github.com/TarsCloud/TarsGo/tars/tools/create_tars_server.sh [App] [Server] [Servant]
  	For example: 
   sh $GOPATH/src/github.com/TarsCloud/TarsGo/tars/tools/create_tars_server.sh TestApp HelloGo SayHello
   ```

2. After the command is executed, the code will be generated into GOPATH, and the directory will be named as `APP/Server`. The actual directory path is also shown in the generated code.

   ```
   [root@1-1-1-1 src]# sh $GOPATH/src/github.com/TarsCloud/TarsGo/tars/tools/create_tars_server.sh TestApp HelloGo SayHello
   [create server: TestApp.HelloGo ...]
   [mkdir: /data/gopath/src/TestApp/HelloGo/]
   >>>Now doing:./config.conf >>>>
   >>>Now doing:./main.go >>>>
   >>>Now doing:./Servant_imp.go >>>>
   >>>Now doing:./makefile >>>>
   >>>Now doing:./Servant.tars >>>>
   >>>Now doing:./start.sh >>>>
   >>>Now doing:client/client.go >>>>
   >>>Now doing:vendor/vendor.json >>>>
   >>>Now doing:debugtool/dumpstack.go >>>>
   >>> Great！Done! You can jump in /data/gopath/src/TestApp/HelloGo
   >>> Tips: After editing the Tars file, execute the following cmd to automatically generate golang files.
   >>>       /data/gopath/bin/tars2go *.tars
   ```

### Define Interface File 

We define two functions respectively called `Add` and `Sub` under the SayHello interface. The client request parameters are two integers a and b. The server calculates the operation of a and b, assigns the result to c, and returns the value of c.

```go
module TestApp
{
	interface SayHello
	{
	    int Add(int a,int b,out int c); // Some example function
	    int Sub(int a,int b,out int c); // Some example function
	};
};
```

**Note**: in the parameters, **out** is the operation output of a and b.

### Server Development 
1. convert the tars protocol file into the format of Golang

   ```shell
cd $GOPATH/src/TestApp/HelloGo/
$GOPATH/bin/tars2go SayHello.tars
   ```

2. Implement the server logic:

   ```vim $GOPATH/src/TestApp/HelloGo/sayhello_imp.go```
  - Calculate `a + b` in `Add`, and assign output to `c`
  - Calculate `a - b` in `Sub`, and assign output to `c`

	```go
    package main

    import (
      "context"
    )

    // SayHelloImp servant implementation
    type SayHelloImp struct {
    }

    // Init servant init
    func (imp *SayHelloImp) Init() (error) {
    //initialize servant here:
    //...
    return nil
    }

    // Destroy servant destory
    func (imp *SayHelloImp) Destroy() {
    //destroy servant here:
    //...
    }

    func (imp *SayHelloImp) Add(ctx context.Context, a int32, b int32, c *int32) (int32, error) {
    //Doing something in your function
    *c = a + b
    return 0, nil
    }
    func (imp *SayHelloImp) Sub(ctx context.Context, a int32, b int32, c *int32) (int32, error) {
    //Doing something in your function
    *c = a - b
    return 0, nil
    }
	```

**Note**: Remember the function names need to be capitalized, according to the exporting regulation of Go language

3. Edit the main function, the initial code has been implemented by the TARS framework.
   ```vim $GOPATH/src/TestApp/HelloGo/main.go```

   ```go
   package main

   import (
     "fmt"
     "os"

     "github.com/TarsCloud/TarsGo/tars"

     "TestApp/HelloGo/TestApp" //Edit like this
   )

   func main() {
     // Get server config
     cfg := tars.GetServerConfig()

    // New servant imp
     imp := new(SayHelloImp)
     err := imp.Init()
     if err != nil {
       fmt.Printf("SayHelloImp init fail, err:(%s)\n", err)
       os.Exit(-1)
     }
     // New servant
     app := new(TestApp.SayHello)
     // Register Servant
     app.AddServantWithContext(imp, cfg.App+"."+cfg.Server+".SayHelloObj")

     // Run application
     tars.Run()
   }
   ```

**Note**: change the `TestApp` in `import` to `TestApp/HelloGo/TestApp`

4. Copy the following 5 files under [HelloGo’s Demo](TestApp/HelloGo) to $GOPATH/src/TestApp/HelloGo/ 
`Dockerfile`、`makefile`、`_server_meta.yaml`、`simpleserver.yaml`、`start.sh` 

5. Modify the ’_server_meta.yaml’ file and fill in the values ​​of the ‘application’, ‘server’, and ‘object’ fields based on your situation. 

   `vim $GOPATH/src/TestApp/HelloGo/_server_meta.yaml`
   ```yaml
   version: v1
   application: TestApp
   server: HelloGo
   adapters:
     - object: SayHelloObj
   ```

6. Modify `start.sh`, change service name and configuration name based on your actual situation
   
   ```vim $GOPATH/src/TestApp/HelloGo/start.sh```
   #!/bin/bash

   # start server
   ${TARS_PATH}/bin/HelloGo --config=${TARS_PATH}/conf/HelloGo.conf
   ```

7. Modify makefile, fill in `APP` and `TARGET` based on your actual situation. 

   `vim $GOPATH/src/TestApp/HelloGo/makefile`
   
   ```makefile 
   APP    := TestApp
   TARGET := HelloGo

   GOBUILD      := go build
   DOCKER_BUILD := docker build

   REPO         ?= ccr.ccs.tencentyun.com/tarsbase
   VERSION  ?= $(shell date "+%Y%m%d%H%M%S") #Edit if you need

   LOWCASE_TARGET := $(shell echo $(TARGET) | tr '[:upper:]' '[:lower:]')
   IMG_REPO       := $(REPO)/$(LOWCASE_TARGET)

   build:
   	GOOS=linux $(GOBUILD) -o $(TARGET)

   img: build
     	$(DOCKER_BUILD) --build-arg SERVER=$(TARGET) -t $(IMG_REPO):$(VERSION) .

   tgz: build
	tar czf $(TARGET).tgz $(TARGET) _server_meta.yaml

   patch: tgz
	curl --data-binary @$(TARGET).tgz "${TARS_EP}/patch?server=$(TARGET)&version=$(VERSION)"
	
   stdout:
	@curl "${TARS_EP}/stdout?server=$(TARGET)"

   listlog:
	@curl "${TARS_EP}/listlog?app=$(APP)&server=$(TARGET)"

   tailog:
	@curl "${TARS_EP}/tailog?app=$(APP)&server=$(TARGET)&filename=$(LOG_NAME)"

   clean:
         rm -rf $(TARGET)
   ```
 
8. Create images
   ```shell
   cd $GOPATH/src/TestApp/HelloGo/ && make img
   ```

   ```
   [root@1-1-1-1 HelloGo]# make img
   GOOS=linux go build -o HelloGo
   docker build --build-arg SERVER=HelloGo -t ccr.ccs.tencentyun.com/tarsbase/hellogo:20200725145313 .
   Sending build context to Docker daemon  23.44MB
   Step 1/5 : FROM ccr.ccs.tencentyun.com/tarsbase/tarscli:latest
    ---> ef39bfa9b5e6
   Step 2/5 : ARG SERVER=please_build_by_make_img
    ---> Using cache
    ---> c3a89b1e1be5
   Step 3/5 : ENV TARS_BUILD_SERVER ${SERVER}
    ---> Using cache
    ---> 5f8fc936380e
   Step 4/5 : COPY $SERVER _server_meta.yaml start.sh $TARS_PATH/bin/
    ---> Using cache
    ---> 05f31e9e6e4a
   Step 5/5 : CMD tarscli supervisor
    ---> Using cache
    ---> be8b4a482834
   Successfully built be8b4a482834
   Successfully tagged ccr.ccs.tencentyun.com/tarsbase/hellogo:20200725145313
   ```

Remember the created image’s name. The example here has the following image name as an example: `ccr.ccs.tencentyun.com/tarsbase/hellogo:20200725145313`

9. Modify the ‘simpleserver.yaml’ file and deploy ‘TestApp.HelloGo’ to the kubernetes cluster

`vim $GOPATH/src/TestApp/HelloGo/simpleserver.yaml` Edit the `image`’s value to the image name from the last step. 

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: hellogo
   spec:
     selector:
       matchLabels:
         app: hellogo
     replicas: 1
     template:
       metadata:
         labels:
           app: hellogo
       spec:
         containers:
         - name: hellogo
           image: ccr.ccs.tencentyun.com/tarsbase/hellogo:20200725145313 #Edit here to the image you just build
           imagePullPolicy: IfNotPresent 
           lifecycle:
             preStop:
               exec:
                 command: ["tarscli", "prestop"]
           readinessProbe:
             exec:
               command: ["tarscli", "hzcheck"]
             initialDelaySeconds: 2
             timeoutSeconds: 8
             periodSeconds: 6
   ```

Execute `kubectl apply -f simpleserver.yaml`
10. Verify Deployment
   - Log in to db_tars, and execute select * from t_server_conf\G to see if HelloGo's node information has been automatically registered.
   - Visit TarsWeb to check that the service has been deployed successfully.

### Client Development
We provide a demo for a [client](TestApp/HelloGo/client) under the directory client.

```go
package main

import (
	"fmt"

	"github.com/TarsCloud/TarsGo/tars"

	"TestApp/HelloGo/TestApp"
)

func main() {
	comm := tars.NewCommunicator()
  //If your server "HelloGo" has been registered to tarsregistry
  obj := fmt.Sprintf("TestApp.HelloGo.SayHelloObj")
  //If your server "HelloGo" hasn't been registerd to tarsregistry
  //obj := fmt.Sprintf("TestApp.HelloNew.SayHelloObj@tcp -h 127.0.0.1 -p 10015 -t 60000")
  //"127.0.0.1" and "10015" should be change to your HelloGo's IP and Port.
	app := new(TestApp.SayHello)

	//If your server has been registered to tarsregistry
  comm.SetProperty("locator", "tars.tarsregistry.QueryObj@tcp -h 192.168.0.55 -p 17890")
  //"192.168.0.55" should be change to your tarsregistry's IP

	comm.StringToProxy(obj, app)
	var out, i int32
	i = 123
	ret, err := app.Add(i, i*2, &out)
	if err != nil {
		fmt.Println(err)
		return
	}
	fmt.Println(ret, out)
}
```

Compile test

```shell
[root@1-1-1-1 client]# go build client.go
[root@1-1-1-1 client]# ./client 
0 369
```