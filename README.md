# 压覆分析 Vue 3 Demo

## 使用方式

1. 安装 Node.js 18 或更高版本。
2. 在本目录执行：

   ```bash
   npm install
   npm run dev
   ```

3. 打开 Vite 输出的地址，通常是 `http://localhost:5173`。
4. 默认使用 Vite 代理：浏览器访问 `5173`，Vite 转发 `/kczyclYfgl` 到 `http://localhost:8080`。如果后端不是这个地址，同时修改 `vite.config.js` 中的 `proxy.target`。
5. 页面首次打开时输入 token。程序会自动把请求头构造成 `Authorization: Bearer <token>`；如果输入内容已经带 `Bearer`，不会重复添加。Token 只保存到当前浏览器会话，不调用登录接口。

## 对接接口

- `POST /kczyclYfgl/analysisNew?xmbh=...`
- `GET /kczyclYfgl/analysis/progress?xmbh=...`
- `GET /kczyclYfgl/analysis/progress/stream?xmbh=...`
- `POST /kczyclYfgl/analysis/cancel?xmbh=...`
- `GET /kczyclYfgl/analysis/results?xmbh=...`

示例不会调用登录接口；可以在页面中点击“更换 Token”重新输入。

如果直接把 `API_BASE_URL` 改成远程后端地址，浏览器会因为 `Authorization` 请求头先发送 CORS `OPTIONS` 预检。生产环境需要由网关或后端放行 `OPTIONS`，并返回允许的 Origin、Headers 和 Methods；本地开发建议保留 Vite 代理。

当前 Demo 的启动分析接口使用 `/kczyclYfgl/analysisNew`；进度、SSE、取消和结果接口仍使用原来的 `/analysis/progress`、`/analysis/progress/stream`、`/analysis/cancel` 和 `/analysis/results` 路径。
