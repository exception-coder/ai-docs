# 简历填写预览现代化编码摘要

## 1. 核心规则

本次只改前端 `resume` 模块的表现与交互引导。保持 `ResumeState`、`ResumeData`、模板枚举、AI 优化、导出和持久化协议不变。

## 2. 涉及文件清单

| 文件 | 职责 | 本次动作 |
|---|---|---|
| `frontend/src/features/resume/pages/ResumePage.tsx` | 简历工具页面壳 | 重构为顶部工作台、质量摘要、沉浸式双栏 |
| `frontend/src/features/resume/components/ResumeEditor.tsx` | 简历编辑分区 | 增强分区状态、字段布局与引导文案 |
| `frontend/src/features/resume/components/BasicsForm.tsx` | 基础信息表单 | 美化头像与核心字段填写体验 |
| `frontend/src/features/resume/components/ListEditor.tsx` | 列表条目编辑 | 美化折叠条、空态、新增入口和操作按钮 |
| `frontend/src/features/resume/components/TemplateSelector.tsx` | 模板与主色选择 | 调整为更紧凑的选择控件 |
| `frontend/src/features/resume/styles/resume-templates.css` | 简历与页面样式 | 追加页面工作台辅助样式，不破坏模板隔离 |

## 3. 关键代码路径

| 文件路径 | 位置 | 说明 |
|---|---:|---|
| `frontend/src/features/resume/pages/ResumePage.tsx` | 1 | 页面主入口，状态、导出、工具栏与布局都在这里 |
| `frontend/src/features/resume/components/ResumeEditor.tsx` | 1 | 所有简历 section 的渲染入口 |
| `frontend/src/features/resume/components/BasicsForm.tsx` | 1 | 基础信息字段编辑入口 |
| `frontend/src/features/resume/components/ListEditor.tsx` | 1 | 工作、项目、教育复用的列表编辑器 |
| `frontend/src/features/resume/components/TemplateSelector.tsx` | 1 | 模板和主色选择器 |
| `frontend/src/features/resume/styles/resume-templates.css` | 1 | 模板样式及本次页面辅助样式 |

## 4. 验证

- 在 `frontend/` 运行 `npm run build`。
- 启动 Vite 后打开简历路由，检查桌面与移动视口下编辑、预览、导出按钮和隐私模糊状态。
