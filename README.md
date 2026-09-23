# CrealityPrint 实时视频流自签名证书过期问题缓解方案

本文档针对 CrealityPrint 切片软件内置 Web 资源中，因打印机上位机自签名 SSL 证书过期导致 WebRTC 实时视频流无法正常显示的故障进行全面分析，涵盖 **问题分析**、**影响范围** 以及 **缓解措施**。

---

## 一、问题背景与核心根因

### 1. 上位机自签名 SSL 证书有效期缺陷
在创想三维（Creality）K2 / K2 Pro / K2 Max 系列打印机的上位机（基于Linux系统）中，官方工程师为 HTTPS/WSS 加密通道配置了一张自签名证书（部署在libhv或go2rtc下）。
- **问题**：该自签名证书在开发时，错误地配置了仅 **1 年（365天）** 的有效期。
```
...

Signature Algorithm: sha256WithRSAEncryption
Issuer: CN=My Private CA, O=Internal Network, C=CN
Validity
    Not Before: Aug 29 03:23:23 2025 GMT
    Not After : Aug 29 03:23:23 2026 GMT
Subject: CN=My Server, O=Internal Service, C=CN

...
```
- **现状**：在2026年8月29日打印机可以继续正常工作，但证书过期生效（`CERT_DATE_INVALID` / `SSL_ERROR_CERT_EXPIRED`）。

### 2. 客户端严格校验与未做错误忽略
- CrealityPrint 使用的 C++ 底层网络请求库以及前端嵌入的 WebView（CEF / Qt WebEngine）在打印机控制及其他功能上使用的是HTTP通信，在2处WebRTC视频流中优先使用HTTPS/WSS进行通信，代码中有降级http/ws的逻辑但判断条件无法生效，且在进行 HTTPS 请求或 WSS 握手时，**没有开启忽略证书校验错误**（如未配置 `--ignore-certificate-errors` 或未实现 SSL 错误忽略的回调）。
- 官方在后续新版切片软件中修复了此问题，但修复此问题并不是通过rebase或patch补丁方式修复，这意味着通过更新的方式修复此问题，用户需要同时接受软件其他部分的改动和默认配置的更新风险。

### 3. 故障触发逻辑：`videoInfo.videoEncryption` 特性分流
在前端代码中，当打印机通过 mDNS/WebSocket 上报其特性支持 `videoInfo.videoEncryption`（视频加密传输）时，前端代码会产生协议分流：
- **开启加密**：强制将 WebRTC 本地 SDP 协商的目标 URL 构建为 `https://${ip}/call/webrtc_local`（走 443 端口），由 C++ 原生层发起请求。此时必将触发 443 端口自签名证书过期拦截，导致 SDP 交换彻底失败。
- **未开启加密**：前端将目标 URL 构建为 `http://${ip}:8000/call/webrtc_local`，并直接使用前端 `axios` / `bt.post` 向上位机的 `go2rtc` 服务发起明文 SDP 交换，**完全不经过 SSL/TLS 握手，因而不受证书过期的任何影响**。

---

## 二、受影响范围与主要表现

| 模块 / 页面 | 文件路径 | 对应 DOM 元素 | 故障表现 |
| :--- | :--- | :--- | :--- |
| **设备管理页 (Device Manager)** | `deviceMgr/assets/C8FZo4i0.js` | `<video id="remoteVideos">` (局域网直连)<br>`<video id="remoteVideo">` (创想云中继) | 视频区域一直显示转圈加载、黑屏，或反复弹出“正在重新连接 (Reconnecting)”，最终提示加载超时 |
| **提交打印页 (Send To Printer)** | `sendToPrinterPage/assets/DDpExr7T.js` | `<video id="remoteVideos">` (局域网直连)<br>`<video id="remoteVideo">` (创想云中继) | 切片后准备发送至打印机时，右侧预览窗中的摄像头画面无法打开，无法确认热床状态及延时摄影画面 |

---

## 三、缓解原理

打印机上位机的 `go2rtc` 流媒体服务底层同时监听两个端口：
1. **443 端口 (HTTPS)**：由上位机 libhv或Nginx作为SSL端点代理了go2rtc服务（理论推测，未root验证）。
2. **8000 端口 (HTTP)**：`go2rtc` 原生监听的明文端口，提供完全相同的 `/call/webrtc_local` SDP 协商能力，**无需证书握手**。

**缓解核心思路**：
- **局域网直连**：在前端强制将加密特性识别标志置为 `false`（或直接改写目标 URL），强制走 `http://${ip}:8000/call/webrtc_local` 明文通道，让切片软件前端直接通过明文 POST 交换 SDP。
- **创想云模式**：将中继信令连接的 `wss://` 协议降级为 `ws://`。(可选，创想云模式覆盖使用场景很小)

---

## 四、适用范围

适用于在不升级CrealityPrint软件的前提下进行临时缓解软件中视频流无法加载的问题。

> 提示：此缓解方案适用于家庭网络或可靠的安全网络环境，由于视频流接口在原厂设计下可以被匿名访问，即仅进行了传输加密，未对接口进行身份验证，也未使用TLS双向认证，虽然SDP Offer中具有信令交换机制，但前序9999端口wsslicer接口中也同样没有认证。因此降级为HTTP明文传输后，视频流在不安全的网络传输过程中被捕获报文，泄露打印机摄像头的拍摄内容。

---

## 五、缓解措施

> **提示**：建议在修改前先备份目标 JS 文件（如备份为 `.js.bak`）。

```
                                  代码修改索引总览
┌───────────────────────────────────────┬────────────────────────────────────────────────────────┐
│ 文件                                  │ 关键特征行 / 修改内容                                  │
├───────────────────────────────────────┼────────────────────────────────────────────────────────┤
│ 1. deviceMgr/assets/C8FZo4i0.js       │ • 第 108794-108795 行: 解除局域网 HTTPS 绑定并降级为 HTTP  │
│                                       │ • 第 108290 行: (可选) 云端 WebSocket wss:// 改为 ws:// │
├───────────────────────────────────────┼────────────────────────────────────────────────────────┤
│ 2. sendToPrinterPage/assets/DDpExr7T.js│ • 第 66151-66157 行: 局域网 SDP 请求禁用加密并指向 8000端口│
│                                       │ • 第 65991 行: 局域网 SDP 回调跳过原生加密分支走 HTTP POST  │
│                                       │ • 第 65655 行: (可选) 云端 WebSocket wss:// 改为 ws:// │
└───────────────────────────────────────┴────────────────────────────────────────────────────────┘
```

---

### 1. 设备管理页：`resources/web/deviceMgr/assets/C8FZo4i0.js`

#### 【核心修复点】局域网 WebRTC SDP 协商（第 108794 - 108795 行附近）

**查找特征关键字**：`videoInfo.videoEncryption` 或 `call/webrtc_local`

**原始代码**：
```javascript
const tr = dataCenter.getPrinter(re.address)
  , nr = Array.isArray(tr == null ? void 0 : tr.features) && tr.features.includes("videoInfo.videoEncryption")
  , sr = nr ? `https://${re.address}/call/webrtc_local` : `http://${re.address}:8000/call/webrtc_local`;
```

**修改后代码**：
```javascript
const tr = dataCenter.getPrinter(re.address)
  , nr = !1
  , sr = `http://${re.address}:8000/call/webrtc_local`;
```

> **修改效果**：
> 将 `nr`（加密标记）强制置为 `false`，并使 `sr` 强制指向 8000 端口 HTTP。此时底层不仅向 C++ 报告未加密，且在下面的 `MessageCenter$1.registerHandler` 中会触发未加密分支，由前端 `axios.post(sr, cr.sdp)` 自动完成明文 SDP 交换，彻底绕开 C++ 层的 HTTPS 证书拦截。

#### 【可选修改】云端 WebRTC WebSocket 信令连接（第 108290 行）

**查找特征关键字**：`api.crealitycloud.cn`

**原始代码**：
```javascript
, rr = `wss://${hr === "China" ? "api.crealitycloud.cn" : "api.crealitycloud.com"}/api/cxy/ws/webrtc/signal/pull/${V.value}/${j.value}`;
```

**修改后代码**：
```javascript
, rr = `ws://${hr === "China" ? "api.crealitycloud.cn" : "api.crealitycloud.com"}/api/cxy/ws/webrtc/signal/pull/${V.value}/${j.value}`;
```

---

### 2. 提交打印页：`resources/web/sendToPrinterPage/assets/DDpExr7T.js`

#### 【修复点 1】局域网 SDP 发送端协议与加密标志（第 66151 - 66157 行附近）

**查找特征关键字**：`sendOfferToCall` 或 `call/webrtc_local`

**原始代码**：
```javascript
sendOfferToCall: h => {
    h = g.cleanSDP(h);
    const p = l.device || {}
      , y = (Array.isArray(p.features) ? p.features : []).includes("videoInfo.videoEncryption")
      , _ = y ? `https://${p.address}/call/webrtc_local` : `http://${p.address}:8000/call/webrtc_local`
      , w = {
        sdp: h,
        url: _,
        videoEncryption: !!y
    };
    y && p.videoToken && (w.token = p.videoToken.trim()),
    Ft.getWebrtcLocalParam(w)
}
```

**修改后代码**：
```javascript
sendOfferToCall: h => {
    h = g.cleanSDP(h);
    const p = l.device || {}
      , y = !1
      , _ = `http://${p.address}:8000/call/webrtc_local`
      , w = {
        sdp: h,
        url: _,
        videoEncryption: !1
    };
    y && p.videoToken && (w.token = p.videoToken.trim()),
    Ft.getWebrtcLocalParam(w)
}
```

#### 【修复点 2】局域网 SDP 应答接收端跳过 C++ 原生解析（第 65991 行附近）

**查找特征关键字**：`h.command === "get_webrtc_local_param"`

**原始代码**：
```javascript
const f = h => {
    if (h.command === "get_webrtc_local_param") {
        const {data: p} = h
          , v = l.device || {}
          , _ = (Array.isArray(v.features) ? v.features : []).includes("videoInfo.videoEncryption")
          , w = b => {
```

**修改后代码**：
```javascript
const f = h => {
    if (h.command === "get_webrtc_local_param") {
        const {data: p} = h
          , v = l.device || {}
          , _ = !1
          , w = b => {
```

> **修改效果**：
> 在收到 C++ 返回的参数后，强制令 `_ = false`，从而不走原先由 C++ 请求本地 HTTPS 的原生应答通道，转而进入 `bt.post(`${p.url}`, p.sdp)`（Axios 明文 HTTP 请求通道），直接由 Web 容器与打印机的 8000 端口交换 SDP Answer。

#### 【可选修改点】云端 WebRTC WebSocket 信令连接（第 65655 行）

**查找特征关键字**：`webrtc/signal/pull`

**原始代码**：
```javascript
, X = `wss://${ee === "China" ? "api.crealitycloud.cn" : "api.crealitycloud.com"}/api/cxy/ws/webrtc/signal/pull/${i.value}/${s.value}`;
```

**修改后代码**：
```javascript
, X = `ws://${ee === "China" ? "api.crealitycloud.cn" : "api.crealitycloud.com"}/api/cxy/ws/webrtc/signal/pull/${i.value}/${s.value}`;
```

---

## 五、回退方式：
   若需恢复原始状态，只需将备份的 `.js.bak` 文件覆盖回原文件即可。

---

# 声明： 
- 实际修复以官方更新为最终方案。
- 此方案仅面向3D打印/网络技术学习用户，一切后果与本文无关。
- 代码各项权益由各自作者所有。