# Claude Code 的 MCP 集成：不是"支持协议"，而是完整客户端实现

MCP 这块原本看似普通，结果越深究越发现这是 Claude Code 里最成熟的系统之一。它不是"顺便兼容一下 MCP"，而是真把自己做成了一个很完整的 MCP 客户端：多传输层、OAuth、重连、工具发现、资源与命令刷新、权限整合，全都有。

## 1. 核心枢纽在 `src/services/mcp/client.ts`

MCP 连接状态在 `src/services/mcp/types.ts` 里是个联合类型：

```typescript
export type MCPServerConnection =
  | ConnectedMCPServer
  | FailedMCPServer
  | NeedsAuthMCPServer
  | PendingMCPServer
  | DisabledMCPServer
```

这个设计很关键。很多客户端只区分"连上/没连上"，Claude Code 明显不是。它需要：

- UI 能显示认证中、失败中、重连中
- 会话逻辑知道哪些 server 可用，哪些只是待授权
- 自动重连知道如何从失败态重新爬起来

连接结果还做了 memoize，按 server name + config 缓存。  
也就是说，同一台服务器不会被重复建立多个连接，除非 onclose 时显式清掉缓存。

这很重要。MCP server 往往不是纯内存对象，可能背后有子进程、网络连接、OAuth token、工具列表。如果每处调用都重新连一次，系统会很快乱套。

## 2. 传输层支持得非常全

`TransportSchema` 列出来的类型包括：

```typescript
z.enum(['stdio', 'sse', 'sse-ide', 'http', 'ws', 'sdk'])
```

这几种传输拆开看，能明显感受到不是"名义支持"，而是各自处理了很多现实世界细节。

### `stdio`

最常见的本地模式。Claude Code 会把 server 当子进程拉起，传环境变量、处理 stderr、保留调试信息，还要避免 stderr 把 UI 弄花。

### SSE

老一代远端 MCP transport。这里有个细节：  
普通 fetch 走超时包装，但 SSE 的长连接 fetch **不能**共用那个超时逻辑，否则 60 秒后自己把连接掐了。

### Streamable HTTP

这个实现认真遵循了 MCP 的 Streamable HTTP 规范，包括 POST 请求上的 `Accept: application/json, text/event-stream`。

### WebSocket

既支持 IDE 侧的 `ws-ide`，也支持一般 `ws`，还专门适配 Bun/Node 的 WebSocket 差异。

### SDK 控制通道

这个很有意思。它不是传统网络传输，而是把 MCP JSON-RPC 包进 CLI 与 SDK 进程之间的控制消息里。  
等于是自己造了一条"进程内外桥接 transport"。

### In-Process

为了避免某些重型 server 子进程太贵，Claude Code 还支持进程内 transport。`createLinkedTransportPair()` 用两个互相连接的对象模拟一对端点，消息通过 `queueMicrotask` 投递。

这说明他们在 MCP 这里追求的不只是"兼容"，而是**统一抽象下的多种部署形态**。

## 3. 配置来源是分层合并的

MCP server 配置可能来自：

1. 插件
2. 用户设置
3. 项目 `.mcp.json`
4. 本地 `.claude/settings.local.json`

企业托管配置存在时，其他来源甚至会被整体压掉。

这里最重要的一点是：项目级 `.mcp.json` 需要显式批准。  
这很好理解，因为一个陌生仓库如果能靠 `git clone` 就自动接入任意 MCP server，那风险太离谱了。

环境变量展开也在配置加载阶段统一处理，支持 `${VAR}` 和 `${VAR:-default}`。这让配置文件既灵活，又不需要每种 transport 自己重复做展开。

## 4. 连接管理不是一次性动作，而是持续编排

`useManageMCPConnections` 负责整个生命周期：

- 读配置
- 分批连接
- 更新状态
- 监听工具/资源/命令列表变更
- 自动重连

本地 server 按小批量连，远端 server 按大批量连，状态刷新还用 16ms flush timer 批处理，避免一堆 server 同时上线时把状态树抖成筛子。

这个思路挺成熟的：MCP 不是"连成功就完"，而是长期存活对象，要考虑波动、抖动和重连风暴。

## 5. 健康监控和重连明显是线上产物

`client.ts` 对各类错误做了很多专门处理：

- `ECONNRESET`
- `ETIMEDOUT`
- `EPIPE`
- `EHOSTUNREACH`
- `ECONNREFUSED`

还会数连续 terminal errors，达到阈值就强制 close，逼连接重建。

自动重连本身也有指数退避：

- 最多 5 次
- 从 1s 起步
- 上限 30s

只有远端 transport 才会自动重连，`stdio` 通常不重连，因为子进程都没了，再原地等没有意义。

这种区别对待很重要。不同 transport 的失败语义根本不一样，统一处理反而容易出问题。

## 6. 工具发现之后，会被包装成 Claude Code 原生工具

服务端 `tools/list` 回来后，Claude Code 会把每个 MCP tool 转成内部 `Tool` 对象。命名规则是：

`mcp__<serverName>__<toolName>`

这样做有几个好处：

- 避免和内建工具重名
- 权限规则可以直接按 server 前缀写
- 系统提示词里能明确区分工具来源

更有意思的是，MCP tool 的 annotations 会直接影响行为：

- `readOnlyHint` 决定是否并发安全 / 只读
- `destructiveHint` 决定是否破坏性操作
- `openWorldHint` 决定是否面向开放世界
- `anthropic/searchHint` / `anthropic/alwaysLoad` 影响 ToolSearch 延迟加载策略

也就是说，Claude Code 没把 MCP tool 当"黑盒外来物"，而是尽量把它们映射进自己既有的工具语义系统。

## 7. OAuth 集成做得比预期重很多

`src/services/mcp/auth.ts` 这一块，不只是"带个 bearer token"，而是完整的 OAuth client provider：

- authorization code + PKCE
- 元数据发现
- token refresh
- step-up auth 检测
- 系统 secure storage 存 token
- 特定 vendor 的错误码兼容

甚至还有 Cross-App Access / IdP token exchange 的企业场景支持。

这里最能体现 production 味道的一点，是它对各种"非标准实现"的妥协：  
比如某些厂商的 refresh error 不按 RFC 命名，Claude Code 会归一化成标准错误再走统一处理逻辑。

这不是好看不好看的问题，而是真接过外部系统就知道，协议兼容往往不是规范问题，而是生态脏活。

## 8. 安全上，它也没把 MCP 放在权限系统外面

MCP tool 的 `checkPermissions()` 返回 `passthrough`，意味着最终还是交给 Claude Code 的统一权限系统。

这点很合理。因为不管工具来自内建还是 MCP，从用户视角看，都是"模型要执行一个动作"。  
权限体验和安全边界不能因为来源不同就两套逻辑。

企业管理员也可以通过 allowlist / denylist 控 MCP server，支持：

- 名字匹配
- 命令匹配
- URL 模式匹配

而且 denylist 优先级最高。

## 9. 对这套 MCP 设计的整体判断

Claude Code 的 MCP 集成之所以显得成熟，是因为它同时做对了三件事：

1. 传输层抽象做得够完整  
   本地、远端、IDE、进程内、SDK 控制桥都能纳入同一模型。

2. 生命周期管理做得够重  
   连接、认证、失败、刷新、重连、列表变化都被认真对待。

3. 和现有工具/权限体系融合得够深  
   MCP tool 不是外挂，而是进入 Claude Code 自己的运行时。

如果非要说缺点，主要还是复杂度：

- `client.ts` 超大，理解成本不低
- transport、auth、状态同步交织较多
- 第三方 server 的不规范实现会不断逼客户端长出补丁

但客观而言，这些几乎是做成熟 MCP 客户端的必经代价。

## 10. 结论

看完这套实现，感受很简单：Claude Code 并不是"顺便支持了 MCP"，而是已经把 MCP 当成一条正式扩展总线。

它支持的不只是协议握手，而是整套现实世界问题：

- 服务器怎么发现
- 权限怎么接
- 认证怎么跑
- 出错怎么恢复
- 工具怎么进入模型上下文
- 远端变化怎么同步回来

这才是一个真正可用的 MCP 客户端该有的样子。
