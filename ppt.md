title: Neo4J與Ice
speaker: 曾德雄
url: https://github.com/ksky521/nodeppt
transition: cards
files: /js/demo.js,/css/demo.css
theme: light

[slide]

# Neo4J與ZeroC.Ice

## 曾德雄

[程式github網址](https://github.com/cedriccefc2002/treediagram)

---

使用**方向踺**（ ↑ ↓ ← →）或**空白鍵**進行簡報翻頁

[slide]

# Neo4J

## 圖形資料庫

- 資料模型：[Property Graph Model](https://github.com/opencypher/openCypher/blob/master/docs/property-graph-model.adoc#pgm-definitions-node)
- Cypher 查詢語言
- 交易
- 其他功能

[slide]

# Neo4J-資料模型

## 實例(**Entity**)

- 節點：Node ()
- 關係：Relationship

## 標記(**Token**)

- 標籤：Label
- 屬性：Property

所有實例都可以有很多標籤與屬性

```cql
CREATE (node: Node)
SET
    node.uuid = $uuid,
    node.data = $data,
    node.root = $root,
    node.parent = $parent,
    node.isBinaryleft = $isBinaryleft
MATCH
    (p {uuid: $parent}),
    (n {uuid: $uuid})
CREATE (p)<-[:IsChild]-(n)
```

[slide]

# Neo4J-Cypher 查詢語言（不分大小寫）

- Parameters 設定參數避免注入攻擊
- Clauses
  - CREATE 建立
  - MATCH [WHERE] 查詢
  - SET 設定
  - RETURN 回傳
  - FOREACH  處理集合
  - WITH 定義
  - [DETACH] DELETE 刪除
- 函數：
  - collect 建立集合
  - endNode, startNode
  - head, last

[slide]

# Neo4J-Cypher 範例

```cql
MATCH p = (:Tree {uuid: $root})<-[r*0..]-(x:Node)
WITH
    collect(DISTINCT x) as nodes,
    [
        r in collect(DISTINCT last(r)) |
        {
            parentUUID: endNode(r).uuid,
            childUUID: startNode(r).uuid
        }
    ] as rels
RETURN size(nodes) AS nodesCount,size(rels) AS relsCount, nodes, rels
```

[slide]

# Neo4J-交易

- 寫入交易與讀取交易
- 叢集模式下將查詢其中一個伺服器

```csharp
using (var session = driver.Session())
{
    var uuid = Guid.NewGuid().ToString();
    logger.LogInformation($"{uuid}|{data}|{root}|{parent}|Start");
    var id = session.WriteTransaction((tx) =>
    {
        var parms = new { uuid, data, root, parent, isBinaryleft };
        var query = @"...";
        var result = tx.Run(query, parms);
        tx.Run(@"...", parms);
        return (result.Single())[0].As<string>();
    });
    logger.LogInformation($"{uuid}|{data}|{root}|{parent}|{id}");
    return true;
}
```

[slide]

# Neo4J-其他功能

- [企業版](https://neo4j.com/docs/operations-manual/3.4/introduction/#editions)（要收費）
  - Auto-reuse of space 資料庫佔用空間
  - 安全認證
  - 叢集與負載平衡
  - 在線備份

[slide]

# ZeroC.Ice

## 中間件平台 跨平台與程式語言

- Slice 規範語言
- **Glacier2** 穿越防火牆與callback
- 其他沒實做的功能

[slide]

# ZeroC.Ice Slice 規範語言

- 共用 Slice 文件 -> 目標語言原始碼 + 目標語言函式庫 -> 執行環境

```csharp
module TreeDiagram
{
    interface ServerEvent {
        void TreeListUpdate();
        void TreeUpdate(string uuid);
        void NodeUpdate(string uuid, string data);
    };

    interface Server
    {
        idempotent void moveNode(string uuid, string newParent);
        idempotent void deleteNode(string uuid);
        TreeView getNodeView(string uuid);
        void initEvent(ServerEvent* event);
    }
}
```

[slide]

# ZeroC.Ice Slice 橋接Class

```csharp
namespace Server.lib.IceBridge
{
    public class Server : TreeDiagram.ServerDisp_
    {
        private readonly ILogger<Server> logger;
        private readonly IList<ServerEventPrxHelper> clients = new List<ServerEventPrxHelper>();

        private readonly Service.ServerService service;
        public Server(ILogger<Server> logger)
        {
            this.logger = logger;
            service = lib.Provider.serviceProvider.GetRequiredService<Service.ServerService>();
            var eventService = lib.Provider.serviceProvider.GetRequiredService<Service.EventService>();
            eventService.evtTreeListUpdate += new Service.TreeListUpdateDelegate(TreeListUpdateHandler);
            eventService.evtTreeUpdate += new Service.TreeUpdateDelegate(TreeUpdateHandler);
            eventService.evtNodeUpdate += new Service.NodeUpdateDelegate(NodeUpdateHandler);
        }
    }
    public override void createTree(Tree tree, Current current)
    {
        logger.LogInformation($"{tree.uuid}");
        service.createTree(new Model.TreeModel()
        {
            uuid = tree.uuid,
            type = tree.type == TreeType.Binary ? Model.TreeType.Binary : Model.TreeType.Normal
        }).Wait();
    }
}
```

[slide]

# ZeroC.Ice Slice 服務

```csharp
public class IceService : IceBox.Service, IDisposable
    {
        private Ice.ObjectAdapter adapter;
        public async static Task StartIceService(string[] args)
        {
            Console.WriteLine("StartIceServer");
            await Task.Delay(0);
            var config = Config.IceAdapterConfig.Config;
            Console.WriteLine($"IceServer.name = \"{config.name}\"");
            using (Ice.Communicator communicator = Ice.Util.initialize(ref args))
            using (IceService iceService = new IceService())
            {
                iceService.start(config.name, communicator, args);
                await Task.Factory.StartNew(() =>
                {
                    communicator.waitForShutdown();
                });
            }
        }

        private void AddService(ref Ice.ObjectAdapter adapter)
        {
            var provider = lib.Provider.serviceProvider;
            adapter.add(provider.GetRequiredService<lib.IceBridge.Server>(), Ice.Util.stringToIdentity("Server"));
        }

        public void start(string name, Communicator communicator, string[] args)
        {
            var config = Config.IceAdapterConfig.Config;
            Console.WriteLine($"IceServer.endpoints = \"{config.endpoints}\"");
            adapter = communicator.createObjectAdapterWithEndpoints(name, config.endpoints);
            AddService(ref adapter);
            adapter.activate();
        }
    }
```

[slide]

# ZeroC.Ice Glacier2

## Client <-> [外網 glacier2 內網] <-> Server

執行

```sh
glacier2router --Ice.Config=src/glacier2.conf
```

glacier2.conf 檔案

```conf
Glacier2.InstanceName=PublicRouter
Glacier2.Client.Endpoints=tcp -p 10001
# Server 可以透過 Glacier2 主動呼叫用戶端
Glacier2.Server.Endpoints=tcp -h 127.0.0.1
#忽略認證
Glacier2.PermissionsVerifier=PublicRouter/NullPermissionsVerifier
Glacier2.Client.ForwardContext=1
Glacier2.Server.ForwardContext=1
```

[slide]

# ZeroC.Ice Glacier2 Client

```typescript
class ClientEvent extends TreeDiagram.ServerEvent {}

// 初始化通訊物件時加入Router宣告
const communicator = Ice.initialize([
    "--Ice.Default.Router=PublicRouter/router:tcp -h 127.0.0.1 -p 10001",
    "--Ice.ACM.Client.Heartbeat=3",
]);

const router = await Glacier2.RouterPrx.checkedCast(
    this.communicator.getDefaultRouter()
);

// 建立Router通訊
await router.createSession("username", "password");

const category = await router.getCategoryForClient();
const proxy = communicator.stringToProxy("Server:tcp -h 127.0.0.1 -p 10000");
const serverEvent = TreeDiagram.ServerEventPrx.uncheckedCast(
    adapter.add(
        new ClientEvent(), 
        new Ice.Identity("serverEvent", category)
    )
);
const server = await TreeDiagram.ServerPrx.checkedCast(proxy);

await server.initEvent(serverEvent);

```

[slide]

# ZeroC.Ice 其他沒實做的功能

- IceStorm
- IceGrid
- IcePatch

[slide]

# ZeroC.Ice 與 gRPC 比較

|   | ICE  | gRPC |
|---|---   |  --- |
| 代理與用戶端推播  | Glacier|  |
| 叢集  | IceGrid|  |
| 服務  | IceBox|  |
| 通訊  |   tcp｜udp   |  HTTP/2    |
| 支援語言  | C++, C#, Java, JavaScript, Objective-C, Python, PHP, Ruby. | 多 Go 與 Dart|
| 規範語言  | Slice| ProtoBuf |

---
[gRPC實做Load Balancing官方建議](https://github.com/grpc/grpc/blob/master/doc/load-balancing.md)

[slide]

# 報告完畢

## 敬請不吝斧正