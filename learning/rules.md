# 学习文档写作规范

## Markdown 写作规范

1. Markdown 写作需要符合 @learning/markdownlint/doc 定义的规则；
2. 应该使用 `markdownlint-cli2 <markdown-file-name>` 命令行工具验证 md 规范；
3. 标题之前不要用分隔线，例如

   ```markdown, 避免这个分隔线
   正文

   ---

   ## 标题
   ```

4. 忽略 MD013（已写入 @learning/.markdownlint.json）普通段落整段话禁止折行，写成一行，例如：

   ```markdown, 避免折行
   这是一个完整的段落，但是可能比较长。
   在标点之后折行。

   这是另一个段落。
   ```

   ```markdown, 写成一行
   这是一个完整的段落，即使比较长，也不要折行。

   这是另一个段落。
   ```

5. 非代码的代码块（例如字符画、输出示例等）标注 plaintext；
