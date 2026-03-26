# CLAUDE.md — 项目约束与规范

## Git 提交规范

1. **每次修改必须以当日日期作为 tag 提交**，格式：`v{YYYY-MM-DD}`（如 `v2026-03-26`）
   - 同一天多次提交时追加序号：`v2026-03-26.1`、`v2026-03-26.2`
2. 提交后合并到 `main` 分支

### 提交流程

```bash
# 1. 在功能分支上开发（分支名建议：feature/xxx 或 日期分支）
# 2. 提交变更
git add <files>
git commit -m "描述信息"

# 3. 打 tag（当日日期）
git tag v$(date +%Y-%m-%d)

# 4. 合并到 main
git checkout main
git merge <branch>
git push origin main --tags
```

## 项目结构

```
Programming-Learning/
├── CLAUDE.md              # 项目约束（本文件）
├── shell-guide/           # Shell 从零到一学习路线
├── python-sre-guide/      # Python SRE 运维实战指南
└── golang-guide/          # Go 语言学习指南
```

## 文档编写规范

- 每篇文档遵循：标题 → 一句话说明 → 概念介绍 → 代码示例 → 实战练习 → 小结
- 目录使用带编号前缀的中文名：`01-章节名/01-文档名.md`
- 代码示例必须可直接运行
- 每个子项目必须包含 `README.md` 作为总索引
