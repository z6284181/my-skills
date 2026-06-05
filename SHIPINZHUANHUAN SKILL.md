---
name: hevc-quicklook-fix
description: Fix macOS Quick Look (spacebar preview) not working for HEVC MP4 videos. Use when user says videos can't be previewed with spacebar, Quick Look fails for MP4 files, or mentions hev1/hvc1 codec issues. Detects HEVC files with hev1 fourcc and losslessly remuxes them to hvc1 (no re-encoding, quality preserved).
---

# HEVC Quick Look Fix

macOS Quick Look 无法预览 `hev1` 编码的 HEVC 视频。无损重封装为 `hvc1` 即可修复，不改画质。

## 核心原则

**只转换 `hev1` 文件！** `avc1`（H.264）和 `hvc1`（HEVC 容器内参数集）已经能正常空格预览，绝对不要碰它们。

## Requirements

- `ffmpeg` 必须已安装。如果没有：`brew install ffmpeg`

## 工作流程

### 第一步：扫描检测（必须先做）

对目标目录扫描所有 MP4，输出三类结果：

```bash
for f in "<目标目录>"/*.mp4; do
  codec=$(python3 -c "
import struct
file = open('$f', 'rb')
file.seek(-26214400, 2)
data = file.read(26214400)
file.close()
idx = data.find(b'moov')
if idx >= 0:
    chunk = data[idx:idx+2048]
    for c in [b'avc1', b'hvc1', b'hev1']:
        if chunk.find(c) >= 0:
            print(c.decode())
            break
    else:
        print('unknown')
else:
    print('no_moov')
" 2>&1)
  echo "$codec $(basename "$f")"
done
```

### 第二步：分类处理

| 编码 | 能否空格预览 | 操作 |
|------|-------------|------|
| `avc1` | ✅ 能 | **跳过，不转换** |
| `hvc1` | ✅ 能 | **跳过，不转换** |
| `hev1` | ❌ 不能 | **转换** |

**重要**：扫描结果中标记为 `avc1` 和 `hvc1` 的文件一律跳过，只处理 `hev1`。

### 第三步：逐个转换 hev1 文件

```bash
ffmpeg -i "input.mp4" -c copy -tag:v hvc1 "input.tmp.mp4" -y && \
rm "input.mp4" && \
mv "input.tmp.mp4" "input.mp4"
```

无损重封装，不改任何画质，6GB 文件约 1-2 分钟。

### 第四步：输出汇总

转换结束后汇报：
- 总共扫描了多少文件
- 跳过了多少（avc1 + hvc1）
- 转换了多少（hev1 → hvc1）

## 注意事项

- **一次一个文件**，转完删原文件再转下一个，防止磁盘空间不足
- 转换前检查磁盘剩余空间是否大于最大文件 × 2
- 如果 ffmpeg 没装，先 `brew install ffmpeg` 再继续
