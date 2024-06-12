---
title: Golang使用ProtoBuf
date: 2019-09-11
updated: 2019-09-11
issueid: 5
tags:
---
两个例子都使用了`Golang`最新的`module feature`
第一个例子还是放到了`$GOPATH`下


#### go.mod >> module chaochaogege.com/filecatcher

如果我的域名`chaochaogege.com`

路径 `$GOPATH/src/chaochaogege.com/`

`chaochaogege.com`里面有个`project`，`filecatcher`

我现在有两个`proto`文件都处于 `chaochaogege.com/filecatcher/common/`

- TaskInfo.proto
- ChunkInfo.proto

<!--more-->

我们需要指定`proto`文件路径

比如我们当前目录是`$GOPATH/src/`
那么运行如下命令可以编译成功

```
protoc --proto_path=./ --go_out=chaochaogege.com/common/ chaochaogege.com/common/TaskInfo.proto
```

- `--proto_path`: 指定从哪里寻找`proto`文件
- `--go_out`: 生成的`pb.go`目录
- `chaochaogege.com/common/TaskInfo.proto`:  要编译的源文件

下面是两个`proto`文件

*ChunkInfo.proto*

```
syntax = "proto3";
package chaochaogege.filecatcher.common;

option go_package = "chaochaogege.com/filecatcher/common";

message ChunkInfo{
uint32 startBytes = 1;
uint32 endBytes = 2;
uint32 totalBytes = 3;
uint32 downloaded = 4;
uint32 orderId = 5;// 间隔1000，便于中间切分

}
```

*TaskInfo.proto*

```
syntax = "proto3";
package chaochaogege.filecatcher.common;

option go_package = "chaochaogege.com/filecatcher/common";

import "chaochaogege.com/filecatcher/common/ChunkInfo.proto";

message TaskInfo{
string name = 1;
string storePath = 2;
uint32 workersNum = 3;
uint32 totalSize = 4;
repeated chaochaogege.filecatcher.common.ChunkInfo chunkInfo = 5;
}
```

`TaskInfo.proto`里面的`go_package`指定的生成路径指的是当前目录下的路径

因为处于`$GOPATH/src/`下面，所以最后生成的路径正好是`$GOPATH/src/ + chaochaogege.com/filecatcher/common/`

--------------------------------------------------------------------------------

#### go.mod >> module filecatcher

因为使用了`module`特性之后我们就不想全部的project都挂载到域名之下，所以现在去掉`chaochaogege.com`

*ChunkInfo.proto*

```
syntax = "proto3";
package filecatcher.common;

option go_package = "filecatcher/common";

message ChunkInfo{
uint32 startBytes = 1;
uint32 endBytes = 2;
uint32 totalBytes = 3;
uint32 downloaded = 4;
uint32 orderId = 5;// 间隔1000，便于中间切分

}
```

*TaskInfo.proto*

```
syntax = "proto3";
package filecatcher.common;

option go_package = "filecatcher/common";
import "filecatcher/common/ChunkInfo.proto";

message TaskInfo{
string name = 1;
string storePath = 2;
uint32 workersNum = 3;
uint32 totalSize = 4;
repeated filecatcher.common.ChunkInfo chunkInfo = 5;
}
```

**第一个例子中go.mod的module是`chaochoagege.com/filecatcher`，既然要去掉域名，那么不要忘记改成`module filecatcher`**

这一次我们不需要到`$GOPATH/src/`目录下了，

我在[官方Github文档](https://github.com/golang/protobuf)上找到一个新的指令格式

注意看我们的`go_package`， 为了运行下面的命令，我们只需要将当前目录置为`filecatcher`的上层

```
protoc --proto_path=. --go_out=paths=source_relative:. filecatcher/common/TaskInfo.proto
```

-----------------------------------------------------------------------------------------

之前没有`go module`的时候代码都要放到`$GOPATH`，并且引用自己写的代码还要加上域名和一堆的前缀

现在使用`module`，只需要在`go.mod`里面指定模块名字，之后就可以用`模块名加包名`引用了