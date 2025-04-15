以下是常见的 Web 安全攻击类型及其详细概念和解决方案，涵盖攻击原理、示例和防御策略：

---

### **1. SQL 注入（SQL Injection）**
#### **概念**  
攻击者通过用户输入（如表单字段）插入恶意 SQL 代码，篡改数据库查询逻辑，窃取或破坏数据。
#### **攻击示例**  
```sql
输入用户名：admin' OR '1'='1
生成的 SQL：SELECT * FROM users WHERE username = 'admin' OR '1'='1' AND password = 'xxx'
```
#### **防御方案**  
- **参数化查询（Prepared Statements）**：使用预编译语句绑定参数（如Java的`PreparedStatement`）。  
- **ORM 框架**：使用Hibernate、Sequelize等工具避免手动拼接SQL。  
- **输入验证**：过滤特殊字符（如单引号、分号）。  
- **最小权限原则**：数据库账户仅分配必要权限。  
- **Web 应用防火墙（WAF）**：拦截恶意SQL特征。

---

### **2. 跨站脚本攻击（XSS, Cross-Site Scripting）**
#### **概念**  
攻击者在网页中注入恶意脚本（JavaScript），当其他用户访问时执行，窃取Cookie或会话信息。  
#### **分类**  
- **存储型 XSS**：脚本存储于数据库（如评论内容）。  
- **反射型 XSS**：脚本通过URL参数传递并反射到页面。  
- **DOM 型 XSS**：通过修改DOM树触发（如`document.write`）。  
#### **防御方案**  
- **输出转义**：对用户输入的内容进行HTML转义（如`<`转义为`&lt;`）。  
- **Content Security Policy (CSP)**：通过HTTP头限制脚本来源。  
- **HttpOnly Cookie**：防止JavaScript读取敏感Cookie。  
- **输入过滤**：禁止`<script>`等标签或特殊属性。

---

### **3. 跨站请求伪造（CSRF, Cross-Site Request Forgery）**
#### **概念**  
诱导用户访问恶意页面，利用已登录的身份发起伪造请求（如转账、修改密码）。  
#### **攻击示例**  
```html
<img src="https://bank.com/transfer?to=attacker&amount=1000">
```
#### **防御方案**  
- **CSRF Token**：服务器生成随机Token嵌入表单，提交时验证。  
- **SameSite Cookie**：设置Cookie的`SameSite`属性为`Strict`或`Lax`。  
- **验证 Referer 头**：检查请求来源是否合法域名。  
- **关键操作二次验证**：如短信验证码、密码确认。

---

### **4. 点击劫持（Clickjacking）**
#### **概念**  
通过透明iframe覆盖诱骗用户点击隐藏按钮（如授权按钮）。  
#### **防御方案**  
- **X-Frame-Options 头**：设置为`DENY`或`SAMEORIGIN`。  
- **CSP 的 frame-ancestors 指令**：限制页面被嵌入的域名。  
- **JavaScript 防御**：通过脚本检查是否被嵌套（如`if (top != self) top.location = self.location;`）。