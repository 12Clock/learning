# AGENTS.md

## 仓库用途

本仓库用于**收集学习知识**。所有沉淀下来的知识以纯 Markdown 形式保存在 `knowledge/` 目录中，
按主题分类，并随着内容增多渐进式地细分。

## 知识收集 Skill

仓库提供 `knowledge-collection` skill（见 [`skills/knowledge-collection/SKILL.md`](./skills/knowledge-collection/SKILL.md)），
负责一切知识收集相关操作。其能力：

1. **问答归类**：对用户提出的问答（Q&A）进行归类；当某一分类下问答增多时，支持进一步细分类。
2. **纯 Markdown + 渐进式加载**：知识库全部为 `.md` 文件。SKILL.md 作为精简入口，详细规则
   （分类、格式）作为引用文件按需加载；当某分类问答过多时，在该分类下额外拆分出独立 md 文件，
   并在索引中登记引用，避免一次性加载全部内容。
3. **绑定本仓库**：仅在本仓库的 `knowledge/` 下进行收集与组织操作。

## 何时触发

当用户出现以下意图时，使用 `knowledge-collection` skill：

- 提出问答并希望记录、收藏到知识库
- 分享一个知识点 / 结论 / 经验需要归档
- 要求整理、归类、检索已有知识
- 某分类条目过多需要细分

## 目录结构

```
AGENTS.md                              # 本文件
skills/
  README.md
  knowledge-collection/
    SKILL.md                           # skill 入口（精简）
    references/
      categorization.md                # 分类与细分规则（按需加载）
      format.md                        # Q&A 文件格式规范（按需加载）
knowledge/
  INDEX.md                             # 分类总索引（路由表）
  <category>.md                        # 各分类的 Q&A 文件
  <category>/                          # 细分后升级为目录
    INDEX.md
    <subtopic>.md
```

## 工作约定

- 收集知识前先读 `knowledge/INDEX.md` 定位分类。
- 新增 / 细分分类后，同步更新 `knowledge/INDEX.md`，保持索引与实际文件一致。
- 全程使用纯 Markdown，不引入其他存储格式。
