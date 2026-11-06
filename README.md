# rust-use
rust 的使用心得

# 目录

- [crates](#crates)
  - [错误处理](#错误处理)
    - [anyhow](#anyhow)
    - [thiserror](#thiserror)
  - [日志](#日志)
    - [log](#log)
    - [env_logger](#env_logger)
    - [fern](#fern)
    - [flexi_logger](#flexi_logger)
  - [配置](#配置)
    - [serde](#serde)
    - [toml](#toml)
  - [派生宏与代码生成](#派生宏与代码生成)
    - [strum](#strum)
    - [getset](#getset)
  - [映射库](#映射库)
    - [dashmap](#dashmap)
    - [indexmap](#indexmap)
  - [其他库](#其他库)
    - [scopeguard](#scopeguard)
    - [rust-i18n](#rust-i18n)
    - [fastrand](#fastrand)
    - [deformat](#deformat)
    - [infer](#infer)
    - [strsim](#strsim)

---

# crates

## 错误处理
- ### [anyhow](https://crates.io/crates/anyhow)
    统一处理及自定义错误为 `anyhow::Result<T>`
- ### [thiserror](https://crates.io/crates/thiserror)
    提供 `#[derive(thiserror::Error, Debug)]`  和 `#[error("your error")] ` 自定义用于作为错误的枚举 **(Enum)**

## 日志
- ### [log](https://crates.io/crates/log)
```toml
[dependencies]
# release 模式下关闭日志
log = { version = "*", features = ["release_max_level_off"] }
```


- ### [env_logger](https://crates.io/crates/env_logger)
    打印日志到终端
```toml
[dependencies]
env_logger = { version = "*", default-features = false, features = ["auto-color", "humantime"] }
```

```rust
fn main() {
    env_logger::Builder::from_env(env_logger::Env::default().default_filter_or("info"))
        .format_source_path(true) // 显示打印所在的文件路径和行号
        .init();

    // ...
}
```

- ### [fern](https://crates.io/crates/fern)
    日志输出到文件（已停更）

- ### [flexi_logger](https://crates.io/crates/flexi_logger)
    日志输出到文件（未尝试）




## 配置

- ### [serde](https://crates.io/crates/serde)
    提供 **序列化（Serialize）** 和 **反序列化（Deserialize）** 的 **Traits**

- ### [toml](https://crates.io/crates/toml)
    编码和解码 toml 文件 

```toml
[dependencies]
serde= { version = "*", features = ["derive"] }
toml = "*"
```

```rust
use std::path::PathBuf;
use std::sync::LazyLock;
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
pub struct Config {
    #[serde(rename = "tray")] // 将 .toml 中的该字段重命名为 tray
    pub tray_options: ...,
    #[serde(rename = "notify")] // 将 .toml 中的该字段重命名为 notify
    pub notify_options: ...,
    // ...
}

static CONFIG_PATH: LazyLock<PathBuf> = LazyLock::new(|| { /* .. */ });

impl Config {
    pub fn open() -> Result<Self> {
        if let Ok(content) = std::fs::read_to_string(CONFIG_PATH) {
            toml::from_str(&content)
        } else {
            let default_config = Config::default();
            let toml_str = toml::to_string_pretty(&default_config)?;
            std::fs::write(CONFIG_PATH, toml_str)?;
            Ok(default_config)
        }
    }

    pub fn save(&self) -> anyhow::Result<()> {
        let toml_str = toml::to_string_pretty(self)?;
        std::fs::write(CONFIG_PATH, toml_str)
    }
}
```




## 派生宏与代码生成

- ### [strum](https://crates.io/crates/strum)
    为 **枚举（Enum）** 提供字段转字符，字符找枚举，迭代枚举等方法
    可以用给一些特殊的库比如创建托盘菜单库，实现通过枚举（**枚举** 转 **字符** 作为 **菜单 ID**）来创建菜单，当菜单事件发生时可通过返回的 **菜单ID** 执行枚举的`from_str()` 找到对应的枚举，最后来执行相应的方法  
```rust
use std::convert::AsRef;
use std::str::FromStr;

// AsRefStr：枚举转字符，默认区分大小写
// EnumString：字符找枚举，默认区分大小写
#[derive(strum::AsRefStr, strum::EnumString)]
// #[strum(prefix = "/")] // 枚举转字符时添加前缀（注意：字符找枚举时忽略前缀）
// #[strum(suffix = ".rs")] // 枚举转字符时添加后缀（注意：字符找枚举时忽略后缀）
// #[strum(ascii_case_insensitive)] // 忽略大小写
// #[strum(serialize_all = "snake_case")] // 所有枚举转换的字符均为小写下划线格式，序列化和 to_string 优先级大于这个
// #[strum(serialize_all = "title_case")] // 所有枚举转换的字符均为两个字符间隔开来，如 HelloWorld 变 Hello World
// #[strum(serialize_all = "UPPERCASE")] // 所有枚举转换的字符均为大写格式
enum YourEnum {
    #[strum(serialize = "r", to_string = "red")] // 转字符时优先使用 to_string 的内容
    Red,
    #[strum(serialize = "orange", serialize = "o")] // 序列化为指定字符（同时存在两个序列化，默认转字符为首个序列化字符）
    Orange,
    #[strum(disabled)] // 禁用该字段被找到
    Yellow,
    #[strum(transparent)] // 如果字段内还有其他字符，则字段转字符时使用的是该字段里面的字符
    Green(String),
    #[strum(transparent)] // 如果字段内还有字段（里面的字段也设置了 ASRefStr ），则字段转字符时使用的是最里面的字段
    Blue(OtherEnum /* [derive(strum::AsRefStr)] */),
}

// Enum.as_ref() 和 Enum.to_string() 的区别，as_ref()会返回设置了前缀后缀的字符串，to_string()会返回通过 to_string设置的字符
```

```rust
use strum::{EnumIter, IntoEnumIterator};

#[derive(EnumIter)]
enum YourEnum { /* ... */ }

YourEnum::iter().for_each(|v| /* ... */ );
```

- ### [getset](https://crates.io/crates/getset)
    为 **结构体（Struct）** 的字段提供 获取（get） 和 设置（set）的各种方法

```rust

pub struct YourStuct {
    #[getset(get = "pub", set = "pub")] // get_copy = "pub"
    pub field: i32,
}

fn main() {
    let mut your_struct = YourStuct { field: 0 };

    your_struct.set_field(114514_i32);

    let field = your_struct.field();

    assert_eq!(field, 114514);
}
```





## 映射库

- ### [dashmap](https://crates.io/crates/dashmap)
    高并发的哈希表，用来代替 `Arc<RwLock<HashMap>>`
```toml
[dependencies]
dashmap = "=7.0.0-rc2"
```

```rust
use std::sync::{Arc, LazyLock};
use dashmap::DashMap;
// use dashmap::DashSet;

// 全局哈希表
static GLOBAL_MAP: LazyLock<DashMap<K, V>> = LazyLock::new(|| DashMap::new());

// 线程安全的哈希表
let map = Arc::new(DashMap::new());
```

- ### [indexmap](https://crates.io/crates/indexmap)
    有顺序的哈希表，可通过 `get_index(&self, index: usize)` 获取 `Option<(&K, &V)>`，注意它没有实现 Sync trait






## 其他库
- ### [scopeguard](https://crates.io/crates/scopeguard)
    继承 `Drop` trait，更高度灵活通过 `scopeguard::defer!()` 或 `scopeguard::guard(f, |f| { /* clean */ })` 实现离开作用域或发生 panic 时的清理工作
- ### [rust-i18n](https://crates.io/crates/rust-i18n)
    文本国际化库
- ### [fastrand](https://crates.io/crates/fastrand)
    获取随机数库
- ### [deformat](https://crates.io/crates/deformat)
    将文档转换为纯文本，如 HTML、PDF、DOCX、EPUB、XLS、XML、PPTX、RTX 等，可以用于剪切板的使用
- ### [infer](https://crates.io/crates/infer)
    推断文件类型库，暂不支持推断 SVG 文件
- ### [strsim](https://crates.io/crates/strsim)
    比较字符串的相似度，有丰富的比较算法