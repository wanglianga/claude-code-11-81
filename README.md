# 地下管网血缘变更工作台（离线）

面向某市地下管网中心月度叠加图层场景的**纯前端离线工作台**：把测绘、阀门台账、管段拓扑、风险模型与施工许可的血缘变更在浏览器本地完成核验、求值、重算与发布审阅。技术栈为 **React 18 + TypeScript + Vite + IndexedDB + WebWorker**，无后端、无登录、断网可用。

## 1. 能力与需求对应

| 需求 | 实现位置 |
| --- | --- |
| 导入器记录包哈希、来源、坐标系、测量历元、单位 | `src/domain/importer.ts`、页面① |
| 语义核验：历元缺失 / 单位误标 / 无法唯一续接，哈希正确不替代语义核验 | `importer.ts`（EPOCH_MISSING、UNIT_MISLABELED、UNRESOLVED_LINEAGE 均为 block） |
| 控制点原始/校正坐标、残差、分区 RMS 与 95% 误差带 | `src/domain/correction.ts`、页面② |
| 实体谱系：一对一 / 拆分 / 合并 / 未能续接 + 测量证据 + 置信度 | `src/domain/lineage.ts`、页面③ |
| 按阀门连接与端点容差生成版本化网络、跨瓦片依赖 | `src/domain/topology.ts`、页面④ |
| 许可账：编号、获批边界、3 米缓冲、工程期、依据管段版本，边界固化不漂移 | `src/domain/permit.ts`、页面⑤ |
| 模型运行账：固定连通特征指纹、拓扑、阈值、输入水位 | `src/domain/model.ts`、页面⑥ |
| 影响求值四类：几何缓冲穿越、拆并改变模型输入、许可覆盖、图层重建 | `src/domain/impact.ts`、页面⑦ |
| 切片闭包重算（含跨瓦片依赖、边界证明、保留未变区域引用）与全链重建对照 | `src/domain/recompute.ts`、页面⑧ |
| 发布快照回放（输入包/转换/谱系/许可/模型版本）与阻断原因 | `src/domain/snapshot.ts`、页面⑨ |
| 离线规则更新使旧影响失效；同步只合并未冲突草案、不覆盖服务器较新发布 | `src/domain/sync.ts`、页面⑩ |
| 重计算在 WebWorker 执行，全部台账在 IndexedDB 持久化 | `src/workers/`、`src/db/indexdb.ts` |

## 2. 虚构样例数据（可重复运行，无随机数）

定义于 `src/domain/sampleData.ts`，两个相邻瓦片 `TILE_A`/`TILE_B`（分界 x=100）：

- **基准月包 `PKG-2026-01`**：6 个管段（S1–S6，S4 跨瓦片）、2 个阀门（V1 开、V2 关）、2 个控制点（CP1/CP2）、许可 `PRM-2026-001`（3 米施工缓冲，工程期 2026-03-01 至 2026-09-30，获批对象 S3）、风险模型 `RM-CONN`（连通特征，阈值 0.7，水位 2.5 米）。
- **控制点改正包 `PKG-2026-02`**：Z-A 分区整体平移约 **0.81 米**，S3 向许可缓冲外沿移动；95% 误差带半宽约 1.55 米，S3 缓冲外沿剩余净距约 0.5 米（未穿越但误差带触及）→ **许可进入复核并阻断该局部发布**。
- **拆并阀门包 `PKG-2026-03`**：跨瓦片管段 S4 拆为 `S4A`/`S4B`，新阀门 `V3` 在瓦片分界接入邻瓦片；切片闭包经 V3 扩展到两瓦片，连通特征指纹改变触发模型重算；组合许可种子后闭包覆盖整包，切片按规则扩大至整包。全链路线给出相同新拓扑与许可判定，仅重建范围不同。
- **异常补测包 `PKG-BAD-09`**：测量历元缺失、单位误标（坐标为米量级却声明厘米）、历史管段 S77 两个等距候选无法唯一续接；即使 SHA-256 正确仍被语义核验阻断，不产生校正/拓扑/快照。

## 3. 本地启动

### 3.1 开发模式（Node 22，已在当前环境验证）

```bash
npm install          # 如全局 npm 缓存无权限，可加 --cache ./.npm-cache
npm run dev          # http://localhost:5173
npm run build        # 类型检查 + 生产构建到 dist/
npm run preview      # 本地预览生产构建 http://localhost:4173
npm test             # vitest 单测（18 个用例）
```

### 3.2 Docker / Compose（一键启动，nginx 托管静态资源）

```bash
docker compose up --build
# 打开 http://localhost:8080
```

Dockerfile 为多阶段构建（node:22-alpine 构建 → nginx:1.27-alpine 托管），含 SPA 回退与 gzip 配置。

### 3.3 账号说明

本应用为**纯离线单用户工具，没有登录体系和服务端账号**。打开即用；发布审阅页默认审阅人身份为「审阅人-张工」，可在界面中修改，审阅草案连同身份保存在本地 IndexedDB。第⑩页的“服务器发布版本”为内置的模拟同步端点，用于演示冲突合并规则。

## 4. 推荐验证路径（与题目叙事一致）

1. 页面①选择「切片重算」，依次导入四个来源包；观察异常包出现三条 block 语义问题且无后续产物。
2. 页面②查看 CP1/CP2 残差、Z-A 的 95% 误差带与 Z-B 的“继承全局”标注，地图上灰色虚线为上一版本几何。
3. 页面③确认 `PKG-2026-03` 中 S4 → S4A+S4B 的拆分关系及共点测量证据。
4. 页面④确认 V3 阀门节点属于 TILE_B、跨瓦片依赖 `TILE_A → TILE_B via S4A/V3`。
5. 页面⑤/⑦确认 `PKG-2026-02` 许可判定为 **review（复核）**、阻断原因指向 PRM-2026-001；`PKG-2026-03` 闭包同时包含两瓦片管段。
6. 页面⑧点击“运行切片 vs 全链对照”，两路线拓扑签名与许可判定一致；窄种子（如仅 S3）时切片只重算 1 条管段并保留 6 条原发布引用。
7. 页面⑨对快照执行“回放快照”，回放指纹一致；阻断快照保留全部阻断原因，禁止发布。
8. 页面⑩先“模拟离线规则更新”，回到⑦看到历史报告标记失效；再“模拟服务器发布版本前进”，保存一条指向服务器终态发布的草案后执行同步，观察其被列为冲突且不覆盖服务器版本。
9. 断网后刷新页面，IndexedDB 中台账、快照、草案仍然可读。

## 5. 已执行的验证（仅记录实际跑过的内容）

- `npx tsc --noEmit`：严格模式（含 noUnusedLocals/noUnusedParameters）通过，无类型错误。
- `npx vitest run`：**2 个测试文件、18 个用例全部通过**，覆盖导入语义拦截、校正残差/误差带、拓扑与跨瓦片依赖、许可 review/blocked 判定、闭包扩展、切片/全链一致性、快照回放、离线失效、同步冲突合并及端到端流水线。
- `npx vite build`：构建成功，主线程包与 `pipeline.worker-*.js` 独立产出。
- `vite preview` 冒烟：`/` 返回 200，Worker 资源返回 200。
- 未执行项：当前容器内没有 Docker 守护进程，`docker compose build` 未实际运行；Dockerfile/compose 配置已按标准多阶段静态站点方式提供，需在具备 Docker 的环境执行。

## 6. 目录结构

```
src/
├─ domain/        # 纯领域逻辑（无 DOM 依赖，可在 Worker 与测试中复用）
│  ├─ types.ts        领域类型
│  ├─ geometry.ts     距离/缓冲/多边形/拆分等几何工具
│  ├─ hash.ts         SHA-256、稳定序列化、FNV 指纹
│  ├─ sampleData.ts   四个可重复虚构来源包
│  ├─ importer.ts     导入与语义核验
│  ├─ correction.ts   控制点校正与分区误差
│  ├─ lineage.ts      实体谱系
│  ├─ topology.ts     版本化网络与下游闭包
│  ├─ permit.ts       许可账与缓冲判定
│  ├─ model.ts        连通特征与风险模型运行账
│  ├─ impact.ts       四维影响求值
│  ├─ recompute.ts    切片/全链重算与对照
│  ├─ snapshot.ts     发布快照与回放
│  ├─ sync.ts         离线失效与同步合并
│  └─ pipeline.ts     端到端编排
├─ workers/       # WebWorker（pipeline.worker.ts）、通信协议与客户端
├─ db/            # IndexedDB 封装
├─ state/         # React Context 全局状态
├─ components/    # SVG 地图与通用 UI
└─ pages/         # 十个工作台界面
```
