- # logseq
  由于个人掌握知识繁杂，故采用 [logseq](https://logseq.com/) 和 [Obsidian](https://obsidian.md/) 以记录，旨在让知识清晰且有条理。
- **访问：[acdiost.github.io](https://acdiost.github.io)**
- 没有目录概念，层级即目录
- 内容可能有纰漏，欢迎 PR 修改和补充。
- ## 初始化
	- 修改日期格式为 `yyyy-MM-dd`
	- 关闭 显示括号
	- 若要发布到 [GitHub Pages](https://pages.github.com/) 则在 `logseq/config.edn` 文件中取消注释 `:publishing/all-pages-public? true`
- ---
- ## 发布到 Github Pages
- ### 目录初始化
	- 在 logseq 库中新建目录 docs
	- 选择 导出图谱 -> Export public pages -> 选择 docs 文件夹
	- 打开 docs 文件夹中的 index.html 以预览
- ### 上传至 GitHub
	- 在 GitHub 中新建 repository，仓库名称须为：username.github.io，点击创建
	- ```bash
	  git init
	  git add README.md
	  git commit -m "first commit"
	  git branch -M main
	  # change username
	  git remote add origin https://github.com/acdiost/acdiost.github.io.git
	  git push -u origin main
	  ```
	- 在 GitHub 仓库的设置中选择 pages，修改 Source 目录为 `/docs`
	- 访问 [acdiost.github.io](https://acdiost.github.io) 测试
- ---
- ## 与 Obsidian 协作
- 在 obsidian 中打开 logseq 库
- 初始化设置
	- 显示行号
	- 关闭制表符
	- 在核心插件中启用日记、大纲、字数统计、幻灯片等功能
	- 在插件选项中进行 日记 配置
		- 日期格式：`YYYY_MM_DD`
		- 新建日记存放位置：`journals`