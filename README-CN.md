# C2Rust Agent

基于大语言模型的智能C项目到Rust项目转换工具

[![Rust](https://img.shields.io/badge/rust-1.70+-orange.svg)](https://www.rust-lang.org)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Build Status](https://img.shields.io/badge/build-passing-brightgreen.svg)]()

## 项目概述

C2Rust Agent 是一个先进的代码翻译系统，利用大语言模型（LLM）将C项目转换为安全、惯用的Rust代码。与简单的转译器不同，该工具能够理解代码语义并生成人类可读、可维护的Rust代码，遵循最佳实践。

## 核心特性

- **智能翻译**: 使用LLM理解C代码语义，生成惯用的Rust代码
- **项目级分析**: 处理整个C项目，保持结构和关系
- **上下文感知**: 利用数据库存储的代码关系提高翻译质量
- **迭代优化**: 基于错误反馈自动重试翻译，直到代码能够编译
- **多种项目类型**: 支持单文件、配对头文件/源文件和多模块项目
- **LSP集成**: 提供语言服务器协议支持，便于IDE集成
- **数据库后端**: 存储代码分析和翻译历史

## 系统架构

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   C项目分析     │───▶│   LSP服务       │───▶│   数据库存储    │
│                 │    │  (代码索引)     │    │(SQLite+Qdrant) │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                                              │
         ▼                                              ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   预处理器      │───▶│   主处理器      │◀───│  提示构建器     │
│  (缓存&映射)    │    │  (翻译协调)     │    │  (上下文生成)   │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │
         ▼                       ▼
┌─────────────────┐    ┌─────────────────┐
│   文件扫描器    │    │   LLM服务       │
│  (项目发现)     │    │  (代码翻译)     │
└─────────────────┘    └─────────────────┘
                                │
                                ▼
                     ┌─────────────────┐
                     │  Rust检查器     │
                     │  (代码验证)     │
                     └─────────────────┘
```

### 核心组件

- **LSP服务**: 分析C代码结构和关系
- **数据库服务**: 在SQLite + Qdrant向量数据库中存储代码分析
- **预处理器**: 生成缓存和文件映射，拆分为编译单元
- **主处理器**: 协调翻译工作流程，包含重试逻辑
- **提示构建器**: 为LLM翻译生成上下文感知的提示
- **LLM请求器**: 与语言模型接口进行代码翻译
- **Rust检查器**: 验证生成的Rust代码编译

## 安装指南

### 系统要求

1. **Rust编程语言**: 从 [rustup.rs](https://rustup.rs/) 安装
   ```bash
   curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
   ```

2. **Qdrant向量数据库**: 用于语义代码搜索
   ```bash
   # 使用Docker
   docker run -p 6333:6333 qdrant/qdrant
   
   # 或本地安装
   # 参见: https://qdrant.tech/documentation/guides/installation/
   ```

3. **LLM API访问**: 配置OpenAI GPT、Claude或其他兼容模型的访问

### 源码构建

```bash
# 克隆仓库
git clone https://github.com/rust4c/c2rust_agent.git
cd c2rust_agent

# 构建所有组件
cargo build --release

# 运行测试
cargo test
```

## 配置

创建 `config/config.toml`:

```toml
[database]
sqlite_path = "data/c2rust.db"
qdrant_url = "http://localhost:6333"

[llm]
provider = "openai"  # 或 "claude", "local"
api_key = "your-api-key-here"
model = "gpt-4"

[translation]
max_retries = 3
concurrent_limit = 4
cache_dir = "cache"

[logging]
level = "info"
file = "logs/c2rust.log"
```

## 使用方法

### 命令行界面

```bash
# 分析并翻译C项目
c2rust-agent translate /path/to/c/project --output /path/to/rust/project

# 使用数据库上下文
c2rust-agent translate /path/to/c/project --with-db --output ./rust_output

# 试运行（仅分析）
c2rust-agent analyze /path/to/c/project --dry-run
```

### 编程接口

```rust
use main_processor::{MainProcessor, ProjectInfo, ProjectType};

#[tokio::main]
async fn main() -> Result<()> {
    // 初始化处理器
    let processor = MainProcessor::new("./cache").await?;
    
    // 创建项目信息
    let project = ProjectInfo {
        name: "my_c_project".to_string(),
        path: "/path/to/c/project".into(),
        project_type: ProjectType::PairedFiles,
    };
    
    // 运行翻译
    let stats = processor.run_translation_workflow().await?;
    println!("翻译完成: {:?}", stats);
    
    Ok(())
}
```

## 翻译流程

1. **发现阶段**: 扫描C项目结构，识别编译单元
2. **分析阶段**: 使用LSP服务理解代码关系和依赖
3. **缓存阶段**: 预处理器创建优化的缓存和文件映射
4. **上下文构建**: 使用数据库知识生成丰富的上下文提示
5. **翻译阶段**: LLM基于语义理解将C代码转换为Rust
6. **验证阶段**: Rust编译器检查生成的代码
7. **优化阶段**: 如果编译失败，基于错误反馈自动重试

### 支持的项目类型

- **单文件**: 简单的C程序 (main.c → main.rs)
- **配对文件**: 头文件/源文件配对 (.h/.c → lib.rs + 模块)
- **多模块**: 包含多个独立模块的复杂项目

### 翻译特性

- 通过Rust所有权系统实现内存安全
- 使用 `Result<T, E>` 类型处理错误
- 对可空指针正确使用 `Option<T>`
- 惯用的Rust模式（迭代器、模式匹配）
- 在需要时自动添加 `unsafe` 块注释
- 为C兼容结构体添加 `#[repr(C)]`

## 示例

### 输入C代码
```c
#include <stdio.h>
#include <stdlib.h>

typedef struct {
    int x, y;
} Point;

Point* create_point(int x, int y) {
    Point* p = malloc(sizeof(Point));
    if (p == NULL) return NULL;
    p->x = x;
    p->y = y;
    return p;
}

void free_point(Point* p) {
    free(p);
}
```

### 生成的Rust代码
```rust
#[repr(C)]
pub struct Point {
    pub x: i32,
    pub y: i32,
}

impl Point {
    pub fn new(x: i32, y: i32) -> Self {
        Point { x, y }
    }
}

pub fn create_point(x: i32, y: i32) -> Option<Box<Point>> {
    Some(Box::new(Point::new(x, y)))
}

// 注意: 不需要free_point - Rust自动处理内存管理
```

## 开发

### 项目结构

```
c2rust_agent/
├── crates/
│   ├── commandline_tool/     # 命令行接口
│   ├── cproject_analy/       # C项目分析
│   ├── db_services/          # 数据库操作
│   ├── env_checker/          # 环境验证
│   ├── file_scanner/         # 文件发现
│   ├── llm_requester/        # LLM API接口
│   ├── lsp_services/         # 语言服务器集成
│   ├── main_processor/       # 核心翻译逻辑
│   ├── prompt_builder/       # 上下文感知提示生成
│   ├── rust_checker/         # Rust代码验证
│   └── ui_main/             # 图形界面
├── config/                   # 配置文件
├── test-projects/           # 测试用例
└── target/                  # 构建产物
```

### 运行测试

```bash
# 运行所有测试
cargo test

# 运行特定包的测试
cargo test -p main_processor

# 带日志运行
RUST_LOG=debug cargo test
```

## 贡献

1. Fork 本仓库
2. 创建特性分支: `git checkout -b feature/amazing-feature`
3. 提交更改: `git commit -m 'Add amazing feature'`
4. 推送到分支: `git push origin feature/amazing-feature`
5. 开启Pull Request

## 限制

- 目前支持C99标准（C11/C18特性开发中）
- 复杂宏可能需要手动调整
- 内联汇编不会自动翻译
- 某些平台特定代码可能需要手动审查

## 许可证

本项目采用MIT许可证 - 详见 [LICENSE](LICENSE) 文件。

## 致谢

- [c2rust](https://github.com/immunant/c2rust) - 转译方法的启发
- [tree-sitter](https://tree-sitter.github.io/) - 代码解析技术
- [Qdrant](https://qdrant.tech/) - 语义搜索向量数据库
- Rust社区提供的优秀工具和库

## 支持

- 📖 [文档](https://github.com/rust4c/c2rust_agent/wiki)
- 🐛 [问题反馈](https://github.com/rust4c/c2rust_agent/issues)
- 💬 [讨论区](https://github.com/rust4c/c2rust_agent/discussions)
- 📧 联系邮箱: rust4c@example.com

## 路线图

- [ ] 支持C11/C18标准
- [ ] 改进宏翻译
- [ ] 添加更多LLM提供商支持
- [ ] 图形用户界面完善
- [ ] 性能优化
- [ ] 增量翻译支持
- [ ] 翻译质量评估工具

---

*本文档持续更新中，如有问题请提交Issue或参与Discussion讨论。*