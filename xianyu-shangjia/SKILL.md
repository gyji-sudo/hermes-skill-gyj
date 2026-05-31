---
name: xianyu-shangjia
description: 闲鱼上架辅助。自适应 Edge 路径、窗口尺寸、按钮位置。确认需求 → 点发闲置 → 确认文案 → 生成/上传图片。自己选分类和点发布。
---

# 闲鱼上架

自适应版，在不同电脑上也能用。

## 前置条件

- Windows + Edge
- Python 3 + `websockets`
- 闲鱼已登录

```bash
pip install websockets
```

## 步骤 0：确认需求

问用户：卖什么、价位、需要生成图还是自带图、文案自己写还是要框架。

## 步骤 1：启动 Edge（自适应）

自动检测 Edge 安装路径 + 最大化窗口确保坐标稳定：

```bash
# 自动找 Edge，找不到就让用户输入
for %p in (
  "C:\Program Files\Microsoft\Edge\Application\msedge.exe"
  "C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe"
  "%LOCALAPPDATA%\Microsoft\Edge\Application\msedge.exe"
) do if exist %p set EDGE_PATH=%~p

if not defined EDGE_PATH (
  echo 未找到 Edge，请输入完整路径：
  set /p EDGE_PATH=
)

start "" %EDGE_PATH% --remote-debugging-port=9222 --new-window --start-maximized https://www.goofish.com
```

Python 版（脚本内自动检测）：

```python
import os, subprocess

def find_edge():
    paths = [
        r"C:\Program Files\Microsoft\Edge\Application\msedge.exe",
        r"C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe",
        os.path.expandvars(r"%LOCALAPPDATA%\Microsoft\Edge\Application\msedge.exe"),
    ]
    for p in paths:
        if os.path.exists(p):
            return p
    # 注册表查询（更准）
    try:
        import winreg
        with winreg.OpenKey(winreg.HKEY_LOCAL_MACHINE, r"SOFTWARE\Microsoft\Windows\CurrentVersion\App Paths\msedge.exe") as key:
            return winreg.QueryValue(key, None)
    except:
        pass
    return None

edge = find_edge()
if not edge:
    edge = input("未找到 Edge，请输入路径: ")
subprocess.Popen([edge, "--remote-debugging-port=9222", "--new-window", "--start-maximized", "https://www.goofish.com"])
```

## 步骤 2：点发闲置（自适应坐标）

检测窗口大小 → 用 XPath 搜"发闲置" → 如果找不到，根据窗口宽度计算按钮位置（闲鱼"发闲置"在右上角约 95% 宽度位置）：

```python
async def find_publish_btn(ws_url):
    # 先 XPath 搜
    pos = await xpath_find(ws_url, "发闲置")
    if pos: return pos
    
    # XPath 失败 → 根据窗口宽度自适应
    async with websockets.connect(ws_url) as ws:
        await cdp(ws, "Page.enable")
        r = await cdp(ws, "Runtime.evaluate", {"expression": "window.innerWidth"})
        w = r.get("result",{}).get("result",{}).get("value", 1920)
        # 发闲置按钮在右上角，约 95% 宽度、距顶部 40px 区域
        return {"x": int(w * 0.95), "y": 80}
```

## 步骤 3：确认文案

问用户：标题/描述/价格是自己写、我出框架、还是我直接写好你复制。

## 步骤 4：生成/上传图片

```python
async def upload_img(ws_url, img_path):
    if not os.path.exists(img_path): return False
    async with websockets.connect(ws_url) as ws:
        await cdp(ws, "Page.enable", msg_id=2)
        await asyncio.sleep(1)
        # 找 input[type=file]（稳定）
        r = await cdp(ws, "DOM.getDocument",{"depth":-1}, msg_id=3)
        did = r.get("result",{}).get("root",{}).get("nodeId")
        r = await cdp(ws, "DOM.querySelector",{"nodeId":did,"selector":"input[type=file]"}, msg_id=4)
        fid = r.get("result",{}).get("nodeId")
        if fid and fid>0:
            await cdp(ws, "DOM.setFileInputFiles",{"nodeId":fid,"files":[os.path.abspath(img_path)]}, msg_id=5)
            await asyncio.sleep(2)
            return True
    return False
```

## 步骤 5：你手动

选分类 + 填内容 + 点发布（约 30 秒）

## 完整脚本

```python
import asyncio, json, websockets, urllib.request, base64, os, http.client, subprocess, winreg

# ===== 自适应工具 =====

def find_edge():
    paths = [
        r"C:\Program Files\Microsoft\Edge\Application\msedge.exe",
        r"C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe",
        os.path.expandvars(r"%LOCALAPPDATA%\Microsoft\Edge\Application\msedge.exe"),
    ]
    for p in paths:
        if os.path.exists(p): return p
    try:
        with winreg.OpenKey(winreg.HKEY_LOCAL_MACHINE, r"SOFTWARE\Microsoft\Windows\CurrentVersion\App Paths\msedge.exe") as k:
            return winreg.QueryValue(k, None)
    except: pass
    return None

async def cdp(ws, method, params=None, msg_id=1):
    if params is None: params = {}
    cmd = json.dumps({"id": msg_id, "method": method, "params": params})
    await ws.send(cmd)
    async for msg in ws:
        data = json.loads(msg)
        if data.get("id") == msg_id: return data

async def xpath_find(ws_url, text):
    async with websockets.connect(ws_url) as ws:
        await cdp(ws, "Page.enable")
        r = await cdp(ws, "Runtime.evaluate", {
            "expression": f"""(()=>{{var x="//*[contains(text(), '{text}')]";var r=document.evaluate(x,document,null,XPathResult.ORDERED_NODE_SNAPSHOT_TYPE,null);for(var i=0;i<r.snapshotLength;i++){{var e=r.snapshotItem(i);var b=e.getBoundingClientRect();if(b.width>20&&b.width<300&&b.height>10&&b.height<200)return{{x:Math.round(b.left+b.width/2),y:Math.round(b.top+b.height/2)}}}}return null}})()"""
        })
        return r.get("result",{}).get("result",{}).get("value")

async def find_btn(ws_url, text):
    pos = await xpath_find(ws_url, text)
    if pos: return pos
    # XPath 失败 → 自适应：按钮在右上角约95%宽度处
    async with websockets.connect(ws_url) as ws:
        await cdp(ws, "Page.enable")
        r = await cdp(ws, "Runtime.evaluate", {"expression": "window.innerWidth"})
        w = r.get("result",{}).get("result",{}).get("value", 1920)
    return {"x": int(w * 0.92), "y": 70}

async def click(ws_url, x, y):
    async with websockets.connect(ws_url) as ws:
        await cdp(ws, "Page.enable")
        await cdp(ws, "Input.dispatchMouseEvent",{"type":"mousePressed","x":x,"y":y,"button":"left","clickCount":1})
        await asyncio.sleep(0.1)
        await cdp(ws, "Input.dispatchMouseEvent",{"type":"mouseReleased","x":x,"y":y,"button":"left","clickCount":1})

async def upload_img(ws_url, img_path):
    if not os.path.exists(img_path): return False
    async with websockets.connect(ws_url) as ws:
        await cdp(ws, "Page.enable", msg_id=2)
        await asyncio.sleep(1)
        r = await cdp(ws, "DOM.getDocument",{"depth":-1}, msg_id=3)
        did = r.get("result",{}).get("root",{}).get("nodeId")
        r = await cdp(ws, "DOM.querySelector",{"nodeId":did,"selector":"input[type=file]"}, msg_id=4)
        fid = r.get("result",{}).get("nodeId")
        if fid and fid>0:
            await cdp(ws, "DOM.setFileInputFiles",{"nodeId":fid,"files":[os.path.abspath(img_path)]}, msg_id=5)
            await asyncio.sleep(2)
            return True
    return False

async def screenshot(html_path, out_path, w=1000, h=700):
    conn=http.client.HTTPConnection("localhost",9222)
    conn.request("PUT",f"/json/new?file:///{html_path.replace(chr(92),'/')}")
    tab=json.loads(conn.getresponse().read())
    conn.close()
    async with websockets.connect(tab["webSocketDebuggerUrl"]) as ws:
        await cdp(ws,"Emulation.setDeviceMetricsOverride",{"width":w,"height":h,"deviceScaleFactor":2,"mobile":False},msg_id=2)
        await asyncio.sleep(1)
        r=await cdp(ws,"Page.captureScreenshot",{"format":"png"},msg_id=3)
        d=r.get("result",{}).get("data","")
        if d:
            with open(out_path,"wb") as f: f.write(base64.b64decode(d))
            return True

# ===== 主流程（含重试） =====

async def publish(html_card_path=None, image_path=None, retry=2):
    # 生成卡片
    if html_card_path and not image_path:
        image_path = html_card_path.replace('.html', '.png')
        await screenshot(html_card_path, image_path)
        print(f"卡片: {image_path}")

    # 启动 Edge（如果不在运行）
    for attempt in range(retry + 1):
        try:
            urllib.request.urlopen("http://localhost:9222/json/version", timeout=2)
            break
        except:
            if attempt == retry:
                print("Edge CDP 未启动，请先启动 Edge")
                return False
            print("CDP 未就绪，尝试启动 Edge...")
            edge = find_edge()
            if edge:
                subprocess.Popen([edge, "--remote-debugging-port=9222", "--new-window", "--start-maximized", "https://www.goofish.com"])
                await asyncio.sleep(6)
            else:
                print("未找到 Edge")
                return False

    # 点发闲置
    for attempt in range(retry + 1):
        resp = json.loads(urllib.request.urlopen("http://localhost:9222/json").read())
        hp = None
        for t in resp:
            if t['type']=='page' and 'goofish.com' in t['url'] and 'publish' not in t['url']: hp=t; break
        
        if not hp:
            print("未找到首页，请先登录")
            return False

        pos = await find_btn(hp["webSocketDebuggerUrl"], "发闲置")
        if pos:
            await click(hp["webSocketDebuggerUrl"], pos['x'], pos['y'])
            break
        print(f"找按钮失败，第 {attempt+1} 次重试...")
        await asyncio.sleep(2)
    
    await asyncio.sleep(5)

    # 上传图片
    resp2 = json.loads(urllib.request.urlopen("http://localhost:9222/json").read())
    pt = None
    for t in resp2:
        if 'publish' in t['url']: pt=t; break
    if not pt:
        print("发布页未打开，可能是分类限制，请用 App 发布")
        return False
    await asyncio.sleep(5)

    if image_path:
        ok = await upload_img(pt["webSocketDebuggerUrl"], image_path)
        print(f"图片: {'OK' if ok else '失败'}")

    print("\n✅ 完成！手动操作：选分类 → 填内容 → 点发布")
    return True

asyncio.run(publish(html_card_path="card.html"))
```
