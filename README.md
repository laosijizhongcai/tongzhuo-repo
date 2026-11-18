# 同桌 · 距离即权限

What If Lab AI 硬件创意黑客松参赛作品。

一块桌面装置，用超声波实时测量人与它的距离，把「物理上的靠近」翻译成
「信息上的完整」：95 厘米外它保持沉默，中距离只给一句明显没说完的话，
凑近到 38 厘米内才肯说出完整答案——你一退开，话当场被没收。

## 它解决的

内容被设计成零成本获取，于是注意力被拉平、没有内容值得人走过去。
「同桌」把获取门槛重新设回身体的距离：想要 AI 说完整，就用身体来换。

## 仓库结构

```
firmware/          设备固件（MicroPython / QuecPython）
  main.py          主循环：超声测距 → 档位状态机 → 分档反馈 → MQTT 上行
  config.py        引脚 / 阈值 / 上报节拍 / 服务器地址
  lib/
    ultrasonic.py  HC-SR04 驱动（量程/超时/温度补偿三处对官方示例的修正）
    proximity.py   双阈值滞回 + 升档确认 + 降档即时的三档判定器
    session.py     学习会话状态机（在座/前倾/离开，可复用于台灯等场景）
    cloud.py       MQTT 上行 + 行为判定/文案拉取 + 离线缓存
    display.py     RGB 分档变色 + 蜂鸣 + 心跳灯
    renderer.py    分层文案渲染
    answer_store.py 三档答案本地缓存
    env.py         温湿度采集（声速补偿用）
cloud/             云端服务（FastAPI + SQLite）
  app.py           state 入库、/api/stats 统计、/api/judge 行为判定、下行调参
  behavior_judge.py轨迹 → 大模型行为判定（路过/犹豫/试探/结伴/赖着）
  llm_provider.py  OpenAI 兼容接口 + 失败降级本地规则
  static/          现场看板（档位、实时距离、事件流、当日人气）
mosquitto.conf     MQTT broker 配置（演示用匿名接入）
docs/              架构图、接线说明
```

## 数据链路

```
HC-SR04(回波经1k/2k分压) → 板端100ms测距 → 窗口最小值滤波+温度补偿 →
双阈值滞回判定三档 → 灯/蜂鸣/分层文案即时反馈；1s 节拍 MQTT 经 4G 上云 →
云端入库 → 看板实时呈现 + 大模型行为判定（失败退缓存文案，徽章如实标注）
```

## 三档权限模型

| 档位 | 距离 | 权限 | 反馈 |
|---|---|---|---|
| 0 OUTSIDE | > 115cm | 只有省略号 | 灯灭，不理你 |
| 1 MID | 52–95cm | 一句话钩子 | 黄灯 |
| 2 CLOSE | < 38cm | 完整答案 + 行为点评 | 绿灯 + 蜂鸣 |

进/退阈值分离（滞回死区）防临界抖动；升档连验 4 帧、降档 1 帧即切——
「被收话」的瞬间是观众主动重复动作的理由。

## 联调排障实录（真实工作量）

EG800Z 官方文档之外踩到的坑，全部有实测复现路径与串口日志：
umqtt 必须先 set_callback 再 subscribe（官方示例顺序相反）；publish 路径的
check_msg() 无下行时阻塞饿死采样循环；后台线程 HTTP 与 MQTT 并发导致硬复位；
本固件 utime.time() 返回 uptime，墙钟须用 localtime()；Echo 5V 必须分压。
详见 `docs/`。

## 快速开始

设备：QPYcom 传 `main.py`、`config.py` 到 `/usr`，`lib/` 到 `/usr/lib/`；
在 `config.py` 填 `MQTT_HOST`、`API_BASE`；上电自启，开机自动注册看板，零配网。

云端：`cd cloud && uvicorn app:app --port 8000`（或 Docker）。
看板地址即服务地址。

## 边界声明（与代码同源）

不拍照、不录音、不存身份；上报的全部数据是距离、档位、时间戳。
「性格点评」是拟人表演不是画像；距离是超声物理测量，不识别「人」。
MQTT 匿名、看板无鉴权是 48 小时演示环境取舍，生产化需 TLS 与设备认证。

## 许可

MIT
