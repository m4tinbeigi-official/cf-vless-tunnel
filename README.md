<!-- DevSponsors Badges -->
<p align="center">
  <a href="https://devsponsors.github.io"><img src="https://img.shields.io/badge/DevSponsors-Verified_OSS-6366f1?style=for-the-badge&logo=github" alt="DevSponsors Verified"></a>
  <a href="https://devsponsors.github.io"><img src="https://img.shields.io/badge/Sponsor-DevSponsors_Hub-emerald?style=for-the-badge&logo=github-sponsors" alt="DevSponsors Sponsor"></a>
  <a href="https://devsponsors.github.io/mediakit.html"><img src="https://img.shields.io/badge/Infrastructure-DevSponsors_Cloud-ec4899?style=for-the-badge&logo=server" alt="DevSponsors Cloud"></a>
</p>

# README
## معماری وی پی ان سه لایه با سرور ایران و Cloudflare Worker و سرور خارج

این مخزن شامل یک راهنمای کاملا عملی برای پیاده سازی وی پی ان شخصی با معماری چند مرحله ای است که برای شرایط فیلترینگ شدید طراحی شده است

ساختار اتصال به این شکل است

Client
Server Iran
Cloudflare Worker
Server Foreign
Internet

در این معماری
آی پی سرور خارج مخفی میماند
ترافیک از Cloudflare عبور میکند
پایداری اتصال داخل ایران بالاتر است
الگوی ترافیک شبیه سرویس های مجاز دیده میشود


## هدف پروژه

۱ استفاده از سرور ایران به عنوان ورودی امن
۲ تونل کردن ترافیک به خارج کشور
۳ استتار کامل با Cloudflare
۴ جلوگیری از شناسایی سرور خارج
۵ افزایش مقاومت در برابر DPI


## پیش نیازها

۱ یک سرور ایران با آی پی سالم
۲ یک سرور خارج
۳ یک دامنه متصل به Cloudflare
۴ نصب XUI یا 3XUI روی هر دو سرور
۵ اکانت Cloudflare با دسترسی Worker


## نمای کلی مسیر ترافیک

Client
VLESS روی سرور ایران
WebSocket
Cloudflare Worker
WebSocket
سرور خارج


## مرحله اول پیکربندی سرور خارج

روی سرور خارج یک Inbound جدید بساز

تنظیمات

Protocol
VLESS

Port
10000

Transport
WebSocket

Path
/ws

TLS
خاموش

UUID
بساز و ذخیره کن

مثال

IP
2.2.2.2

Port
10000

Path
/ws


## مرحله دوم تنظیم دامنه در Cloudflare

یک ساب دامنه بساز

مثال
edge.example.com

DNS Record

Type
A

Name
edge

IP
2.2.2.2

Proxy
روشن

SSL Mode
Full


## مرحله سوم ساخت Cloudflare Worker

در داشبورد Cloudflare وارد بخش Workers شو و Create Worker را بزن

کد Worker

```js
export default {
  fetch(request) {
    if (request.headers.get("Upgrade") !== "websocket") {
      return new Response("OK")
    }

    const pair = new WebSocketPair()
    const client = pair[0]
    const server = pair[1]

    server.accept()

    const upstream = new WebSocket("wss://edge.example.com/ws")

    upstream.onmessage = e => server.send(e.data)
    server.onmessage = e => upstream.send(e.data)

    upstream.onclose = () => server.close()
    server.onclose = () => upstream.close()

    return new Response(null, {
      status: 101,
      webSocket: client
    })
  }
}
```

Worker را Publish کن


## مرحله چهارم اتصال دامنه به Worker

در بخش Worker
Custom Domain اضافه کن

مثال
tunnel.example.com

از این لحظه این دامنه مستقیما به Worker متصل است


## مرحله پنجم پیکربندی سرور ایران

روی سرور ایران یک Inbound بساز

Protocol
VLESS

Port
443

Transport
WebSocket

Path
/ir

TLS
فعال

UUID
مخصوص سرور ایران


## مرحله ششم ساخت Outbound در سرور ایران

یک Outbound جدید به سمت Worker بساز

Protocol
VLESS

Address
tunnel.example.com

Port
443

UUID
UUID سرور خارج

Transport
WebSocket

Path
/cloud

TLS
فعال

SNI
tunnel.example.com


## مرحله هفتم تنظیم مسیریابی در سرور ایران

تمام ترافیک Inbound سرور ایران
به Outbound Cloudflare هدایت شود

در XUI
Inbound Iran
Connected to
Outbound Cloudflare


## مرحله هشتم تنظیم کلاینت

در کلاینت V2Ray یا v2rayNG

Protocol
VLESS

Address
IP یا دامنه سرور ایران

Port
443

UUID
UUID سرور ایران

Network
WebSocket

Path
/ir

TLS
فعال

SNI
دامنه سرور ایران


## نتیجه نهایی

مسیر واقعی ترافیک

Client
Server Iran
Cloudflare Worker
Server Foreign

آی پی سرور خارج مخفی است
Path واقعی قابل شناسایی نیست
الگوی ترافیک HTTPS عادی است


## نکات مهم

۱ برای مصرف شخصی بسیار مناسب است
۲ برای فروش انبوه ریسک مسدودی دارد
۳ Worker رایگان محدودیت دارد
۴ میتوان چند Worker چرخشی استفاده کرد
۵ میتوان چند سرور خارج پشت یک Worker قرار داد


## ارتقاهای حرفه ای

۱ اضافه کردن احراز هویت در Worker
۲ محدود سازی آی پی ورودی
۳ توزیع بار بین چند سرور خارج
۴ طراحی مخصوص موبایل
۵ استتار دامنه مشابه سرویس های هوش مصنوعی
