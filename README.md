# MerkleHash

文档的 Merkle Tree 根签名工具（单文件 Swift 实现）。

为一篇文档计算一个唯一的 **Merkle Root** 签名：只要任意一分片内容不同（或分片长度不同），根签名就会改变；反之根相同 ⟺ 逐片内容完全一致。可用于大批量文档的内容去重、完整性校验与版本比对。

## 算法

1. 按 **Unicode 字符**分片（`--chunk-size`，默认 2000 字；汉字、字母均计 1 字），每片取其 UTF-8 字节 → `BLAKE2b` 哈希 = 叶子节点。
2. 相邻两片哈希拼接（`left || right`）→ 再算一次 `BLAKE2b` = 中间节点。
3. 从底层逐层向上两两聚合；某层节点数为奇数时，最后一个节点**复制自身与自身配对**。
4. 最后一层仅剩两个中间节点时，拼接它们的哈希再算最后一次哈希 → **Merkle Root**（整本文档唯一根签名）。
   - 只有一片时：根 = 该片哈希。
   - 空文件时：根 = `BLAKE2b(空)`。

`BLAKE2b` 为纯 Swift 实现（RFC 7693，12 轮压缩，输出 1..64 字节，默认 32 字节 = BLAKE2b-256），无外部依赖，自检测试用 Python `hashlib.blake2b` 的权威向量交叉校验。

## 编译

单文件需 `--parse-as-library` 才能使用 `@main`：

```sh
swiftc -O -parse-as-library -o /tmp/bookmerkle MerkleHash.swift
```

## 用法

```sh
bookmerkle <文件或目录> [更多路径...] [选项]
```

支持同时传入多个文件或目录（目录递归扫描，跳过以 `.` 开头的隐藏路径，仅取指定扩展名）。

### 示例

```sh
# 对单个文件计算根签名
bookmerkle book.md

# 递归扫描目录，写 JSON 报告
bookmerkle ./docs --ext md,txt --report report.json

# 报告中附带全部叶子哈希（可定位内容差异出现在哪一片）
bookmerkle ./docs --report report.json --leaves

# 自定义分片长度与哈希字节数（BLAKE2b-512）
bookmerkle book.md --chunk-size 1000 --digest-bytes 64

# 运行自检
bookmerkle --self-test
```

### 选项

| 选项 | 说明 |
| --- | --- |
| `--chunk-size N` | 分片长度（Unicode 字符数，默认 2000） |
| `--digest-bytes N` | `BLAKE2b` 输出字节数 1..64（默认 32，即 BLAKE2b-256） |
| `--report PATH` | 写 JSON 报告（路径/大小/字符数/叶子数/层数/根） |
| `--leaves` | 报告中附带全部叶子哈希（可定位内容差异在哪片） |
| `--ext LIST` | 目录扫描的扩展名，逗号分隔（默认 `md,txt`） |
| `--limit N` | 最多处理 N 个文件 |
| `--self-test` | 运行自检（含 BLAKE2b 权威向量与多组 Merkle 交叉验证） |
| `--help` | 显示帮助 |

### 输出

- 标准输出：每文件一行 `<root>  <path>`（可管道 `sort` 做去重比对）。
- 标准错误：进度与统计日志（如唯一根签名数量）。
- `--report` 时额外写出 JSON 报告，结构含 `chunkSize`、`digestBytes`、`fileCount`、`uniqueRoots` 及每个文件的 `path/bytes/chars/leafCount/levelCount/root/leaves`。

### 退出码

- `0`：成功完成。
- `1`：没有可处理的文件。
- `2`：参数错误（未知选项、缺少路径等）。

## 自检

`--self-test` 会校验：

- BLAKE2b 权威向量（空串、短串、跨 128 字节块、整块边界等，覆盖 256/512 两种输出长度）；
- Merkle 树的多种情形（单分片、空文件、两/三分片、中文 UTF-8 分片、多层递归）；
- 树结构与确定性（层级收敛、奇数节点自配对、同输入同根、任一分片改动即改变根、分片长度影响根）；
- 端到端真实文件读取，结果须与硬编码预期值一致。

全部通过则打印 `MerkleHash self-test: all checks passed` 并返回 `0`，否则列出失败项并返回 `1`。

## 备注

- 分片扫描边走边累积 UTF-8 字节，内存只占用一个分片，无需把整本大书转为字符数组，适合超大文档。
- 根签名取决于 `(内容, --chunk-size, --digest-bytes)` 三元组；比较两份文档是否一致时，这些参数必须保持一致。
