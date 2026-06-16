# Token效率 — 仓仓项目教训 (2026-06-08)

## 浪费模式 (避免重复)

### 1. 修复损坏的整合包 > 3轮
- **信号**: 二进制包缺关键文件 (python.exe / DLL)。修复一轮后仍无法import。
- **纠正**: 2次修复失败后立即放弃。提取数据(预训练模型/配置文件)，用系统Python+pip。
- **本次代价**: ~8轮对话修embedded Python runtime。最终方案=系统Python+opencc wheel 1行解决。
- **铁律**: 修复环境不超过2轮。第3轮=放弃。

### 2. GPU/硬件不兼容多轮盲试
- **信号**: RTX 5060 + PyTorch CUDA 11/12.4 = sm_120不兼容。Segfault或DLL load失败。
- **纠正**: 1次GPU失败后立即切CPU验证。硬件问题≠配置问题。不等用户说。
- **本次代价**: VoxCPM2 5+轮GPU/CPU segfault试错，最终放弃。GPT-SoVITS runtime torch CUDA11=sm_120不可用。

### 3. 压缩工具交互式提示卡死
- **信号**: Bandizip `bz x` 不加 `-y` 导致覆盖提示，后台卡死。
- **纠正**: 所有压缩命令加 `-y` 或 `-o` (overwrite)。优先用系统自带的 `unzip`。

### 4. 下载前不验证URL
- **信号**: curl GitHub raw URL 返回14字节 (HTML错误页)。
- **纠正**: 下载后先验证文件大小 > 1KB。

## 成功模式 (强化使用)

### 1. 系统Python + pre-built wheels
- opencc: `pip install opencc` 直接获取cp312 wheel。无需cmake编译。
- 依赖优先看PyPI wheel，pip install直接试，不行再找源码。

### 2. 浅克隆 (shallow clone)
- `git clone --depth 1` 解决GitHub RPC超时。对大仓库必用。

### 3. 数据提取策略
- 整合包损坏≠数据没用。只取pretrained_models，复制到工作目录，删除剩余。
- 本次从6.3GB整合包中只取3.4GB模型，节省12GB磁盘。

### 4. 先ls再判断
- 任何"文件存在/缺失"判断前先ls。已写入项目CLAUDE.md。

## 环境速查

| 环境 | Python | torch | CUDA | RTX 5060兼容 |
|------|--------|-------|------|-------------|
| System Python | 3.12.9 | 2.6.0+cu124 | 12.4 | ❌ sm_120需CUDA12.8+ |
| GPT-SoVITS runtime | 3.9.13(修复后) | CUDA11 | 11.0 | ❌ 完全不兼容 |
| 推荐 | 3.12 | 2.7+cu128(nightly) | 12.8+ | ✅ 唯一方案 |

## 决策树: 整合包启动失败

```
1. 运行启动脚本 → 失败
2. 检查报错: 缺文件? 版本不匹配? DLL load失败?
3. 如果是"缺python.exe/DLL":
   → 检查是否有系统Python可用
   → 是: pip install依赖 → 复制预训练模型 → 用系统Python跑
   → 否: 下载匹配版本的embeddable Python → 重试
   → 2次失败: 放弃整合包。从GitHub源码+系统Python重新搭建。
4. 如果是"CUDA/sm_120不兼容":
   → CPU回退测试: CUDA_VISIBLE_DEVICES=""
   → CPU通过 → 确认问题在GPU → 告知用户选项(升级torch/用CPU/等)
5. 耗时>10分钟: 必须向用户汇报并给选项。不默默死磕。
```

## Token消耗估算 (本次会话)

| 阶段 | 操作 | 估计token | 浪费原因 |
|------|------|-----------|----------|
| VoxCPM2尝试 | 5轮GPU/CPU segfault调试 | ~15K | 硬件不可用仍死磕 |
| 整合包解压 | Bandizip提取 | ~3K | 正常 |
| Embedded Python修复 | 下载/解压/配置/调试DLL | ~12K | 应2轮后放弃 |
| pip依赖安装 | 多轮尝试(opencc失败→wheel→成功) | ~8K | cmake失败后应直接找wheel |
| 端口冲突处理 | kill进程+重试 | ~2K | 正常 |
| WebUI启动成功 | 最终验证 | ~2K | 正常 |
| **合计** | | **~42K** | **~27K可避免(64%)** |

## 改进措施

1. **硬件兼容性矩阵前置检查**: 新工具安装前先验证 (GPU架构, CUDA版本, PyTorch版本) 三角匹配。
2. **2轮放弃原则**: 环境修复类操作最多2轮。第3轮前必须问用户。
3. **整合包=数据源**: 不信任整合包的runtime。始终优先系统Python。
4. **pip install直接试**: 不读requirements.txt逐行装。先pip install看wheel可用性。
