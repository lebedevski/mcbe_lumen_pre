# Lumen后处理前置开发指南（26.1）

## 1. 文档概述

### 1.1 前言
本文档详细说明**Lumen后处理系统**的前置配置流程，包含核心配置文件结构、UI绑定规则及跨Pass参数传递配置。适用于网易我的世界基岩版平台的光影开发者。

### 1.2 基本信息
- **配置文件语法**：JSON（**不支持** `//` 或 `/**/` 等注释）
- **前置开发者**：白上青天
- **平台**：网易我的世界基岩版

## 2. 文件结构说明

光影资源包中的配置文件应遵循以下目录结构：

```
光影资源包/
└── modconfigs/
    └── postprocess/
        ├── config.json       # 主配置文件（UI设置面板）
        ├── pass.json         # 渲染Pass配置
        └── para.json         # 参数传输配置
```

## 3. 基础配置 (config.json)

### 3.1 配置示例
```json
{
  "name": "ExampleConfig",
  "config": [
    {
      "title": "ExampleGroup",
      "setting": [
        {
          "text": "§e§lExample控制面板 §r§f",
          "default": "",
          "bind": "example_main"
        },
        {
          "text": "对比度控制",
          "sub": "开启对比度控制",
          "default": false,
          "bind": "example_contrast_enable"
        },
        {
          "text": "对比度",
          "sub": "控制画面明暗对比度",
          "default": 0.5,
          "bind": "example_contrast"
        }
      ]
    }
  ]
}
```

### 3.2 关键字段说明
| 字段 | 类型 | 必填 | 说明 | 示例 |
|------|------|------|------|------|
| name | string | 是 | 光影ID，需唯一 | "ExampleConfig" |
| text | string | 是 | 显示文本（支持颜色代码） | "§e对比度控制" |
| sub | string | 否 | 次级描述文本 | "控制画面明暗对比度" |
| default | 多种 | 是 | 默认值，决定控件类型 | 0.5(滑动条)、true(开关) |
| bind | string | 是 | 绑定参数ID（需全局唯一） | "example_contrast" |

### 3.3 UI控件类型映射规则
`default` 字段的值类型自动决定设置面板中显示的控件类型：
- `number` → **滑动条**
- `boolean` → **开关**
- `string` → **文本输入框**

## 4. 渲染Pass配置 (pass.json)

### 4.1 标准结构示例
```json
[
  {
    "name": "example_pass",
    "paras": [
      {
        "name": "example_brightness",
        "value": 0.0,
        "range": [0.0, 1.0]
      },
      {
        "name": "example_contrast", 
        "value": 0.5,
        "range": [0.0, 1.0]
      }
    ],
    "pass_array": [
      {
        "render_target": {
          "width": 1.0,
          "height": 1.0
        },
        "material": "example_shader"
      }
    ]
  }
]
```

### 4.2 关键配置项说明
| 字段 | 类型 | 说明 | 注意事项 |
|------|------|------|----------|
| name | string | 后处理Pass名称 | 需唯一，用于跨Pass引用 |
| paras | array | 参数绑定数组 | 每个参数需与config.json中bind对应 |
| pass_array | array | 渲染Pass组 | 按顺序执行渲染操作 |
| material | string | 着色器材料名称 | 对应游戏内材质文件 |

**推荐参考**：https://mc.163.com/dev/mcmanual/mc-dev/mcguide/16-美术/7-材质与着色器/4-材质实战.html

## 5. 参数传输配置 (para.json)

### 5.1 参数映射示例
```json
[
  {
    "name": "example_shader",
    "para": [
      "example_contrast_enable",
      "example_contrast"
    ]
  }
]
```

### 5.2 参数传输规则
1. **顺序一致性**：`para` 数组中的参数顺序需与着色器中的uniform位置严格对应
2. **通道限制**：支持最多16个参数通道传输
3. **数值规范化**：所有数值均被限制在[0,1]范围内
4. **类型转换**：布尔值会自动转换为浮点数（true=1.0, false=0.0）

### 5.3 Uniform位置映射表
| Uniform | X | Y | Z | W |
|---------|---|---|---|---|
| EXTRA_VECTOR1 | 1 | 2 | 3 | 4 |
| EXTRA_VECTOR2 | 5 | 6 | 7 | 8 |
| EXTRA_VECTOR3 | 9 | 10 | 11 | 12 |
| EXTRA_VECTOR4 | 13 | 14 | 15 | 16 |

参数将按照x-y-z-w的顺序分别传递到对应的EXTRA_VECTOR uniform中。

## 6. 跨Pass纹理传递

### 6.1 配置示例
```json
{
  "name": "ExampleComposite",
  "texture": [
    {
      "source": ["postprocess0", 2],
      "target": ["postprocess1", 0],
      "glTexture": 7
    }
  ]
}
```

### 6.2 传递参数说明
| 字段 | 类型 | 说明 | 限制 |
|------|------|------|------|
| source | array | [源Pass名称, 纹理索引] | 源Pass必须在目标Pass之前渲染 |
| target | array | [目标Pass名称, 纹理索引] | 不能覆盖系统关键纹理 |
| glTexture | number | OpenGL纹理ID（0-7） | 在后处理中对应TEXTURE_X |

### 6.3 重要注意事项
- **深度图保护**：如需使用深度图，避免覆盖纹理索引2
- **渲染顺序**：确保源Pass在目标Pass之前执行
- **纹理索引**：glTexture对应着色器中的TEXTURE_X

## 7. 错误排查与常见问题

### 7.1 前置报错信息对照表
| 错误信息 | 可能原因 | 解决思路 |
|----------|----------|----------|
| No postprocess pass | pass.json不存在或路径错误 | 检查文件路径与命名 |
| No postprocess para | para.json不存在或路径错误 | 验证配置文件完整性 |
| Add Post Process Error | 后处理插入失败 | 检查JSON语法与结构 |
| Post Pass Error | 跨pass传递纹理失败 | 验证纹理索引与Pass顺序 |

### 7.2 常见问题解决方案

#### 问题1：后处理未加载
- **检查项**：JSON文件是否符合规范、是否存在语法错误
- **验证步骤**：
  1. 使用JSON验证工具检查语法
  2. 确认material文件配置正确
  3. 检查着色器代码无语法错误

#### 问题2：参数设置未生效
- **检查项**：bind字段一致性、参数类型匹配
- **验证步骤**：
  1. 检查config.json与para.json中bind字段完全一致
  2. 确认参数数据类型匹配（如布尔值vs数值）
  3. 验证数值范围是否符合预期

#### 问题3：设置面板显示异常
- **检查项**：JSON语法、bind字段重复、光影ID冲突
- **验证步骤**：
  1. 检查JSON格式有效性
  2. 确保所有bind字段全局唯一
  3. 确认光影ID不与现有资源冲突

## 8.命名规范
- 使用**snake_case**命名风格（如`lumen_contrast_factor`）
- bind字段添加**项目前缀**防止冲突（如`lumen_`、`example_`）

---
*本文档最后更新：2026年2月1日*  
