# 多个 TXT 在线订阅合并为 M3U

GitHub Actions 读取仓库根目录的 `url.txt`，下载其中的 TXT 频道列表，转换、合并、去重后更新 `live.m3u`。播放器订阅固定 Raw 地址即可。

本版不再读取 `TXT_URL` 环境变量或 Actions Secret。已有的同名 Secret 可以删除，保留也不会影响运行。

## 配置来源

直接编辑仓库根目录的 `url.txt`，每行填写一个完整的 HTTP/HTTPS TXT 订阅网址：

```text
# 可以用整行备注说明来源
https://example.com/source-a.txt
https://example.org/source-b.txt
```

以上 example 网址仅演示格式，请替换为实际来源。随包 `url.txt` 沿用了原说明中的 `https://jk.html-5.me/i/03537.txt`；它不是从旧 Secret 导出的完整来源清单，请按需替换或补充。本次修改没有在线验证该来源。

- 文件必须命名为小写 `url.txt`，与 `convert.py` 位于同一目录。
- 使用 UTF-8 保存，支持 UTF-8 BOM、Windows 换行、空行和以 `#` 开头的整行备注。
- 一个网址一行，不要用逗号连接多个网址，也不要添加行尾备注。
- 相同网址只下载一次；修改 `url.txt` 并提交到 `main` 后会触发更新。
- 文件缺失、没有有效网址或包含格式错误的网址时，任务失败并保留旧的 `live.m3u`。
- 网址必须指向频道文本；脚本不会从任意网页中寻找订阅链接。

## 从 Secret 版本迁移

1. 替换仓库根目录的 `convert.py`。
2. 替换 `.github/workflows/update.yml`，注意保留完整目录路径。
3. 在根目录新增 `url.txt`，把原先配置在 `TXT_URL` Secret 中的全部来源按行复制进去。
4. 提交上述三个文件后检查 Actions；也可手动运行 `Update M3U`。

播放器原有的 `live.m3u` 订阅地址不变。压缩包保留了原播放列表，只有运行成功后才会更新；旧列表不代表当前 `url.txt` 配置的生成结果。

## 新仓库部署

1. 新建公开 GitHub 仓库，默认分支使用 `main`。
2. 把 `convert.py`、`url.txt` 和 README 放在仓库根目录，把工作流放在 `.github/workflows/update.yml`。
3. 修改 `url.txt` 为实际 TXT 订阅网址，无须创建 Secret。
4. 打开 Actions → Update M3U → Run workflow，等待成功。
5. 在播放器添加以下地址，替换用户名和仓库名：

```text
https://raw.githubusercontent.com/你的GitHub用户名/你的仓库名/main/live.m3u
```

不要使用 `github.com/.../blob/...` 的文件预览地址。播放器所在网络必须能够访问 Raw 域名。

## 输入格式与合并规则

TXT 来源内容示例：

```text
央视频道,#genre#
CCTV-1 综合,https://example.com/cctv1.m3u8
CCTV-2 财经,https://example.com/cctv2.m3u8
卫视频道,#genre#
湖南卫视,https://example.com/hunan.m3u8
```

- 频道行以第一个英文逗号分隔名称与地址，地址参数里的后续逗号会保留。
- 按 `url.txt` 中的来源顺序合并，各源内部顺序保留；每个来源独立解析分组。
- 分组、名称和地址相同的条目去重；同名不同地址或不同分组的条目保留。
- 支持 UTF-8、GB18030 和有 BOM 的 UTF-16 频道文本。
- 允许 HTTP、HTTPS、RTSP、RTMP、RTMPS、UDP、RTP 播放地址。
- 输入不支持 M3U，也不合并 EPG；不会检测播放、解析网页播放接口、转码或处理特殊鉴权头。

## 下载、更新与失败处理

先使用普通 HTTP 请求，每个来源最多尝试三次，每次读取上限 10 MiB。HTTP 下载失败或返回 HTML 时，尝试用隔离 Chromium 会话读取页面中唯一的 `<pre>` 文本。此方式不保证能通过源站验证；需要登录、验证码或页面未展示频道文本时可能失败。

任何来源下载失败、浏览器读取失败或没有有效频道，整次更新停止，旧列表保留。若某来源仍有有效频道，其他无法识别的行会被跳过；当前没有频道数量下降保护，因此成功不代表来源完整，也不代表能够播放。

脚本先写临时文件，再替换同目录的 `live.m3u`。工作流仅在列表内容改变时创建提交。

工作流触发方式：

- 手动运行 `Update M3U`。
- 提交到 `main` 且修改 `convert.py`、`url.txt` 或工作流文件。
- 定时：`17 */6 * * *`，按 UTC 每六小时的第 17 分钟运行，可能排队延迟。

工作流总超时为 30 分钟，每次安装 Playwright 和 Chromium。来源较多或较慢时可能超时。公开仓库长时间无活动可能被停用定时工作流，需要检查 Actions 状态。

若推送失败，检查工作流的仓库写入权限、分支保护和组织限制。运行期间分支出现新提交，也可能导致推送冲突。播放器缓存旧列表时需要刷新订阅。

## 本地运行

在 Python 环境中安装浏览器支持：

```bash
python -m pip install playwright
python -m playwright install --with-deps chromium
python convert.py
```

如果所有来源均可通过普通 HTTP 下载，脚本本身只需要 Python 标准库；只有进入浏览器分支时才导入 Playwright。脚本始终读取自身目录下的 `url.txt`，并把 `live.m3u` 写回同一目录。

## 公开范围

公开仓库中的 `url.txt` 及其 Git 历史会公开全部来源网址；`live.m3u` 及其历史会公开播放地址，包括网址中的令牌或订阅参数。请只填写允许公开的来源。
