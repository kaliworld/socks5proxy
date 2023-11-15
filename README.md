# socks5proxy
这是一个代理池本地客户端，实现了socks的监听和转发，勉强能用

利用http://www.dmdaili.com/的付费代理实现的socks5提取，用于实现绕过waf，默认在127.0.0.1:12345启动监听端口
默认一次获取45个代理，30秒后刷新代理池
```
import socket
import select
import sys
import requests
import random
import time
import threading

BUFFER_SIZE = 8096
API_URL = "http://api.dmdaili.com/dmgetip.asp?apikey=078d3a2f&pwd=01eb294b2c06ae9bc983dafcfa403758&getnum={}&httptype=1&geshi=2&fenge=1&fengefu=&Contenttype=1&operate=all"

class Proxy:
    def __init__(self, addr):
        self.proxy = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        self.proxy.bind(addr)
        self.proxy.listen(10)
        self.inputs = [self.proxy]
        self.route = {}
        self.proxies = []  # 存储代理列表
        self.last_refresh_time = 0
        self.lock = threading.Lock()

    def serve_forever(self):
        while True:
            self.refresh_proxies()  # 刷新代理列表
            readable, _, _ = select.select(self.inputs, [], [])
            for sock in readable:
                if sock == self.proxy:
                    self.on_join()
                else:
                    threading.Thread(target=self.handle_client, args=(sock,)).start()

    def refresh_proxies(self):
        current_time = time.time()
        if current_time - self.last_refresh_time > 30:
            with self.lock:
                # 刷新代理列表
                self.last_refresh_time = current_time
                self.proxies = self.get_proxies()

    def get_proxies(self):
        getnum = 45  # 一次获取45个代理
        test = requests.get(API_URL.format(getnum))
        ip_list = [(i['ip'], i['port']) for i in test.json()['data']]
        print(ip_list)
        return ip_list

    def get_proxy(self):
        if not self.proxies:
            self.refresh_proxies()

        random_number = random.randint(0, len(self.proxies) - 1)
        print("当前使用的IP是", self.proxies[random_number])
        return self.proxies[random_number]

    def on_join(self):
        client, addr = self.proxy.accept()
        print(addr, '连接')

        forward = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        max_retries = 2  # 最大重试次数
        for attempt in range(max_retries):
            try:
                forward.connect(self.get_proxy())
                break  # 如果连接成功，跳出重试循环
            except (TimeoutError, ConnectionRefusedError, OSError):
                if attempt < max_retries - 1:
                    print(f"连接失败，正在重试 ({attempt + 1}/{max_retries})...")
                    time.sleep(1)  # 等待一秒再进行重试
                else:
                    print("连接失败，重试次数已达上限，放弃连接。")
                    client.close()
                    return  # 放弃连接

        with self.lock:
            self.inputs += [client, forward]
            self.route[client] = forward
            self.route[forward] = client

    def handle_client(self, sock):
        try:
            data = sock.recv(BUFFER_SIZE)
            if not data:
                self.on_quit(sock)
            else:
                self.route[sock].send(data)
        except OSError as e:
            if e.errno == 10053 or e.errno == 10038:
                self.on_quit(sock)
            else:
                print(f"Socket error: {e}")

    def on_quit(self, sock):
        if sock in self.inputs:
            target_sock = self.route.get(sock)
            if target_sock:
                try:
                    with self.lock:
                        self.inputs.remove(sock)
                        self.inputs.remove(target_sock)
                        del self.route[sock]
                        del self.route[target_sock]
                        sock.shutdown(socket.SHUT_RDWR)
                        sock.close()
                        target_sock.shutdown(socket.SHUT_RDWR)
                        target_sock.close()
                except OSError as e:
                    # Handle exception (e.g., socket already closed)
                    pass

if __name__ == '__main__':
    try:
        address = ('127.0.0.1', 12345)  # 代理服务器监听的地址
        print(f"----------------正在监听 {address}")
        Proxy(address).serve_forever()
    except KeyboardInterrupt:
        sys.exit(1)

```
特殊情况每次HTTP请求切换一个ip,注意扫描和fuzz使用量
```
import socket 
import select 
import sys 
import requests
import random

def get_proxy():
    getnum = 1
    test = requests.get("http://api.dmdaili.com/dmgetip.asp?apikey=078d3a2f&pwd=01eb294b2c06ae9bc983dafcfa403758&getnum={}&httptype=1&geshi=2&fenge=1&fengefu=&Contenttype=1&operate=all".format(getnum))
    ip_list = []
    ip = test.json()['data']
    for i in ip:
        ip_list.append((i['ip'],i['port']))
    print(ip_list)
    random_number = random.randint(0, getnum-1)
    print("now using ip is",ip_list[random_number])
    return ip_list[random_number]

class Proxy: 
  def __init__(self, addr): 
    self.proxy = socket.socket(socket.AF_INET,socket.SOCK_STREAM) 
    self.proxy.bind(addr) 
    self.proxy.listen(10)
    self.inputs = [self.proxy]
    self.route = {} 
   
  def serve_forever(self): 
    while 1: 
      readable, _, _ = select.select(self.inputs, [], []) 
      for self.sock in readable: 
        if self.sock == self.proxy: 
          self.on_join() 
        else: 
          data = self.sock.recv(8096) 
          if not data: 
            self.on_quit() 
          else: 
            self.route[self.sock].send(data) 
   
  def on_join(self): 
    client, addr = self.proxy.accept() 
    print(addr,'connect')
    forward = socket.socket(socket.AF_INET, socket.SOCK_STREAM) 
    forward.connect(get_proxy()) 
    self.inputs += [client, forward] 
    self.route[client] = forward 
    self.route[forward] = client 
   
  def on_quit(self): 
    for s in self.sock, self.route[self.sock]: 
      self.inputs.remove(s) 
      del self.route[s] 
      s.close()

if __name__ == '__main__': 
  try: 
    address = ('127.0.0.1',12345) #代理服务器监听的地址
    print("----------------now listening in ",address)
    Proxy(address).serve_forever()
  except KeyboardInterrupt: 
    sys.exit(1)
```




