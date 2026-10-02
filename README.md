# نمونه Unity پوش‌پنل

پروژه نمونه Unity برای اتصال کتابخانه پوش PushPanel در اندروید.

کتابخانه: `ir.push-panel:push-sdk:1.8.3` از MavenCentral

> این راهنما بر اساس API نیتیو SDK نوشته شده و در این محیط بیلد نشده است.

## ۱. اضافه کردن AAR

1. فایل `push-sdk-1.8.3.aar` را از MavenCentral دانلود کن.
2. بگذار در:
```
Assets/Plugins/Android/push-sdk-1.8.3.aar
```
3. در Inspector همان فایل، تیک Android را فعال کن.

## ۲. گریدل

`Project Settings > Player > Android > Publishing` تیک `Custom Main Gradle Template` را بزن.

در `Assets/Plugins/Android/mainTemplate.gradle` داخل `dependencies`:

```gradle
dependencies {
    implementation("ir.push-panel:push-sdk:1.8.3")
}
```

مخازن `google()` و `mavenCentral()` به‌صورت پیش‌فرض در قالب یونیتی هستند. اگر نبود، به `repositories` اضافه کن.

## ۳. منیفست و دسترسی‌ها

تیک `Custom Main Manifest` را بزن و در `Assets/Plugins/Android/AndroidManifest.xml`:

```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
```

## ۴. فراخوانی از C#

```csharp
using UnityEngine;

public class PushPanelInit : MonoBehaviour
{
    void Start()
    {
#if UNITY_ANDROID && !UNITY_EDITOR
        // دسترسی نوتیفیکیشن اندروید ۱۳+
        // Unity 2020.2+ :
        // UnityEngine.Android.Permission.RequestUserPermission("android.permission.POST_NOTIFICATIONS");

        using (var unityPlayer = new AndroidJavaClass("com.unity3d.player.UnityPlayer"))
        using (var activity = unityPlayer.GetStatic<AndroidJavaObject>("currentActivity"))
        using (var pushSdk = new AndroidJavaClass("ir.pushpanel.sdk.PushPanel"))
        {
            pushSdk.CallStatic("init", activity);
        }
#endif
    }
}
```

این اسکریپت را روی یک GameObject در صحنه اول بگذار.

> در نسخه 1.8.3 نقطه ورود به `PushPanel` تغییر نام داده (قبلاً `PushSdk`) و به `handleIntent` و کد جداگانه برای اسپلش نیازی نیست؛ فقط `init` کافی است.

## ۵. فایربیس (برای دریافت واقعی پوش — اجباری)

بدون این مرحله توکن FCM ساخته نمی‌شود و پوشی دریافت نمی‌کنی (بیلد موفق می‌شود ولی خبری از پوش نیست).

۱. فایل `google-services.json` را از کنسول فایربیس بگیر و با ابزار `Google Services` یونیتی یا دستی در `Assets/Plugins/Android` قرار بده؛ پکیج‌نیم یونیتی (`Project Settings > Player > Android > Package Name`) باید با `package_name` داخل json یکی باشد.

۲. وابستگی‌ها را با `Android Resolver` نهایی کن (`Assets > External Dependency Manager > Android Resolver > Resolve`) تا مطمئن شوی کتابخانه‌های Firebase/Messaging واقعاً در بیلد هستند و `google-services.json` پردازش می‌شود (معادل مرحله پلاگین `google-services` در پروژه‌های گریدلی).

۳. برای اطمینان بعد از نصب روی دیوایس، لاگ‌کت را با فیلتر `PushSDK` ببین — باید ثبت توکن را نشان بدهد. اگر ثبت توکن دیده نشد، برگرد به قدم ۱ و ۲.

## ۶. سرویس فایربیس اختصاصی (اختیاری — فقط اگر کتابخانه پوش دیگری هم داری)

FCM در هر اپ فقط به **یک** `FirebaseMessagingService` پیام تحویل می‌دهد. اگر کتابخانه دیگری هم سرویس خودش را دارد، باید سرویس داخلی SDK را حذف کنی و همه پیام‌ها را از سرویس واحد به هر کتابخانه فوروارد کنی — پیام‌هایی که مال پنل نیستند توسط SDK نادیده گرفته می‌شوند (مارکر `pushpanel=pushpanel`).

۱. در `Custom Main Manifest` سرویس داخلی SDK را حذف کن (`xmlns:tools` را هم به تگ `manifest` اضافه کن):

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools">
    <application ...>
        <!-- حذف سرویس داخلی SDK تا فقط سرویس خودمان پیام بگیرد -->
        <service
            android:name="ir.pushpanel.sdk.PushMessagingService"
            tools:node="remove" />
        <!-- سرویس خودمان -->
        <service
            android:name=".MyFirebaseService"
            android:exported="false">
            <intent-filter>
                <action android:name="com.google.firebase.MESSAGING_EVENT" />
            </intent-filter>
        </service>
    </application>
</manifest>
```

۲. سرویس واحد (جاوا/کاتلین، داخل همان `Assets/Plugins/Android`) همه پیام‌ها را فوروارد کند:

```java
PushPanel.forwardMessage(context, remoteMessage); // فقط پیام‌های پنل هندل می‌شود، بقیه نادیده گرفته می‌شود
PushPanel.forwardToken(context, token);
```

نکته: `init` از C# (بخش ۴) همچنان لازم است؛ فوروارد قبل از init نادیده گرفته می‌شود. اگر کتابخانه دیگری نداری، این بخش را رد کن — سرویس داخلی SDK به‌صورت پیش‌فرض کار می‌کند.

## ۷. بیلد

`File > Build Settings > Android > Build` — خطای Resolve با `Android Resolver` (`Assets > External Dependency Manager > Android Resolver > Resolve`) معمولاً با همین وابستگی حل می‌شود.
