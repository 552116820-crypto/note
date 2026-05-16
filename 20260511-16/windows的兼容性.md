# Claude Code 对话记录

- **来源**: Claude Code
- **会话**: `b06a886f-0e2c-4618-8326-2e9e61e94358`
- **范围**: Turn 1
- **导出时间**: 2026-05-16 16:45:49

---

### Turn 1  *(2026-05-16 08:33)*

**User:**

Windows 干跑 Python pipeline 坑多，强建议全部走 WSL，强烈建议 Windows 同事都走 WSL（playwright/pypinyin 在 Windows 原生少数包会 build 挂），为什么这么说windows

**Assistant:**

主要是这几个具体坑，按本项目实际会踩到的频率排：

**1. C 扩展包编译链**
- `rapidfuzz`（C++ Levenshtein）、`pypinyin` 部分版本、`pillow` 老版本——Windows 原生 pip install 会走 source build，需要装 Visual Studio Build Tools（几个 G）。WSL 直接 manylinux wheel 落地。
- `playwright install chromium` 在 Windows 偶尔卡在防火墙 / SmartScreen / 沙箱权限，WSL 走 apt 依赖 + Chromium 一次到位。

**2. 文件编码 / 路径**
- 本项目目录全是中文（`positions/P03_三防漆/`），Windows console 默认 cp936（GBK），Python 读 `verified.jsonl` 不显式 `encoding='utf-8'` 就乱码 / UnicodeDecodeError。WSL 默认 UTF-8。
- `pathlib` 在 Windows 反斜杠 + 长路径（>260 字符）问题，evidence/ 目录下截图文件名一长就崩。

**3. 行结束符 + 脚本**
- `decisions/*.tsv`、`verified.jsonl` 在 Windows 编辑器一存就 CRLF，`merge_batches.py` 按 `\n` split 会带 `\r`，下游字符串比对失败（"Live\r" != "Live"）。
- `.claude/commands/` 里如果有 bash 段，Windows cmd / PowerShell 不认。

**4. playwright 行为差异**
- Windows 上 headless chromium 的字体回退跟 Linux 不一样，SPA 站点（NEPCON / HKPCA）渲染出来的 DOM 偶尔差几个元素，selector 配置就得双份。

**5. 文件锁**
- Windows 上 `result.xlsx` 被 Excel 打开时是独占锁，`build_xlsx.py` 直接写失败。WSL 在另一个 mount，绕过这层。

简言之：这个项目的所有 dev 工具链（rapidfuzz / playwright / 中文路径 / jsonl + utf-8 / bash agent 调用）都是 Linux 生态原生友好的，Windows 原生跑等于每个环节都要做一次兼容性 patch，得不偿失。

---
