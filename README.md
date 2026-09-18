# Prometheus + Alertmanager 飞书告警中转方案
> Alertmanager 原生不支持飞书机器人，用 Flask 做简易 webhook 中转，把告警转成飞书卡片消息。

## 原理
`Prometheus → Alertmanager → Flask中转服务 → 飞书机器人webhook`
Alertmanager 推送告警到 Flask，Flask 组装飞书卡片 JSON，调用飞书机器人接口发消息。支持告警恢复通知。

## 效果展示
![网卡了](./屏幕截图%202026-09-18%20175732.png)

## alertmanager.yml（重点）
```yaml
receivers:
  - name: 'feishu.hook'
    webhook_configs:
      - url: '[http://localhost:4000/alert](http://localhost:4000/alert)' # 必须带http://
        send_resolved: true # 告警恢复也推送
route:
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
  receiver: 'feishu.hook'
```

## Flask 中转脚本 `alert_feishu_hook.py`
```python
from flask import Flask, request, jsonify
from datetime import datetime
import json
import requests

app = Flask(__name__)
webhook_url = "[https://open.feishu.cn/open-apis/bot/v2/hook/xxxxxxxxx](https://open.feishu.cn/open-apis/bot/v2/hook/xxxxxxxxx)" # 替换成你的飞书hook

@app.route('/alert', methods=['POST'])
def receive_alert():
    try:
        payload = request.json
        for alert in payload["alerts"]:
            feishu_body = build_feishu_card(alert)
            requests.post(webhook_url, headers={"Content-Type":"application/json"}, data=feishu_body)
        return jsonify({"code":0}), 200 # 必须返回200，否则alertmanager重试
    except Exception as e:
        return str(e),500

def build_feishu_card(alert):
    # 提取告警信息
    alertname = alert["labels"].get("alertname","未知告警")
    severity = alert["labels"].get("severity","unknown")
    desc = alert["annotations"].get("description","")
    start_at = utc2cn(alert.get("startsAt",""))

    card = {
        "msg_type":"interactive",
        "card":{
            "schema":"2.0",
            "config":{"wide_screen_mode":True},
            "header":{"template":"red","title":{"tag":"plain_text","content":"服务器告警"}},
            "body":{"elements":[
                {"tag":"div","text":{"tag":"lark_md","content":f"**告警类型**：{alertname}"}},
                {"tag":"div","text":{"tag":"lark_md","content":f"**级别**：{severity}"}},
                {"tag":"div","text":{"tag":"lark_md","content":f"**触发时间**：{start_at}"}},
                {"tag":"div","text":{"tag":"lark_md","content":f"**详情**：{desc}"}}
            ]}
        }
    }
    return json.dumps(card)

def utc2cn(utc_str):
    if not utc_str: return "未知"
    utc = datetime.fromisoformat(utc_str.replace("Z","+00:00"))
    return utc.astimezone().strftime("%Y-%m-%d %H:%M:%S")

if __name__ == '__main__':
    app.run(host="0.0.0.0", port=4000)
```
