## Web漏洞

### 1. 目标X-Content-Type-Options响应头缺失

HTTP X-Content-Type-Options 可以对 script 和 styleSheet 在执行是通过MIME 类型来过滤掉不安全的文件。

nginx中增加如下配置：

```nginx
add_header X-Content-Type-Options nosniff;
```

注意：该项设置可能会导致IE9及以上版本拒绝加载没有返回Content-Type的资源，因此需要在业务服务端响应时均进行返回响应头。

### 2. 目标X-XSS-Protection响应头缺失

防止跨站脚本 Cross-site scripting (XSS)

```nginx
add_header X-XSS-Protection "1; mode=block";
```

### 3. 目标Content-Security-Policy响应头缺失

CSP 可以帮助防止跨站脚本攻击 (XSS) 和数据注入攻击。CSP 可以限制网页中可执行的脚本源、样式表源、图片源等资源的来源，从而确保这些资源只能从预期的服务器加载。如果网页中存在来自未知或不受信任的来源的脚本或资源，CSP 将阻止这些资源的加载，从而保护网页的安全性。

```nginx
add_header Content-Security-Policy "default-src 'self' https://apis.map.qq.com/  https://mapapi.qq.com/ data: blob: 'unsafe-eval' 'unsafe-inline'";
```

### 4. 目标Strict-Transport-Security响应头缺失

Strict-Transport-Security（HSTS）是一个HTTP响应头，用于通知浏览器应该始终使用 HTTPS 连接与网站通信，以确保数据在传输过程中始终得到加密保护。使用HSTS可以减少网站受到中间人攻击的风险，因为攻击者无法使用SSLstripping攻击来转换加密连接为非加密连接。

当浏览器首次访问支持HSTS的网站时，网站会向浏览器发送包含 HSTS 响应头的 HTTP 响应。该头包含一个指定的时间值（最小值为180天），告诉浏览器将来访问该网站时始终使用HTTPS。浏览器将缓存该头，以便在未来的访问中始终使用 HTTPS。

```nginx
## 即浏览器应该在将来的31536000秒内始终使用 HTTPS，并且应该包括所有子域名。preload 参数表示网站已经被添加到HSTS预加载列表中。
## "always" 参数表示响应头应该始终发送，即使请求使用HTTP时也是如此。

add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
```

### 5. 目标Referrer-Policy响应头缺失

Referrer-Policy 是一个 HTTP 响应头，用于控制浏览器在请求资源时是否应该发送 Referrer。Referrer 是指当前页面的 URL，浏览器在请求资源时会自动发送 Referrer。Referrer-Policy 可以控制发送的 Referrer 的信息量和是否发送 Referrer。

Referrer-Policy 响应头的值可以是以下之一：

- no-referrer：浏览器不发送 Referrer。
- no-referrer-when-downgrade：默认选项。如果从 HTTPS 页面链接到 HTTP 页面，浏览器不发送 Referrer。否则，浏览器发送完整的 Referrer。
- origin：只发送源的信息，即 URL 的协议、主机和端口号。例如，如果当前页面的 URL 是 https://example.com/path，而请求资源的 URL 是 https://example.com/otherpath，那么浏览器会发送 Referrer: [https://example.com。](https://example.com./)
- origin-when-cross-origin：默认选项。如果请求的资源与当前页面的源相同，则发送完整的 Referrer；否则，只发送源的信息。
- same-origin：只有当请求的资源与当前页面的源相同时，才发送完整的 Referrer。否则，不发送 Referrer。
- strict-origin：只发送源的信息，无论是同源请求还是跨源请求。与 origin 相同，但不会将 Referrer 发送到不安全的目标（例如 HTTP）。这是 Chrome 的默认设置。
- strict-origin-when-cross-origin：与 strict-origin 类似，但是只有当请求的目标 URL 与当前页面的源不同且是安全的（HTTPS）时，才发送源的信息。否则，不发送 Referrer。
- unsafe-url：始终发送完整的 Referrer，无论请求的资源是否跨域。

```nginx
add_header Referrer-Policy "no-referrer-when-downgrade";
```

### 6. 目标X-Permitted-Cross-Domain-Policies响应头缺失

X-Permitted-Cross-Domain-Policies 是一个响应头，用于指定跨域策略文件是否可用于该站点。跨域策略文件（cross-domain policy file）是一个 XML 文件，用于指定允许跨域请求的源和方法。通过使用 X-Permitted-Cross-Domain-Policies 响应头，网站可以控制是否允许其他站点使用跨域策略文件访问它们的资源。

X-Permitted-Cross-Domain-Policies 响应头有以下几个可选值：

- none：禁止使用跨域策略文件。这是默认值。
- master-only：只允许跨域策略文件从主域名（例如 example.com）加载，不允许从子域名（例如 [www.example.com）加载。
- by-content-type：允许在 MIME 类型为 text/x-cross-domain-policy 的响应中使用跨域策略文件。
- all：允许任何站点使用跨域策略文件访问该站点。

```nginx
add_header X-Permitted-Cross-Domain-Policies "all";
```

### 7. 目标X-Download-Options响应头缺失

X-Download-Options 是一个 HTTP 响应头，用于指示浏览器是否应该阻止文件的自动打开和执行。该头的值可以是以下之一：

- noopen：浏览器不应自动打开下载的文件，而应该提示用户将其保存或打开。
- nosniff：浏览器不应尝试执行下载的文件。这可以防止恶意文件被下载并在用户计算机上自动执行。

```nginx
add_header X-Download-Options noopen;
add_header X-Download-Options nosniff;
```

### 8. X-Frame-Options未配置

```nginx
add_header X-Frame-Options "SAMEORIGIN";
```

## 全部配置

```nginx
server { 
 
    location / {

    }

    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "no-referrer-when-downgrade" always;
    add_header Content-Security-Policy "default-src 'self' https://apis.map.qq.com/  https://mapapi.qq.com/ data: blob: 'unsafe-eval' 'unsafe-inline'" always;
    add_header X-Permitted-Cross-Domain-Policies "all" always;
    add_header X-Download-Options "noopen" always;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
}
```

## 参考：

- https://blog.csdn.net/haoqi9999/article/details/123271036

