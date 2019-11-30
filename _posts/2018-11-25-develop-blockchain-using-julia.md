---
title: "用200行Julia代码开发一个区块链"
layout: post
date: 2018-11-25T23:50:00Z
tags: ["区块链", "Julia"]
---

对，是标题党，民科的必备技能之一。至于为什么是标题党的原因后面你会看到。

事情的起因是Julia 1.0的发布。1.0出来也有些时间了，这段时间一直被人安利说他既有C++的性能，又像Python那样开发友好，可以用来做科学计算、数据处理等等，让人实在不可忽视。

而学习一个事情最好的方法就是使用它，所谓“以战代练”。刚好以前看过一个[用Python写区块链的例子](https://hackernoon.com/learn-blockchains-by-building-one-117428612f46)，那么应该也可以用Julia来开发一个区块链试试看（Julia：我实在也不是谦虚。我一个科学计算的语言，怎么来开发区块链了？？）

所以标题党的原因之一在于，这次其中一个目的更多是学习Julia，而并不是开发一个区块链。

![这么鼓励人造轮子……别闹了，费曼先生  (图片来源: https://github.com/danistefanovic/build-your-own-x) ](https://upload-images.jianshu.io/upload_images/12457563-057a02dc0089dca7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# “区块链完备”
先剧透一下，这次我们仿照那个Python写区块链的样例，在实现上不存在什么方向上的问题，最后实现了一个区块链原型。在这过程中，除了学习了Julia语法之外，其实还有另外一个更为重要的感想：

以往评价一个编程语言功能是否完备，会说他的语法是否图灵完备、是否适合企业级开发等等。我认为以后也可以加上另一个标准：是否能实现一个区块链原型，即是否**区块链完备**。如果可以，那么说明这个语言已经具备了开发一个较为复杂系统的能力与生态环境，因为至少已包括：
- 密码学的库或底层支持以保证安全
- JSON等数据格式解析
- 可用于生产开发的web框架

那么Julia如何？先放这次结论，Julia已经具有区块链完备的特性。

但是Julia 1.0和之前版本还是有较多升级，也有一些前后兼容性以及需要更好配套生态支持等问题，需要在开发中注意

# 0. 开发前的工作
## 开发环境
Julia 推荐使用Juno——基于Atom扩展的编辑器。事实证明，这一说法是在现阶段还是有道理的。

因为我平常用VS Code更多，所以一开始是找了一个Julia插件准备来开发。但发现一直配置不成功，后来才找到原因是VS Code还没支持Julia 0.7以后版本（https://github.com/JuliaEditorSupport/julia-vscode/issues/537）

所以目前还是得老老实实的用Juno。这也从一个侧面说明，**目前Julia的一些生态支持还没跟上**。

## 安装
安装过程按照官网指导还是比较顺利：先安装Julia，然后再安装Atom上的Juno即可。所以详细过程此处略过。

## 其他准备
为了调试，也需要再准备一个Postman或类似工具。

# 1. 搭建区块链的整体结构
这里我们先定义好`Blockchain`的类型，代表区块链类型。其中会包含两个数组：`chain`表示链；`current_transactions`表示待处理交易

```
mutable struct Blockchain
    chain::Array{Block}
    current_transaction::Array{Transaction}
end

function init(blockchain::Blockchain)
    # TODO
end

function new_block(;blockchain::Blockchain, proof::Int64, previous_hash = nothing)
    # TODO
end

function new_transaction(;blockchain::Blockchain, sender::String, recipient::String, amount::Float64)
    # TODO
end

function blockhash(block::Block)
    # TODO
end
```

当时也不知道谁给我说过Julia语法很像Python。学了之后才发现，至少从语法层面看一点都不像。

通过上述代码也可以看到一些Julia特点：
- 没有class。Julia不算一种OOP语言，没有自带class这种类型。比较像的是struct
- 可以不指定变量类型，交给Julia来推测，但指定后肯定会加快执行速度。另外，像给函数参数也可以有一些默认的值，但需要放在`;`之后


# 2. 区块与交易
有了链之后，我们继续来定义区块和交易。每个区块的结构中主要会包括：
- 区块序号
- 时间戳，用Unix时间表示
- 交易列表，本区块内打包的交易数量
- 工作量证明
- 前一个区块的哈希值

而每个交易的结构定义的比较简单，只有以下3部分：
- 交易发送者
- 交易接受者
- 交易金额

具体代码实现如下：
```julia
struct Transaction
    sender::String
    recipient::String
    amount::Float64
end

mutable struct Block
    index::Int64
    timestamp::Int64
    transaction_list::Array{Transaction}
    proof::Int64
    previous_hash::String
end

```
通过上述的格式，我们就可以得到一个比较典型的块链式数据结构了。

# 3. 新增交易
有了基本的数据结构之后，下面我们从上述数据的最小单元开始来定义操作。

首先是新增交易。区块链上的交易一般要在待处理的交易队列里追加进去，并在函数最后返回将打包该交易的区块。
```julia
function new_transaction(;blockchain::Blockchain, sender::String, recipient::String, amount::Float64)
    new_tx = Transaction(sender, recipient, amount)
    push!(blockchain.current_transaction, new_tx)
    return length(blockchain.chain) + 1
end
```

这部分的Julia特点（坑）包括：
- 要追加元素的话，要记得用push这个函数，而不是append。因为Julia里的append其实是concatenate……
- 函数名后面的感叹号意思是允许函数修改传入的参数。所以在上面的例子里，我们必须得用push!

# 4. 新增区块
当需要处理交易时，就要将交易打包进区块了。而在能新增区块之前，得先生成一个创世区块。由于只需要生成一次创世区块，这一工作可以在初始化时来完成。

## 初始化
如果是在一般的OOP里，初始化函数可以在对象的constructor里写好；但Julia里没有这种机制，还是需要单独写一个初始化函数来做这部分工作，包括：
- 初始化链
- 初始化待处理的交易列表
- 新增创世区块。其中为了统一起见，创世区块也仍然需要工作量证明，也需要定义一个前区块哈希。在本程序里我们不妨将两者定义为100和"genesis_hash"
```julia
function init(blockchain::Blockchain)
    blockchain.chain = []
    blockchain.current_transaction = []
    new_block(blockchain=blockchain, proof=100, previous_hash="genesis_block_hash")
end
```
## 新增区块
上面代码里的`new_block()`这个函数就是我们用来新增区块的。参数中的前一个区块哈希可不输入，默认为链中最后一个区块的哈希值；新增区块只要按照上面定义的区块数据结构进行填写即可。最后把这个区块追加到目前区块的链中，并清空待处理交易列表，实现如下：

```julia
function new_block(;blockchain::Blockchain, proof::Int64, previous_hash = nothing)
    if previous_hash == nothing
        previous_hash = blockhash(blockchain.chain[end])
    end
    block = Block(length(blockchain.chain)+1, round(Int64, time()), blockchain.current_transaction, proof, previous_hash)

    push!(blockchain.chain, block)
    blockchain.current_transaction = []
    return block
end

```
上面这段的Julia特点包括：
- nothing，Julia里用来空值的一个常量
- Julia和其他很多科学计算类语言一样，下标是从1开始的
- end，表示数组中的最后一个元素下标。Julia不支持Python那种负数下标，所以相对而言不是那么灵活

## 哈希值的计算
在增加区块的时候还有一个要注意的是，需要计算最后一个区块的哈希值。这时候需要借助一些额外的包来完成了，这里用到了SHA和JSON：JSON用来将一个结构化的变量拍平为json格式；而SHA是根据这个json文本生成对应的哈希值，具体如下
```julia
using SHA, JSON
...
function blockhash(block::Block)
    return bytes2hex(sha256(JSON.json(block)))
end
```

增加包，在1.0里面在REPL环境中，输入`]`，进入pkg环境。pkg是Julia自带的包管理器，输入add命令即可，例如：
```
(v1.0) pkg> add JSON
```
也可以在REPL中通过下面这种方式来实现
```
using Pkg
Pkg.add("JSON")
```

另外，值得注意的是，Julia是支持函数式编程的，所以上面这个嵌套了多层的哈希函数完全可以用一种更现代化的形式来实现。

# 5. PoW
工作量证明，Proof of Work，是最为经典的生成新区块并可产生共识的一个协议。比特币里采用的方式是碰撞到一种特定的哈希值，例如包含了若干个连续0的哈希值；位数越多，难度越大。这里我们也用类似的方式来实现。

程序片段如下：
```julia
...
difficulty = Int8(4)
...
function proof_of_work(last_proof)
    proof = 0
    while valid_proof(last_proof, proof) == false
        proof += 1
    end
    return proof
end

function valid_proof(last_proof, proof)
    header = "0"^difficulty
    guess = string(last_proof) * string(proof)
    guess_hash = bytes2hex(sha256(guess))
    return guess_hash[1:difficulty] == header
end
```
这段代码里，我们先假定难度是4个0。在`proof_of_work()`这个函数里，我们循环去尝试nonce是否可以符合我们区块的要求，符合的话即返回该值，提供给产生区块的代码去打包。

验证难度值的函数则根据尝试的nonce来验证是否符合我们4个0的要求。

这段Julia代码和字符串、数组操作都比较相关：
- 要生成重复元素的时候，可以使用`^`这个操作符。例如例子里的`"0"^difficulty`
- 连接字符串，可以使用`*`
- 获取数组的一部分元素可以使用冒号，例如：`guess_hash[1:difficulty]`

# 6. 对外API接口
光定义好了函数还不够，还要能让外部（客户端、其他节点等）可以调用。在我们参考的Python实现中使用的是比较流行的Flask框架，实现了几个必要的Restful接口。我们这里也是类似的，首先需要有以下几个接口：
- `POST /transactions/new`给区块创建一个新交易
- `GET /mine`挖矿（产生区块）
- `GET /chain`返回当前所有的区块数据

Julia里其实也已经有不少web框架了，最有名的是Genie。目前Genie似乎还没有加入到官方包管理器索引中，如果要使用的话需要手动输入地址来添加：`Pkg.clone("https://github.com/essenciary/Genie.jl")`

但Genie实在太重，包含了太多不必要的程序部分。我们在实现区块链的过程中只要关心对外的几个Restful接口就够了，因此我后来直接使用了**Restful**这个包，实现轻量化的功能即可
```julia
...
using Restful, Logging
using UUIDs
...
const app = Restful.app()

    blockchain = Blockchain([], [], [])
    node_identifier = replace(string(uuid4()), "-" => "")
    init(blockchain)
    
    app.get("/mine", json) do req, res, route
    end

    app.post("/transactions/new", json) do req, res, route
    end

    app.get("/chain", json) do req, res, route
        res.json(Dict("chain"=>blockchain.chain, "length"=>length(blockchain.chain)) |> collect)
    end

    @async with_logger(SimpleLogger(stderr, Logging.Warn)) do
        app.listen("127.0.0.1", 3001)
    end
```
其中UUID是用来生成一个唯一的ID号用来标识本节点身份

## 交易接口
首先，我们按照交易变量的结构定义好报文接口：
```
{
 "sender": "付款人地址",
 "recipient": "收款人地址",
 "amount": 金额
}
```
接着用post方式来实现这一接口：
```julia
    app.post("/transactions/new", json) do req, res, route
        required = ["sender", "recipient", "amount"]
        data = JSON.parse(req.body)

        if all(i->(i in keys(data)), required) == false
            res.code(400)
        else
            block_id = new_transaction(blockchain=blockchain, sender=data["sender"], recipient=data["recipient"], amount=data["amount"])
            res.json(Dict("message" => "Transaction will be added to Block " * string(block_id)) |> collect)
        end
    end
```
从上面我们也可以看到：Julia是支持匿名函数的。所以这里我也用这个特性进行了快速判断JSON参数

## 挖矿接口

```julia
    app.get("/mine", json) do req, res, route
        last_block = blockchain.chain[end]
        last_proof = last_block.proof
        proof = proof_of_work(last_proof)

        new_transaction(blockchain=blockchain, sender="0", recipient=node_identifier, amount=1.0)

        previous_hash = blockhash(last_block)
        block = new_block(blockchain=blockchain, proof=proof, previous_hash=previous_hash)
        res.json(Dict("message" => "New Block Forged",
        "index" => block.index,
        "transaction" => block.transaction_list,
        "proof" => block.proof,
        "previous_hash" => block.previous_hash) |> collect)
    end
```
在打包交易进区块时，首先会进行大量哈希计算。得到nonce值后，在本案例中，我们再多追加一个发送者为0、接收者为本节点、金额为1的特殊交易。这其实就是表示挖矿奖励：
- 发送者为0，表示是一笔没有发送者的新增金额，即比特币中的coinbase。
- 接收者是本出块节点，代表挖矿奖励应计入到本节点的账户中
- 挖矿金额奖励为1。这个是简化后的固定值，当然也可以仿照比特币每4年减半来设置一个更复杂的运算。

最后，再调用之前我们定义好的`new_block`函数，完成区块生成的全部过程。

# 7. 与其他节点通信
以上为止，我们的区块链实际上还是单机版的，并没有涉及到与其他节点通讯的部分。这显然不符合实际情况，需要加上。
```julia
using HTTP, URIParser
...
mutable struct Blockchain
    ...
    nodes::Array{String}
end
...
function register_node(blockchain::Blockchain, address::String)
    push!(blockchain.nodes, address)
end
...
app.post("/nodes/register", json) do req, res, route
    data = JSON.parse(req.body)
    nodes = data["nodes"]

    if nodes == nothing
        res.code(400)
    end
    
    for node in nodes
        register_node(blockchain, node)
    end
    
    res.json(Dict("message" => "Nodes will be added to Blockchain") |> collect)
end
```
这里，我们在blockchain的数据结构上增加登记节点列表，以最简单粗暴的方式来实现“节点发现”的过程。

目前只有增加功能，因此只写了一个函数，将节点信息push进这个节点列表中，最后以`POST /nodes/register`的接口来提供外部调用

# 8. 用最长链的方式进行共识
有了不同节点之后就可以进行PoW竞争记账了。虽然PoW已经是一个比较具体的方法，但在实现上仍有比较多的策略，例如比特币是采用最长链方式、以太坊采用的是“最重子树”(Heaviest Subtree)的方式。这里我们使用的是类似比特币的只比较长度的简单方式。

```julia
function valid_chain(chain::Array{Block})
    last_block = chain[end]
    current_index = 1
    while current_index < length(chain)
        block = chain[current_index]
        if block.previous_hash != blockhash(last_block)
            return false
        end

        if valid_proof(last_block.proof, block.proof) == false
            return false
        end

        last_block = block
        current_index += 1
    end

    return true
end

function resolve_conflict(blockchain::Blockchain)
    neighbours = blockchain.nodes
    new_chain = nothing

    my_height = length(blockchain.chain)

    for node in neighbours
        response = HTTP.request("GET", "http://" * node * "/chain")

        if response.status == 200
            response_body = JSON.Parser.parse(String(response.body))
            height = response_body[1]["length"]
            chain = response_body[2]["chain"]

            if height > my_height && valid_chain(chain)
                my_height = height
                new_chain = chain
            end
        end
    end

    if new_chain != nothing
        blockchain.chain = new_chain
        return true
    end

    return false
end
...
app.get("/nodes/resolve", json) do req, res, route
    replaced = resolve_conflict(blockchain)
    if replaced == true
        res.json(Dict("message" => "Our chain was replaced") |> collect)
    else
        res.json(Dict("message" => "Our chain is main chain") |> collect)
    end
end
```

以上代码中，我们定义了一个接口`GET /nodes/resolve`来解决一致性问题，具体方式是调用到`resolve_conflict`函数，遍历所有已登记的节点，找到他们现在的链上区块，比较其长度，并通过`valid_chain`函数来检验其区块有效性。

而检验方法是通过哈希与nonce值计算后的比较进行，由于哈希运算的单向特性，所以这部分的运算也比较快。检验完成后，如果是长度比本节点自己的链上而且是有效的，则进行替换，在这个新链的基础上继续挖矿，完成达成一致性的过程。

# 9. 总结展望
以上，我们已基本完成一个区块链的原型：
- 可以收发交易
- 可以PoW共识

具体代码可在 https://github.com/HuLaTown/blockchain 上找到。

可以看到，经过上述开发，我们可以基本分析出一个开发语言的功能与生态情况，虽然“**区块链完备**”这个概念目前只是我拍脑袋的想法，还没有一个可以定量化的标准。

另外经过区块链原型开发，也可以快速熟悉一个开发语言的基本特性，包括变量类型、空值等特殊取值、字符串与数组处理、函数传参与修改、函数间的相互调用等语言特性，当然也不可避免的要用工具来debug，这些都对于后续用该语言开发会很有帮助。

但还有很多地方可以继续开发。这也是为啥是标题党的原因之二。这些要改进的地方至少包括：
- 代码可能还有一些未发现的bug
- 需要进行很多的输入逻辑校验判断
- 需要Kademlia等DHT节点维护功能
- 需要增加密码学身份认证，例如公私钥等
- 需要增加UTXO或账户的账本结构
- 需要增加智能合约支持
等等……

![第5步就留作作业](https://upload-images.jianshu.io/upload_images/12457563-09237b2f040e30c2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 参考资料
1. Learn Blockchains by Building One https://hackernoon.com/learn-blockchains-by-building-one-117428612f46
2. Julia 1.0 Documentation https://docs.julialang.org/en/v1/
3. https://learnxinyminutes.com/docs/julia/

