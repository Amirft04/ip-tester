# ip-tester
آموزش کامل ساخت تستر IP برای CDN/VPN در ترموکس (اندروید)
# تستر حرفه‌ای IP برای CDN/VPN در Termux

این پروژه یک تستر حرفه‌ای IP برای اندروید با استفاده از Termux است.

این اسکریپت:
- اتصال TCP را تست می‌کند
- TLS Handshake را بررسی می‌کند
- SNI را برای CDN Fronting تست می‌کند
- IPهای سریع و قابل استفاده را رتبه‌بندی می‌کند
- یک لیست تمیز از بهترین IPها تولید می‌کند

مناسب برای:
- CDN Fronting
- VPN Routing
- Edge IP Testing
- تست IPهای CDN

---

# ویژگی‌ها

- تست سریع و چندنخی (Multi Thread)
- تست واقعی TLS
- تست SNI
- حذف IPهای unusable
- تولید خودکار لیست IP خوب
- مناسب برای Termux روی اندروید

---

# نصب Termux

از گوگل پلی termux رو دانلود و‌ نصب کنید


---

# نصب پیش‌نیازها

ابتدا Termux را باز کنید و این دستورات را اجرا کنید:

```bash
pkg update && pkg upgrade -y
pkg install python openssl git -y
termux-setup-storage
```

وقتی درخواست دسترسی به حافظه آمد:
روی Allow بزنید

ساخت فایل IPها
فایل IP بسازید:
```Bash
nano ips.txt
```
سپس IPها را داخلش وارد کنید:
```Plain text
1.1.1.1
8.8.8.8
104.16.0.1
```
ذخیره فایل:
CTRL + X
سپس Y
سپس Enter

ساخت اسکریپت
فایل اسکریپت را بسازید:
```Bash
nano tester_pro.py
```
سپس کل کد زیر را داخلش Paste کنید:
```Python
import socket
import time
import ssl
from concurrent.futures import ThreadPoolExecutor

INPUT_FILE = "ips.txt"
OUTPUT_FILE = "good_ips.txt"

LIMIT = 500
TOP_LIMIT = 50
PORT = 443
TIMEOUT = 2

# دامنه SNI
SNI_HOST = "www.google.com"

results = []


def tcp_test(ip):
    try:
        s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        s.settimeout(TIMEOUT)

        start = time.time()
        s.connect((ip, PORT))
        tcp_latency = (time.time() - start) * 1000

        return s, tcp_latency

    except:
        return None, None


def tls_test(sock, ip):
    try:
        context = ssl.create_default_context()
        context.check_hostname = False
        context.verify_mode = ssl.CERT_NONE

        start = time.time()

        tls_sock = context.wrap_socket(
            sock,
            server_hostname=SNI_HOST
        )

        tls_latency = (time.time() - start) * 1000

        tls_sock.settimeout(1)
        tls_sock.send(b"HEAD / HTTP/1.0\r\n\r\n")

        return True, tls_latency

    except:
        return False, None


def test_ip(ip):
    sock, tcp_lat = tcp_test(ip)

    if not sock:
        return None

    ok, tls_lat = tls_test(sock, ip)
    sock.close()

    if not ok:
        return None

    score = tcp_lat + tls_lat
    return (ip, score)


with open(INPUT_FILE, "r") as f:
    ips = [x.strip() for x in f if x.strip()]

# فقط 200 IP اول
ips = ips[:LIMIT]

print(f"Testing {len(ips)} IPs...\n")

with ThreadPoolExecutor(max_workers=50) as ex:
    for i, res in enumerate(ex.map(test_ip, ips)):

        if res:
            ip, score = res
            results.append((ip, score))
            print(f"[OK] {ip} -> score {score:.1f}")

        else:
            print(f"[FAIL] {ips[i]}")

# مرتب‌سازی بر اساس سرعت
results.sort(key=lambda x: x[1])

best = results[:TOP_LIMIT]

# ذخیره خروجی
with open(OUTPUT_FILE, "w") as f:
    for ip, score in best:
        f.write(ip + "\n")

print("\n--- DONE ---")
print(f"Valid IPs: {len(results)}")
print(f"Saved top {len(best)} to {OUTPUT_FILE}")
```

ذخیره:
CTRL + X
سپس Y
سپس Enter

اجرای اسکریپت
```Bash
python tester_pro.py
```
فایل خروجی
بعد از پایان تست:
```Plain text
good_ips.txt
```
ساخته می‌شود.
این فایل شامل:
بهترین IPها
سریع‌ترین IPها
IPهایی که TLS و SNI موفق دارند
است.

تغییر تعداد IP تست
داخل اسکریپت:
```Python
LIMIT = 500
```
مثلاً:
100
500
1000
تغییر تعداد خروجی
```Python
TOP_LIMIT = 50
```
تغییر SNI
داخل اسکریپت:
```Python
SNI_HOST = "www.google.com"
```
مثال:
```Python
SNI_HOST = "www.cloudflare.com"
```
یا:
```Python
SNI_HOST = "www.microsoft.com"
```
نکات مهم
Ping پایین همیشه به معنی usable بودن VPN نیست
بعضی IPها فقط TCP باز دارند ولی TLS رد می‌شود
CDN Routing پویاست
بهتر است هر چند وقت IPها دوباره تست شوند
فایل‌های پروژه
```Plain text
ips.txt
tester_pro.py
good_ips.txt
```
نتیجه
این اسکریپت:
خیلی بهتر از ping ساده عمل می‌کند
فقط IPهای usable را نگه می‌دارد
سرعت اتصال را بهتر می‌کند
برای CDN Fronting مناسب‌تر است


