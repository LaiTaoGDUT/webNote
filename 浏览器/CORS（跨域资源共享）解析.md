### CORS（跨域资源共享）深度解析

CORS（Cross-Origin Resource Sharing）是现代浏览器支持的标准跨域解决方案，允许服务器声明哪些外部源（域、协议、端口）可以访问其资源。以下是 CORS 的核心机制和实现细节：

---

#### 1. **CORS 的核心原理**
   - **安全模型**：由浏览器强制实施，但需服务端配合响应特定 HTTP 头部。
   - **关键角色**：
     - **浏览器**：拦截跨域请求，检查服务端返回的权限头。
     - **服务端**：通过响应头（如 `Access-Control-Allow-Origin`）声明允许的源、方法、头等。

---

#### 2. **CORS 的两种请求类型**

##### **2.1 简单请求（Simple Request）**
   - **触发条件**：
     - 请求方法为 `GET`、`POST` 或 `HEAD`。
     - 请求头仅包含以下字段：
       - `Accept`、`Accept-Language`、`Content-Language`、`Content-Type`。
       - `Content-Type` 的值仅限于 `text/plain`、`multipart/form-data`、`application/x-www-form-urlencoded`。
   - **流程**：
     1. 浏览器直接发送请求，并在请求头中添加 `Origin`（如 `Origin: http://your-site.com`）。
     2. 服务端响应时，需包含以下头部：
        ```http
        Access-Control-Allow-Origin: http://your-site.com  // 或 *（不推荐）
        Access-Control-Expose-Headers: X-Custom-Header     // 可选：允许前端访问的自定义头
        ```
     3. 浏览器验证 `Access-Control-Allow-Origin` 是否匹配当前源，若匹配则允许前端读取响应。

##### **2.2 预检请求（Preflight Request）**
   - **触发条件**：
     - 使用 `PUT`、`DELETE` 等非简单方法。
     - 包含自定义请求头（如 `Authorization`）。
     - 简单方法但`Content-Type` 为 `application/json` 等非简单值。
   - **流程**：
     1. 浏览器先发送 `OPTIONS` 预检请求，询问服务器是否允许实际请求。
        ```http
        OPTIONS /api/data HTTP/1.1
        Origin: http://your-site.com
        Access-Control-Request-Method: PUT
        Access-Control-Request-Headers: Content-Type, Authorization
        ```
     2. 服务端响应预检请求，需返回以下头部：
        ```http
        Access-Control-Allow-Origin: http://your-site.com
        Access-Control-Allow-Methods: PUT, POST, GET      // 允许的方法
        Access-Control-Allow-Headers: Content-Type, Authorization  // 允许的头
        Access-Control-Max-Age: 86400                   // 预检结果缓存时间（秒）
        ```
     3. 浏览器确认权限后，发送实际请求（如 `PUT`）。
     4. 服务端需在实际响应中再次包含 `Access-Control-Allow-Origin`。

---

#### 3. **CORS 的关键 HTTP 头部**

| **请求头（客户端）**             | **响应头（服务端）**                   | 作用                                                                 |
|----------------------------------|----------------------------------------|----------------------------------------------------------------------|
| `Origin: http://your-site.com`   | `Access-Control-Allow-Origin: *`       | 声明允许访问的源（`*` 表示任意源，但不可与 `Credentials` 共用）。     |
| `Access-Control-Request-Method` | `Access-Control-Allow-Methods: GET`    | 预检请求中声明实际请求的方法以及服务器允许跨域请求的方法。                                        |
| `Access-Control-Request-Headers`| `Access-Control-Allow-Headers: X-Auth` | 预检请求中声明实际请求携带的自定义头以及服务器允许跨域请求携带的自定义头。                                |
| -                                | `Access-Control-Expose-Headers: X-ID`  | 允许前端 JavaScript 访问的额外响应头（默认只能读取部分基础头）。      |
| -                                | `Access-Control-Allow-Credentials: true`| 允许发起请求时携带接口所在域名的 Cookie 或 HTTP 认证信息，以及允许请求返回时设置接口所在域名的 Cookie 或 HTTP 认证信息，不可与`Access-Control-Allow-Origin: *`一起用（需前端设置 `withCredentials: true`）。|

---

#### 4. **CORS 实现步骤（以不同框架为例）**

##### **4.1 Node.js（Express）**
   ```javascript
   const express = require('express');
   const app = express();

   // 中间件处理 CORS
   app.use((req, res, next) => {
     res.header('Access-Control-Allow-Origin', 'http://your-frontend.com'); // 指定允许的源
     res.header('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE, OPTIONS');
     res.header('Access-Control-Allow-Headers', 'Content-Type, Authorization');
     res.header('Access-Control-Allow-Credentials', 'true'); // 允许携带 Cookie
     res.header('Access-Control-Max-Age', '86400'); // 预检缓存 24 小时

     // 直接放行预检请求
     if (req.method === 'OPTIONS') {
       return res.sendStatus(200);
     }
     next();
   });

   // 业务路由
   app.get('/api/data', (req, res) => {
     res.json({ data: 'Hello CORS!' });
   });
   ```

##### **4.2 Spring Boot（Java）**
   ```java
   @Configuration
   public class CorsConfig implements WebMvcConfigurer {
     @Override
     public void addCorsMappings(CorsRegistry registry) {
       registry.addMapping("/api/**")
         .allowedOrigins("http://your-frontend.com")
         .allowedMethods("GET", "POST", "PUT", "DELETE")
         .allowedHeaders("Content-Type", "Authorization")
         .allowCredentials(true)
         .maxAge(86400);
     }
   }
   ```

##### **4.3 Django（Python）**
   使用 `django-cors-headers` 库：
   ```python
   # settings.py
   INSTALLED_APPS = [
     'corsheaders',
     # ...
   ]
   MIDDLEWARE = [
     'corsheaders.middleware.CorsMiddleware',
     # ...
   ]
   CORS_ALLOWED_ORIGINS = ["http://your-frontend.com"]
   CORS_ALLOW_METHODS = ['GET', 'POST', 'PUT', 'DELETE']
   CORS_ALLOW_HEADERS = ['Content-Type', 'Authorization']
   CORS_ALLOW_CREDENTIALS = True
   ```

---

#### 5. **携带凭证（Cookies/HTTP Auth）**
   - **前端配置**：
     ```javascript
     fetch('http://api.example.com/data', {
       method: 'GET',
       credentials: 'include' // 携带 Cookie
     });
     ```
   - **服务端要求**：
     - 必须设置 `Access-Control-Allow-Credentials: true`。
     - **不能使用 `Access-Control-Allow-Origin: *`**，必须明确指定域名（如 `http://your-frontend.com`）。

---

#### 6. **常见问题与调试**
   - **错误提示**：
     - `No 'Access-Control-Allow-Origin' header is present`：服务端未正确配置 CORS。
     - `Response to preflight request doesn't pass access control check`：预检响应头缺失或权限不足。
   - **调试工具**：
     - 浏览器开发者工具（Network 标签查看请求/响应头）。
     - `curl` 或 Postman 模拟预检请求：
       ```bash
       curl -X OPTIONS -H "Origin: http://your-frontend.com" -H "Access-Control-Request-Method: PUT" http://api.example.com/data
       ```

---

#### 7. **CORS 的安全最佳实践**
   - 避免使用 `Access-Control-Allow-Origin: *`，除非是公开 API。
   - 限制 `Access-Control-Allow-Methods` 和 `Access-Control-Allow-Headers` 为最小必要集合。
   - 启用 `Access-Control-Max-Age` 减少预检请求次数。
   - 对敏感操作（如携带 Cookie）严格校验 `Origin`，防止 CSRF 攻击。

---

### 总结
CORS 是跨域问题的标准化解决方案，通过服务端设置响应头与浏览器协作实现安全控制。理解简单请求与预检请求的区别、合理配置头部、注意凭证安全性是关键。结合具体框架的中间件或库，可以快速实现安全可靠的跨域支持。