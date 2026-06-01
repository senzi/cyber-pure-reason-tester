# 逻辑抗性评测

一个纯前端、local-first 的中文逻辑评测小应用。题目使用高密度中文语义包装基础命题逻辑，用户完成测试后会得到正确率、等效 B 数和本地历史记录。

## 功能特性

- 随机抽题：默认 10 题，最少 10 题，最多 172 题。
- 按需加载题目：首屏只加载题目清单，开始测试后只请求本轮抽中的题目文件。
- 前端密钥判题：题库不包含明文正确答案，答案由 `secret_h` / `secret_l` 通过 XOR 规则实时计算。
- 单页答题体验：进度、题干、选项和下一题按钮尽量保持在同一屏。
- 结果清算：展示正确率、等效 B 数、能力画像、耗时和正确题数。
- 错题审视：结果页可查看本轮答错的题目、所选答案和正确答案。
- 历史账本：完成测试后写入浏览器 `localStorage`，可在页面中查看历史记录，不提供清空按钮。
- 移动端适配：支持窄屏下的题数选择、答题和结果查看。

## B 数计算

当前等效 B 数公式：

```js
function calculateEquivalentB(scoreRate) {
  return 22 * Math.pow(scoreRate, 2) + 5 * scoreRate
}
```

其中 `scoreRate` 为 `0` 到 `1` 之间的正确率。页面展示时最多保留 8 位小数，并自动隐藏末尾补零。

## 题库结构

运行时题库位于：

```text
public/questions/manifest.json
public/questions/q-001.json
public/questions/q-002.json
...
```

`manifest.json` 只保存题目 id 和文件名。开始测试时，应用先从 manifest 中随机抽取指定数量的题目，再按需加载对应的单题 JSON。

## 本地历史记录

历史记录保存在浏览器：

```js
localStorage.getItem('cyber_reason_history_records')
```

每条记录包含开始时间、结束时间、耗时、题目数、正确数、正确率和等效 B 数。

## 开发运行

安装依赖：

```bash
npm install
```

启动开发服务器：

```bash
npm run dev
```

构建生产版本：

```bash
npm run build
```

预览生产版本：

```bash
npm run preview
```

## 技术栈

- Vite
- Vue 3 SFC
- 原生 CSS
- 浏览器 `localStorage`

## 设计原则

本项目不依赖后端服务。所有抽题、判题、计分、错题审视和历史记录都在浏览器本地完成。
