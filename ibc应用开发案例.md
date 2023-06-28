# ibc应用开发案例

## 1. 创建 Images && Container 
```shell
# 创建 Images
docker build -f Dockerfile-ubuntu . -t planet_images

# 创建 Container
 docker create --name planet_c -i -v W:/Cosmos/planet:/planet -w /planet -p 1317:1317 -p 4500:4500 -p 26657:26657 -p 1318:1318 -p 4501:4501 -p 26658:26658 planet_images

# 启动容器
docker start planet_c
```

## 2. 创建项目
**1.**   用ignite生成一条新的区块链名字叫planet。

```
docker exec -it planet_c ignite scaffold chain planet --no-module
```

**2.**  使用ignite生成一个Blog的模块，并且集成IBC。

```
docker exec -it planet_c ignite scaffold module blog --ibc
```

**3.** 给blog模块添加针对博文（post）的增删改查。

```shell
# 参数list 表示存储时以数组的形式存储
docker exec -it planet_c ignite scaffold list post title content creator --no-message --module blog

```

**4.** 添加已发生成功博文（sentPost）的增删改查。

```
docker exec -it planet_c ignite scaffold list sentPost postID title chain creator --no-message --module blog
```

**5.** 添加发送超时博文（timeoutPost）的增删改查。

```
docker exec -it planet_c ignite scaffold list timedoutPost title chain creator --no-message --module blog
```

**6.** 添加IBC发送数据包和确认数据包的结构。

```
docker exec -it planet_c ignite scaffold packet ibcPost title content --ack postID --module blog

```

## 3. 修改业务逻辑
  
**7.** 在proto/blog/packet.proto目录下修改`IbcPostPacketData`，添加创建人`Creator`， 并重新编译proto文件。在x/blog/keeper/msg_server_ibc_post.go。编译完成后在x/blog/keeper/msg_server_ibc_post.go中发送数据包前更新`Creator`。

```shell
# 在proto/blog/packet.proto目录下修改`IbcPostPacketData`，添加创建人`Creator`
  string creator = 3;
  
# 重新编译proto文件
docker exec -it planet_c ignite chain build

# 在x/blog/keeper/msg_server_ibc_post.go中的SendIbcPost更新`Creator`
	packet.Creator = msg.Creator

```

**8.** 修改keeper方法中的`OnRecvIbcPostPacket `。

```
id := k.AppendPost(
        ctx,
        types.Post{
            Creator: packet.SourcePort + "-" + packet.SourceChannel + "-" + data.Creator,
            Title:   data.Title,
            Content: data.Content,
        },
    )

    packetAck.PostID = strconv.FormatUint(id, 10)
```

**9.** 修改keeper方法中的`OnAcknowledgementIbcPostPacket `。

```
k.AppendSentPost(
            ctx,
            types.SentPost{
                Creator: data.Creator,
                PostID:  packetAck.PostID,
                Title:   data.Title,
                Chain:   packet.DestinationPort + "-" + packet.DestinationChannel,
            },
        )
```

**10.** 修改keeper方法中的`OnTimeoutIbcPostPacket `。

```
k.AppendTimedoutPost(
        ctx,
        types.TimedoutPost{
            Creator: data.Creator,
            Title:   data.Title,
            Chain:   packet.DestinationPort + "-" + packet.DestinationChannel,
        },
    )
```

**11.** 添加链启动的配置文件。

```
# earth.yml
accounts:
  - name: alice
    coins: ["1000token", "100000000stake"]
  - name: bob
    coins: ["500token", "100000000stake"]
validator:
  name: alice
  staked: "100000000stake"
faucet:
  name: bob
  coins: ["5token", "100000stake"]
genesis:
  chain_id: "earth"
init:
  home: "$HOME/.earth"
  
# mars.yml
accounts:
  - name: alice
    coins: ["1000token", "1000000000stake"]
  - name: bob
    coins: ["500token", "100000000stake"]
validator:
  name: alice
  staked: "100000000stake"
faucet:
  host: ":4501"
  name: bob
  coins: ["5token", "100000stake"]
host:
  rpc: ":26659"
  p2p: ":26658"
  prof: ":6061"
  grpc: ":9092"
  grpc-web: ":9093"
  api: ":1318"
genesis:
  chain_id: "mars"
init:
  home: "$HOME/.mars"
```


**12.** 分别启动两条链

```shell
# docker 
docker exec -it planet_c ignite chain serve -c earth.yml

docker exec -it planet_c ignite chain serve -c mars.yml
```

**13.** 启动中继器

```shell
# 清理旧内容
docker exec -it planet_c rm -rf ~/.ignite/relayer

# 一行
docker exec -it planet_c ignite relayer configure -a --source-rpc "http://0.0.0.0:26657" --source-faucet "http://0.0.0.0:4500" --source-port "blog" --source-version "blog-1" --source-gasprice "0.0000025stake" --source-prefix "cosmos" --source-gaslimit 300000 --target-rpc "http://0.0.0.0:26659" --target-faucet "http://0.0.0.0:4501" --target-port "blog" --target-version "blog-1" --target-gasprice "0.0000025stake" --target-prefix "cosmos" --target-gaslimit 300000

# 多行
docker exec -it planet_c ignite relayer configure -a \
  --source-rpc "http://0.0.0.0:26657" \
  --source-faucet "http://0.0.0.0:4500" \
  --source-port "blog" \
  --source-version "blog-1" \
  --source-gasprice "0.0000025stake" \
  --source-prefix "cosmos" \
  --source-gaslimit 300000 \
  --target-rpc "http://0.0.0.0:26659" \
  --target-faucet "http://0.0.0.0:4501" \
  --target-port "blog" \
  --target-version "blog-1" \
  --target-gasprice "0.0000025stake" \
  --target-prefix "cosmos" \
  --target-gaslimit 300000

docker exec -it planet_c ignite relayer connect
```

**14.** 从earth链向mars链发送博文数据包（注意修改channel id）

```
planetd tx blog send-ibc-post blog channel-4 "Hello" "Hello Mars, I'm Alice from Earth" --from alice --chain-id earth --home ~/.earth
```

**15.** 通过rpc查询验证结果。

```
planetd q blog list-post --node tcp://localhost:26659

planetd q blog list-sent-post
```

## 4. 课后作业

1.	参考该文档https://docs.ignite.com/guide/install， 在自己的电脑上安装Ignite CLI环境。

2.	按照课堂上演示的案例，重复每条步骤，在本地生成一个同样的项目。（在生成的过程中，确保理解每个步骤的意义，通过生成的代码变化，弄清楚每条命令具体做了什么）

3.	创建一个新的ibc数据包，名称为updatePost，包含postID、title、content。通过将该数据包发送给对手链，可更新存储在对手链中某条Post（通过postID）的标题和内容。对手链在ack确认数据包中将返回是否更新成功，发送链收到确认数据包后更新本地之前存储的SentPost为新的title。
```shell
# 
docker exec -it planet_c ignite scaffold packet updatePost postID title content --ack postID --module blog
```

4.	启动中继器和两条链节点，发送一条post给对手链，发送成功后，确保对手链能查询到该post以及当前链保存了相应的sentPost。获取该post对应的postID，向对手链发送一条updatePost的ibc数据包，更新该post的title和content。发送成功后，查询对手链的post和当前链的sentPost，确保更新成功。

