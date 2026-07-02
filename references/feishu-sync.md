# 飞书同步规则

## 同步时机

文档类产物在本地发生新增或更新后，**同一轮工作内**必须同步到飞书：
- 学习材料、题库、答案、面试材料、面试实录
- 聚合数据、上传清单、冲刺页面数据
- 每次批量备课、冲刺材料更新后也必须同步

## 同步流程

1. 上传前检查 markdown 格式：代码块闭合、无孤立缩进、内联代码成对
2. 用 lark-drive skill 导入为 docx
3. 目标路径：`Daily-Learn/YYYY-MM-DD-<slug>/`
4. 全部 6 个文件：lesson / exam / exam-answers / interview / interview-answers / interview-transcript
5. 记录飞书文件夹链接到 `state.json`

## 失败处理

- 上传失败必须明确告知原因
- 保留本地待同步清单，不能假装已同步

## 范围

- 代码、脚本等非文档文件不强制上传
- 承载学习入口或题库数据的文件按文档类产物处理
