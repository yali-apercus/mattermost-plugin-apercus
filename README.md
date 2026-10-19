# mattermost-plugin-apercus

Aperçu 工作区的 Mattermost 定制插件 —— 字体排版、主题、贴纸、大表情与壳层 UI,以正式插件形式发布。

English: workspace customizations (WenKai typography, Catppuccin theme, stickers, jumbo emoji and shell UI) for our self-hosted Mattermost, packaged as a first-class plugin.

## 为什么是插件 / Why a plugin

Mattermost 没有官方 Custom CSS。我们试过反代层 CSS 注入(Caddy + replace-response),观感层完全可行,但一旦涉及"行为与结构"就会撞墙——典型案例如"通话面板停靠左下角":`.calls-notification-bar` 挂载在带 transform/contain 的虚拟滚动容器里,`position:fixed` 被劫持为容器内定位,纯 CSS 三连败。**插件是官方许可的正路**。完整失败笔记见文末。

## 功能 / Features

- [x] 样式注入:霞鹜文楷(LXGW WenKai)+ Catppuccin Mocha 细节(滚动条/选中色/聚焦环)
- [ ] `/sticker` 贴纸命令 + 大图渲染
- [ ] Jumbo emoji:纯表情消息自动放大
- [ ] 隐藏 "TEAM EDITION" 角标
- [ ] 通话面板停靠(需重排 Calls 插件组件,见笔记)
- [ ] Kaneo / Outline 的 RHS 面板(插件壳主菜)

## 构建 / Build

依赖:Go 1.24+(工具链自动切 1.26)、Node 24。

Linux/macOS:

```bash
make dist
```

Windows(无 make 时的手工等价流程,已在实践验证):

```bash
# 1. 生成 manifest 胶水(plugin.json → server/manifest.go + webapp/src/manifest.ts)
go run ./build/manifest apply
# 2. Go 服务端
cd server && go build ./...
GOOS=linux GOARCH=amd64 go build -o dist/plugin-linux-amd64 .
# 3. Webapp
cd ../webapp && npm ci && node node_modules/webpack/bin/webpack.js --mode production
# 4. 打包(tar 根必须含 plugin.json)
cd .. && tar -czf dist/plugin-apercus.tar.gz plugin.json assets public webapp/dist/main.js server/dist/plugin-linux-amd64
```

## 安装 / Install

System Console → Plugin Management → 上传 `plugin-apercus.tar.gz` → Enable。

## 踩坑笔记 / Field notes(对社区稀缺,欢迎引用)

1. **反代层 CSS 注入的边界**:Caddy/nginx `sub_filter` 注入 `<link>` 对"观感"完全够用(theme.park 同源思路),对"行为与结构"必败——DOM 归 React 管。
2. **containing-block 陷阱**:`position:fixed` 会被祖先的 `transform/contain/will-change` 劫持为容器内定位。Mattermost 消息列表正是这种容器。CSS 三连败(overlay / 侧栏留位 / JS 解除祖先陷阱)后回滚,结论:动组件挂载点=改源码或写插件。
3. **自定义表情 512KB 上限是编译期常量**:`server/channels/app/emoji.go: MaxEmojiFileSize = 1 << 19`,配置不可改,压缩是唯一解。
4. **新版 starter-template 模板的 manifest 机制**:`webapp/src/manifest.ts` 与 `server/manifest.go` 由 `go run ./build/manifest apply` 生成,`import manifest from 'manifest'` 靠 `resolve.modules: ['src', ...]` 解析——跳过 make 手工构建时极易漏掉这一步。

## License

Apache-2.0
