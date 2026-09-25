# 核心中文化发布流程修复

此文件包用于修改 fvtt-cn/foundry_chn 源码仓库，不是安装进 FVTT 的模组。
基于本次核对的现有发布工作流；不自动向 GitHub 推送或发布。

## 接入

1. 将本包的 `.github/workflows/` 和 `scripts/` 按同名目录复制到核心中文化仓库根目录。文件管理器可能隐藏 `.github` 文件夹，请一并复制。
2. 在仓库的源码 `cn.json` 中删除旧的 `"EDITOR.Font": "字体",` 那一行，保留新版 `EDITOR.Font.Title` 及其 Color、Size 子项。也可以在仓库根目录执行：

   ```sh
   git apply --check cn-source-cleanup.patch
   git apply cn-source-cleanup.patch
   ```

   附带的小补丁针对本次检查的 14.365 源码；如果源码已经改动，手动删除该旧键即可。不要对已经展开的安装版 cn.json 套用源码补丁。

3. 在有 Python 3.12 的环境中，从仓库根目录运行：

   ```sh
   python -m unittest discover -s scripts -p "test_*.py" -v
   python scripts/build_language.py cn.json
   ```

   第二条仅校验，不写文件。可选的本地打包前转换命令为：

   ```sh
   python scripts/build_language.py cn.json cn.json
   ```

4. 提交修复后，为包含这些修改的提交创建新版本标签和 Release。原来的版本命名规则仍适用。发布工作流会自动设置 Python、执行测试、构建并校验 cn.json，然后继续原来的 ZIP 打包、Release 附件上传和 Foundry 发布步骤。

旧标签通常指向旧代码；只对旧标签点击 Re-run jobs 不能视为使用了修复后的代码。建议发布一个新修订版，而不是继续分发原 14.365 损坏附件。

## 原理

- 使用 Python 标准库的区分大小写字典，保留 `Token` 和 `TOKEN` 等独立键。
- 递归展开点分隔键与嵌套对象，汇总完整翻译路径后统一重建。
- 遇到重复 JSON 键、重复翻译路径或字符串/对象路径冲突立即退出，打印具体键名。
- 保留数组、占位符和 HTML 文本，不对译文内容进行替换。
- 写入前后展平对比全部路径和值，保证构建过程不丢译文。
- 验证成功后才原子替换目标文件；失败保留已有文件。
- 验证失败时 GitHub Actions 后续打包与上传步骤不会执行。Release 创建事件本身已经发生，工作流不会撤销或删除那个 Release。
- 另附 validate_localization.yml，在相关 push、PR 或手动触发时提前运行同样的测试与源码检查；该工作流只读，不发布。

构建脚本只负责正确转换，不自动判定某条译文是否过时，也不会从网络下载译文。如果已经损坏的安装版 JSON 被当作源码传入，仅凭剩余内容无法恢复丢失的条目，应始终从源码仓库构建。

## 验证结果

11 项测试通过。未经清理的 14.365 源码被明确拒绝，错误指出 EDITOR.Font 冲突，原文件未改变。
清理旧键后，完整源码构建得到 3916 个条目，解析后的结果与你已经验证正常的修复版 cn.json 完全一致。
本地验证了源码清理补丁能够通过 git apply --check 并成功应用；未实际运行远程 GitHub Actions 或上传 Release。

## 参考

- 原发布流程：https://github.com/fvtt-cn/foundry_chn/blob/14.365/.github/workflows/create_release.yml
- Python Action 用法：https://github.com/actions/setup-python
