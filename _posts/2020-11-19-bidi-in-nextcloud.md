---
title:  "Fix bidirectional (bidi) text problem in Nextcloud (Farsi)"
layout: post
last_modified_at: 2020-11-19T23:20:12-05:00
author: Ahmad Haghighi
summary: "Handle RTL and text aligment in Nextcloud (e.g. Talk, Deck, Calendar) "
local_thumbnail: nextcloud
categories: tutorial
tags:
  - nextcloud
  - floss
  - bidi
  - rtl
---

امنیت و محرمانگی اطلاعات و این‌که داده‌های ما کجا، چگونه و توسط چه فرد یا نهادی ذخیره می‌شود برای بسیاری از افراد و شرکت‌ها/سازمان‌ها مهم و به عبارتی دغدغه است، و به همین دلیل (+ دلایل دیگری نظیر تحریم‌های خارجی و یا وضعیت قوانین داخلی) شرکت‌ها و افراد بهترین گزینه را این می یابند که خود داده‌های خود را مدیریت و نگهداری کنند و واقعا و کاملا مالک داده‌های خود باشند.    

بدون شک نکست‌کلود (https://nextcloud.com) یکی از بهترین راهکار‌های ابری موجود است که نه‌تنها امکانات و app‌هایی فراوان و رایگان دارد، بلکه از همه مهم‌تر **آزاد** (نرم‌افزار آزاد) است و تیم فعال، پویا و در حال رشدی دارد.    

از آنجایی که این مطلب در مورد نکست‌کلود و معرفی آن نیست به همین مقدار بسنده می‌کنم و سایر اطلاعات را می‌توانید از وب‌سایت رسمی نکست‌کلود به آدس [nextcloud.com](https://nextcloud.com/) دریافت کنید.    

> برای راه‌اندازی سرور نکست‌کلود شخصی نیاز به منابع زیاد و سنگینی ندارید و لذا اینکونه نیست که باید حتما دارای کسب و کار و شرکت باشید تا به فکر راه‌اندازی سرور ابری شخصی خود بیفتید، چرا که اگر به امنیت و مالکیت و محرمانگی داده‌های خود اهمیت می‌دهید، می‌توانید یک [سرور ارزان قیمت با منابع کم](https://docs.nextcloud.com/server/latest/admin_manual/installation/system_requirements.html) تهیه کنید و برای خود و خانواده بدون مشکل از آن استفاده کنید.    

## رفع مشکل متن‌های دوسویه (Bidirectional) در نکست‌کلود     
برای این منظور ابتدا وارد قسمت **Apps** شوید و سپس [Custom CSS](https://apps.nextcloud.com/apps/theming_customcss) را جستجو و سپس نصب و فعال کنید.    

![nextcloud-bidi-custom-css-app](/assets/images/nextcloud-bidi-custom-css-app.png) 

سپس وارد قسمت **Settings**  شوید و از بخش **Administration** وارد قسمت تنظیمات مربوط به پوسته یا همان **Theming** شوید:    

![nextcloud-bidi-custom-css-settings](/assets/images/nextcloud-bidi-custom-css-settings.png)    

حال اسکریپت **CSS** مورد نظر خود را که مایلید روی تمامی سرور اعمال شود در این قسمت وارد کرده و کلید **Save** را بزنید. پس از ذخیره تغییرات اعمال می‌شود.     

### اسکریپت پیشنهادی برای استفاده به عنوان Custom CSS :     
```css
p,h1,div,span,a,ul,h2,h3,h4,li,input {
    direction: ltr;
    unicode-bidi: plaintext;
    text-align: initial;
}
```

### تصاویر ضمیمه    
![nextcloud-bidi-deck](/assets/images/nextcloud-bidi-deck.png)    
![nextcloud-bidi-talk](/assets/images/nextcloud-bidi-talk.png)   


