+++
date = '2025-01-03T15:57:31+08:00'
draft = false
title = 'Windows下游戏手柄的读取'
tags = ['SDL2', 'Rust', 'C++']
+++

> 项目需要，需接入操纵杆、油门外部控制设备，以支持太空漫游相关控制操作。

## 0 概览

设备为罗技的`X56 H.O.T.A.S.`。分为油门和操纵杆两个设备。

## 1 尝试

之前并未接触过这类设备，首先查到的方向是`JoyStick`。因为仿真服务是C++，但我对C++并不是太熟。故找了一下Python相关的读取代码。

然后发现`pygame`有封装好的相关模块，名称即为`joystick`
。示例代码: [https://github.com/pygame/pygame/blob/main/examples/joystick.py](https://github.com/pygame/pygame/blob/main/examples/joystick.py)

不过在此之前，找到了另外一个测试代码：[https://github.com/denilsonsa/pygame-joystick-test](https://github.com/denilsonsa/pygame-joystick-test)

![](https://static.lpe234.xyz/pic/202501031615318.png)

最右侧设备为之前找的一个模拟器，仓库地址：[https://github.com/Cleric-K/vJoySerialFeeder](https://github.com/Cleric-K/vJoySerialFeeder)

但实际使用起来，总感觉有那么一点点的延迟。所以想着在找找其他的。

## 2 继续尝试

然后找到了Rust的 `https://crates.io/crates/gilrs` 库。连接设备，可以接收到事件，但有个很大的问题，就是这个库主要目标应该是为游戏手柄，而不是这种操作杆。

而且示例代码无法读取`gamepad`信息，不过x56应该不能算是游戏手柄。这就导致无法识别设备，应该要区分`油门`和`操纵杆`，代码中没找到相应的接口去获取设备ID等数据。

## 3 再次尝试

然后就发现了。`SDL` [https://www.libsdl.org/](https://www.libsdl.org/)

> Simple DirectMedia Layer is a cross-platform development library designed to provide low level access to audio, keyboard, mouse, joystick, and graphics hardware via OpenGL and Direct3D.


这个项目是C写的。通过静态库编译或者直接调用DLL应该都可以实现。但对C++不太熟，后续找到了 `https://github.com/Rust-SDL2/rust-sdl2` 。

## 4 一些基础知识

### 4.1 USB设备的VID和PID

- VID: VID 是供应商识别码（Vendor ID）的缩写，它是由 USB 实施者论坛（USB Implementers Forum，简称 USB-IF）分配给 USB 设备制造商的唯一标识符。
- PID: PID 是产品识别码（Product ID）的缩写，它是由设备制造商分配给其特定产品的唯一标识符。

### 4.2 SDL2中 GUID

SDL2 中的 GUID 是一个全局唯一标识符，用于标识游戏手柄。它是一个 128 位的数字，通常以十六进制表示。

示例GUID: `030017173807000021a2000000000000` 其中 `VID: 0738` `PID: 2221` 。由此我们可以区分出哪个设备发出的信号(事件Event)

### 4.3 轴、球、苦力帽、按钮

- 轴: Axis 通常表示在一定范围内可连续变化的输入维度。
- 球: Ball 一般在一些老式的游戏设备上会有轨迹球。(老式鼠标?)
- 苦力帽: Hat 也称为方向键或者POV(Point Of View) 通常可以向八个方向推动的开关。上下左右，以及相邻的两个方向的组合。
- 按钮: 最常见的输入设备。提供离散输入，通常只有按下和未按下两种状态。

在`X56`设备中，很多类似`苦力帽`的按钮，被识别为`轴`，其实就是两个轴组成了一个苦力帽。可用有八个方向上的输入变化。

### 4.4 SDL中 游戏设备相关事件

```c
// SDL2-2.30.10/include/SDL_events.h

/* Joystick events */
SDL_JOYAXISMOTION  = 0x600, /**< Joystick axis motion */
SDL_JOYBALLMOTION,          /**< Joystick trackball motion */
SDL_JOYHATMOTION,           /**< Joystick hat position change */
SDL_JOYBUTTONDOWN,          /**< Joystick button pressed */
SDL_JOYBUTTONUP,            /**< Joystick button released */
SDL_JOYDEVICEADDED,         /**< A new joystick has been inserted into the system */
SDL_JOYDEVICEREMOVED,       /**< An opened joystick has been removed */
SDL_JOYBATTERYUPDATED,      /**< Joystick battery level change */
```

### 4.5 如何区分设备及事件来源

在事件Event中，能获取到哪个设备ID发出的事件，在不同的事件类型中也会有相应的按钮ID。

设备ID在`SDL_JOYDEVICEADDED`设备添加时就能获取到，根据该ID可以读取设备的`GUID`等信息。

而按钮的ID，根据实际操作，看响应即可定位。

## 5 完整流程

```text
初始化SDL → 监听事件 → 一般事件 进行响应处理
              ↓
        设备添加/移除事件
              ↓
        设备 添加/移除 集合
```

### 5.1 GUID解析

```rust
/*
 * 解析GUID 获取VID和PID
 *
 * GUID: 030017173807000021a2000000000000
 * VID: 0738
 * PID: A221
 * 返回值: (VID, PID) 大写
 */
pub fn parse_guid(guid_str: &str) -> (String, String) {
    let vid = format!("{}{}", &guid_str[10..12], &guid_str[8..10]);
    let pid = format!("{}{}", &guid_str[18..20], &guid_str[16..18]);

    (vid.to_uppercase(), pid.to_uppercase())
}

#[cfg(test)]
mod test {

    #[test]
    fn test_parse_guid() {
        let guid = super::parse_guid("03003187380700002122000000000000");
        assert_eq!(guid, ("0738".to_string(), "2221".to_string()));
    }
}
```

### 5.2 完整流程

```rust
use sdl2::joystick::{Guid, Joystick};
use sdl2::JoystickSubsystem;

mod utils;

pub struct JoyDevice {
    pub id: u32,
    pub name: String,
    pub guid: Guid,
    pub joystick: Joystick,
}

impl JoyDevice {
    pub fn new(joystick: Joystick) -> Self {
        Self {
            id: joystick.instance_id(),
            name: joystick.name(),
            guid: joystick.guid(),
            joystick,
        }
    }
}

// 读取游戏设备
fn read_joy_device(ins_id: u32, js: &JoystickSubsystem) -> JoyDevice {
    if let Ok(joystick) = js.open(ins_id) {
        JoyDevice::new(joystick)
    } else {
        panic!("Failed to open joystick ID {}", ins_id);
    }
}

// 设备GUID
const GUIDS: &[&str] = &[
    "03003187380700002122000000000000", // 摇杆
    "030017173807000021a2000000000000", // 油门
];

fn main() -> Result<(), String> {
    let sdl_context = sdl2::init()?;

    let joystick_subsystem = sdl_context.joystick()?;

    // 存储所有的游戏设备
    let mut joysticks: Vec<JoyDevice> = Vec::new();

    let mut event_pump = sdl_context.event_pump()?;

    for event in event_pump.wait_iter() {
        use sdl2::event::Event;

        match event {
            Event::JoyAxisMotion {
                which,
                axis_idx,
                value,
                ..
            } => {
                println!("Device {}: Axis {} moved to {}", which, axis_idx, value);
            }
            Event::JoyButtonDown {
                which, button_idx, ..
            } => {
                println!("Device {}: Button {} pressed", which, button_idx);
            }
            Event::JoyButtonUp {
                which, button_idx, ..
            } => {
                println!("Device {}: Button {} released", which, button_idx);
            }
            Event::JoyHatMotion {
                which,
                hat_idx,
                state,
                ..
            } => {
                println!("Device {}: Hat {} moved to {:?}", which, hat_idx, state);
            }
            Event::JoyDeviceRemoved { which, .. } => {
                joysticks.iter().for_each(|joy| {
                    if joy.id == which {
                        println!("Device ({}){} removed", joy.id, joy.name);
                    }
                });
                joysticks.retain(|joy: &JoyDevice| joy.id != which);
            }
            Event::JoyDeviceAdded { which, .. } => {
                let joy_device = read_joy_device(which, &joystick_subsystem);
                if GUIDS.contains(&joy_device.guid.to_string().as_str()) {
                    println!("Device ({}){} added", joy_device.id, joy_device.name);
                    joysticks.push(joy_device);
                }
            }
            Event::Quit { .. } => break,

            // 其他事件
            oe => {
                println!("Other event: {:?}", oe);
            }
        }
    }

    Ok(())
}
```

## 6 后续

既然Rust能写出来，C++应该也不是问题。本打算将代码封装成一个服务，但后续做三维开发的同事说他们来接入，然后就没我啥活了。给他们发了一下SDL C++的代码。

## 7 Gist

[https://gist.github.com/lpe234/9c5a2118a7fa0349b69d03e87683df46](https://gist.github.com/lpe234/9c5a2118a7fa0349b69d03e87683df46)
