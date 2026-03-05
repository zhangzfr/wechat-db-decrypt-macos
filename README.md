# macOS arm64 微信 4.x 数据库解密

提取微信 (WeChat) 数据库密钥，解密 SQLCipher 加密的本地数据库，导出聊天记录。

## 须知

1. 目前只在微信 4.1.x (macOS arm64) 测试过，4.0 以下版本不适用
2. macOS 需要禁用 SIP (`csrutil disable`)

## 快速开始

### 1. 安装依赖

```bash
brew install llvm sqlcipher
```

### 2. 提取密钥

确保微信已登录并正在运行：

```bash
PYTHONPATH=$(lldb -P) python3 find_key_memscan.py
```

脚本会自动扫描微信进程内存，提取所有数据库的密钥，保存到 `wechat_keys.json`。

> **备选方案**: `find_key.py` 通过在 `setCipherKey` 函数上设置断点来捕获密钥，需要在微信中操作触发数据库访问才能逐个捕获。`find_key_memscan.py` 是一次性全量扫描，推荐使用。

### 3. 验证密钥（可选）

```bash
python3 verify_keys.py
```

### 4. 解密数据库

```bash
python3 decrypt_db.py
```

解密后的数据库保存到 `decrypted/` 目录，可以用 `DB Browser for SQLite` 等工具直接打开。

### 5. 导出聊天记录

```bash
# 列出所有会话
python3 export_messages.py

# 导出指定会话
python3 export_messages.py -c <username>

# 导出指定会话最近 N 条
python3 export_messages.py -c <username> -n 50

# 导出所有会话
python3 export_messages.py --all
```

导出的消息保存到 `exported/` 目录。

## 密钥提取原理

微信使用 [WCDB](https://github.com/Tencent/wcdb)（SQLCipher 封装）加密本地数据库，每个数据库有独立的 enc_key 和 salt。WCDB 会在进程内存中缓存 raw key，格式为 `x'<64hex_enc_key><32hex_salt>'`。

**内存扫描方式** (`find_key_memscan.py`): 扫描微信进程的全部可读内存，搜索上述格式的字符串，通过 salt 匹配数据库文件，再用 HMAC-SHA512 验证密钥正确性。

**断点方式** (`find_key.py`): 在 `setCipherKey` 函数（通过 `malloc(67)` 调用模式定位）上设置断点，微信打开数据库时从寄存器中读取密钥。

详细分析见 [微信数据库解密](./微信数据库解密.md)。

## Thanks

- [ylytdeng/wechat-decrypt](https://github.com/ylytdeng/wechat-decrypt) — Windows 版内存搜索方案，本项目的 `find_key_memscan.py` 参考了其实现
