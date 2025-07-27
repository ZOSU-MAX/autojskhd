# AutoJS服务端对接文档

## 基础信息
- **服务端地址**: `http://localhost:3000` (HTTP API)
- **WebSocket地址**: `ws://localhost:8080` (实时通信)
- **通信协议**: HTTP/HTTPS, WebSocket/WSS
- **数据格式**: JSON

## 认证机制
### 设备注册认证
1. **流程**: 设备首次连接需注册并获取JWT令牌
2. **认证方式**: JWT (JSON Web Token) + 设备指纹
3. **安全机制**: AES-256-CBC加密 (敏感接口)

## HTTP API接口

### 1. 设备注册
- **URL**: `/api/device/register`
- **方法**: POST
- **请求体**: 
  ```json
  {
    "deviceId": "设备唯一标识符",
    "model": "设备型号",
    "androidVersion": "Android版本",
    "ip": "设备IP地址"
  }
  ```
- **响应**: 
  ```json
  {
    "deviceId": "服务器分配的设备ID",
    "token": "JWT认证令牌"
  }
  ```
- **备注**: 设备需保存返回的token用于后续认证

### 2. 心跳检测
- **URL**: `/api/device/heartbeat`
- **方法**: POST
- **请求头**: `Authorization: Bearer {token}`
- **请求体**: 
  ```json
  {
    "battery": 0.85
  }
  ```
- **响应**: 200 OK
- **频率**: 建议60秒一次

### 3. 脚本下发
- **URL**: `/api/script/push`
- **方法**: POST
- **请求头**: `Authorization: Bearer {token}`
- **请求体**: 
  ```json
  {
    "scriptId": "脚本ID",
    "deviceIds": ["设备ID列表"]
  }
  ```
- **响应**: 
  ```json
  {
    "success": true
  }
  ```

### 4. 批量设备操作
- **URL**: `/api/device/batch-action`
- **方法**: POST
- **请求头**: `Authorization: Bearer {token}`
- **请求体**: 
  ```json
  {
    "deviceIds": ["设备ID列表"],
    "action": "操作类型",
    "payload": {"参数": "值"}
  }
  ```
- **响应**: 
  ```json
  [
    {"deviceId": "id1", "status": "success"},
    {"deviceId": "id2", "status": "offline"}
  ]
  ```

## WebSocket通信
### 连接方式
- **URL**: `ws://localhost:8080?token={jwt_token}`
- **认证**: 连接时需在URL参数中携带JWT令牌

### 消息类型
| 类型 | 方向 | 描述 | 数据格式 |
|------|------|------|----------|
| `SCRIPT_PUSH` | 服务端→客户端 | 下发脚本 | `{type: 'SCRIPT_PUSH', script: {name, content, version}}` |
| `LOG` | 客户端→服务端 | 发送日志 | `{type: 'LOG', content: '日志内容'}` |
| `STATUS_UPDATE` | 客户端→服务端 | 状态更新 | `{type: 'STATUS_UPDATE', status: '运行状态'}` |
| `SCRIPT_RESULT` | 客户端→服务端 | 脚本执行结果 | `{type: 'SCRIPT_RESULT', result: '执行结果'}` |
| `SCREENSHOT` | 服务端→客户端 | 请求截图 | `{type: 'SCREENSHOT'}` |
| `SCREENSHOT_RESULT` | 客户端→服务端 | 返回截图 | `{type: 'SCREENSHOT_RESULT', image: 'base64图片'}` |

## 数据模型定义
### 设备信息 (Device)
```json
{
  "deviceId": "设备唯一ID",
  "model": "设备型号",
  "androidVersion": "Android版本",
  "ip": "IP地址",
  "status": "online/offline",
  "lastHeartbeat": "2023-11-01T12:00:00Z",
  "token": "JWT令牌"
}
```

### 脚本信息 (Script)
```json
{
  "name": "脚本名称",
  "version": "1.0.0",
  "content": "脚本内容",
  "targetDevices": ["设备ID数组"],
  "createdAt": "2023-11-01T12:00:00Z"
}
```

## 错误码说明
| 状态码 | 描述 | 可能原因 |
|--------|------|----------|
| 401 | 未授权 | token无效或过期 |
| 404 | 资源不存在 | 请求的设备或脚本不存在 |
| 500 | 服务器错误 | 服务端内部错误 |
| 400 | 请求参数错误 | 请求格式不正确 |

## 安全注意事项
1. 所有API请求需使用HTTPS (生产环境)
2. WebSocket需使用WSS加密连接
3. 敏感数据传输需使用AES-256-CBC加密
4. 设备token有效期为30天，需定期更新

## 示例代码片段
### 设备注册示例 (JavaScript)
```javascript
fetch('http://localhost:3000/api/device/register', {
  method: 'POST',
  headers: {'Content-Type': 'application/json'},
  body: JSON.stringify({
    deviceId: 'android_device_123',
    model: 'Xiaomi MI 11',
    androidVersion: '12',
    ip: '192.168.1.100'
  })
})
.then(res => res.json())
.then(data => {
  localStorage.setItem('deviceToken', data.token);
});
```

### WebSocket连接示例 (JavaScript)
```javascript
const token = localStorage.getItem('deviceToken');
const ws = new WebSocket(`ws://localhost:8080?token=${token}`);

ws.onopen = () => {
  console.log('WebSocket连接已建立');
};

ws.onmessage = (event) => {
  const message = JSON.parse(event.data);
  if (message.type === 'SCRIPT_PUSH') {
    executeScript(message.script);
  }
};
```