HTTP状态码是服务器在响应客户端请求时返回的三位数字代码，用于表示请求的处理结果。以下是常见的HTTP状态码及其详细说明：

---

### **1xx 信息响应**
- **100 Continue**  
  服务器已收到请求头，客户端应继续发送请求体（用于大文件上传前确认）。

- **101 Switching Protocols**  
  服务器同意切换协议（如从HTTP升级到WebSocket）。

---

### **2xx 成功**
- **200 OK**  
  请求成功，响应中包含请求的数据（如GET请求返回资源）。

- **201 Created**  
  资源已成功创建（通常用于POST请求后返回新资源的URL，如新增用户）。

- **202 Accepted**  
  请求已接受但尚未处理完成（适用于异步任务，如后台处理）。

- **204 No Content**  
  请求成功，但响应无内容（如DELETE请求成功后的响应）。

---

### **3xx 重定向**
- **301 Moved Permanently**  
  资源永久重定向到新URL，客户端应更新书签（SEO权重会转移）。

- **302 Found**  
  资源临时重定向到新URL，客户端后续仍用原URL访问。

- **304 Not Modified**  
  资源未修改，客户端可使用本地缓存（配合`If-Modified-Since`等头使用）。

- **307 Temporary Redirect**  
  临时重定向，且必须保持原请求方法（如POST请求不会变为GET）。

- **308 Permanent Redirect**  
  永久重定向，且必须保持原请求方法。

---

### **4xx 客户端错误**
- **400 Bad Request**  
  请求格式错误（如参数错误、JSON解析失败）。

- **401 Unauthorized**  
  需要身份验证（如未提供有效Token）。

- **403 Forbidden**  
  服务器拒绝请求（如用户无权访问资源）。

- **404 Not Found**  
  请求的资源不存在（URL错误或资源已删除）。

- **405 Method Not Allowed**  
  请求方法不被允许（如用GET访问只支持POST的接口）。

- **408 Request Timeout**  
  服务器等待请求超时（客户端需重新发送请求）。

- **409 Conflict**  
  请求与服务器当前状态冲突（如重复创建相同资源）。

- **429 Too Many Requests**  
  客户端请求过于频繁（触发限流策略）。

---

### **5xx 服务器错误**
- **500 Internal Server Error**  
  服务器内部错误（通用错误，如代码异常）。

- **501 Not Implemented**  
  服务器不支持请求的功能（如未实现的API）。

- **502 Bad Gateway**  
  网关或代理服务器收到无效响应（如上游服务宕机）。

- **503 Service Unavailable**  
  服务器暂时不可用（如过载或维护中）。

- **504 Gateway Timeout**  
  网关或代理服务器未及时获取响应（上游服务超时）。

- **505 HTTP Version Not Supported**  
  服务器不支持请求的HTTP协议版本。

---

### **其他特殊状态码**
- **418 I'm a teapot**  
  彩蛋状态码（源自愚人节玩笑，表示服务器是“茶壶”无法煮咖啡）。

---

### **常见场景示例**
- **登录失败** → 401 Unauthorized  
- **访问未授权页面** → 403 Forbidden  
- **提交表单参数缺失** → 400 Bad Request  
- **删除资源成功但无需返回内容** → 204 No Content  
- **网站更换域名** → 301 Moved Permanently  

理解这些状态码能帮助开发者快速定位问题，优化用户体验和系统调试。