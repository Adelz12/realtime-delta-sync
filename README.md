# نظام مزامنة/نسخ احتياطي لحظي (Real-time Delta Sync)

سكريبت خفيف يراقب مجلد معيّن ويزامن أي تغيير يحصل فيه فوراً مع مجلد نسخ احتياطي، بدون
الاعتماد على برامج بواجهة رسومية (بديل عملي وخفيف عن Syncthing).

## الفكرة

- **rsync**: ينقل "الفروقات فقط" بين المجلدين (مو نسخ الملفات كاملة كل مرة).
- **inotify-tools**: يراقب المجلد لحظياً ويكتشف أي حدث (إضافة/تعديل/حذف) فور حدوثه.
- السكريبت يربط الأداتين مع بعض: أي حدث يلتقطه inotify → يشغّل rsync تلقائياً.

## سياسة الحذف

النسخة الاحتياطية **أرشيف تراكمي** — لو انحذف ملف من المصدر، يبقى محفوظ في النسخة
الاحتياطية ولا ينحذف تلقائياً (لا يوجد `--delete` في أمر rsync). هذا مقصود.

## تجميع الأحداث (Debounce)

لو صار تعديل/نسخ لعدة ملفات دفعة وحدة، السكريبت ينتظر ثانيتين هدوء بعد آخر حدث قبل ما
يشغّل rsync، بدل ما يشغّله مع كل حدث لحاله (يوفر موارد ويمنع تكدس العمليات).

## 1) تثبيت الأدوات المطلوبة (على السيرفر)

على توزيعات Debian/Ubuntu:
```bash
sudo apt update
sudo apt install -y rsync inotify-tools
```

تأكد من التثبيت:
```bash
which rsync inotifywait
```

## 2) السكريبت الكامل (my_sync.sh)

انسخ الكود التالي وضعه في ملف باسم `my_sync.sh` على السيرفر:

```bash
#!/bin/bash

# =============================================================
# my_sync.sh - Real-time Delta Backup/Sync Script
# rsync + inotifywait, one-way (Source -> Backup)
# =============================================================

# ------------------ الإعدادات (عدّل هذا الجزء فقط) ------------------
SOURCE=/full/path/to/source/        # مجلد المصدر المطلوب مراقبته - لازم / بالنهاية
TARGET=/full/path/to/backup/        # مجلد النسخة الاحتياطية - لازم / بالنهاية
DEBOUNCE_SECONDS=2                  # مدة الانتظار بعد آخر حدث قبل تشغيل rsync
# ---------------------------------------------------------------------

# التأكد من وجود المصدر، وإنشاء الوجهة تلقائياً لو ناقصة
if [ ! -d "$SOURCE" ]; then
    echo "❌ خطأ: مجلد المصدر غير موجود: $SOURCE"
    exit 1
fi
mkdir -p "$TARGET"

echo "🚀 بدأنا المراقبة اللحظية لمجلد: $SOURCE"
echo "تعديل الملفات الحاصل هنا سينتقل فوراً إلى: $TARGET"
echo "------------------------------------------------"

# مزامنة أولية عند بدء التشغيل (لضمان التطابق من البداية)
rsync -av "$SOURCE" "$TARGET"

# -r : يراقب المجلدات الفرعية كلها أيضاً (مهم في بيئة سيرفر حقيقية)
# -m : وضع مستمر (monitor)
inotifywait -m -r -e modify,create,delete "$SOURCE" | while read -r path action file; do
    echo "🔄 تم رصد تغيير ($action) على الملف: $file. جاري تجميع الأحداث..."

    # تجميع الأحداث (Debounce): نبلع أي أحداث إضافية تجي خلال فترة قصيرة
    # عشان ما نشغل rsync عشرات المرات لو صار نسخ/تعديل جماعي للملفات
    while read -t "$DEBOUNCE_SECONDS" -r extra_path extra_action extra_file; do
        echo "   (تجميع) حدث إضافي: $extra_action -> $extra_file"
    done

    echo "🔁 جاري المزامنة..."
    rsync -av "$SOURCE" "$TARGET"
    echo "✅ تمت المزامنة بنجاح!"
    echo "------------------------------------------------"
done
```

عدّل هذين السطرين في أعلى الملف بالمسارات **الفعلية**
(يفضّل مسارات مطلقة كاملة، مو `~`):

```bash
SOURCE=/full/path/to/source/
TARGET=/full/path/to/backup/
```

⚠️ لاحظ الشرطة المائلة `/` في نهاية كل مسار — مهمة لـ rsync.

اجعل السكريبت قابل للتنفيذ:
```bash
chmod +x my_sync.sh
```

## 3) التشغيل اليدوي (للاختبار)

```bash
./my_sync.sh
```

سويّ تعديل تجريبي على أي ملف داخل مجلد المصدر وتأكد إن السكريبت يرصده وينفذ المزامنة
تلقائياً (يطبع رسائل توضح ذلك). أوقفه بـ `Ctrl+C`.

## 4) التشغيل الدائم كخدمة (systemd)

عشان السكريبت يشتغل بالخلفية دائماً، ويرجع يشتغل لوحده لو كرّش أو أعيد تشغيل السيرفر:

```bash
sudo cp my_sync.sh /usr/local/bin/my_sync.sh
sudo chmod +x /usr/local/bin/my_sync.sh
```

أنشئ ملف الخدمة مباشرة:
```bash
sudo tee /etc/systemd/system/my_sync.service > /dev/null << 'EOF'
[Unit]
Description=Real-time Delta Backup Sync (rsync + inotify)
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/my_sync.sh
Restart=always
RestartSec=5
User=root

[Install]
WantedBy=multi-user.target
EOF
```

ثم:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now my_sync.service
sudo systemctl status my_sync.service
```

متابعة السجلات مباشرة:
```bash
sudo journalctl -u my_sync.service -f
```

## ملاحظات مهمة

- **نسخ احتياطي على سيرفر بعيد (Remote):** لو الوجهة سيرفر ثاني، هذا السكريبت بشكله
  الحالي لا يكفي — يحتاج rsync عبر SSH مع مفاتيح SSH بدون كلمة مرور. إذا هذا هو السيناريو
  عندك، تواصل قبل التشغيل عشان نضيف الإعداد المناسب.
- **الصلاحيات:** إذا الخدمة تعمل بصلاحية `root`، تأكد إن root فعلاً عنده صلاحية قراءة/كتابة
  على المجلدين.
- **مجلدات فرعية:** السكريبت يراقب المجلد المحدد وكل المجلدات الفرعية بداخله (`-r`). لو
  عندك عدد ملفات/مجلدات ضخم جداً، تأكد إن `fs.inotify.max_user_watches` كافي:
  `cat /proc/sys/fs/inotify/max_user_watches`.
- **الاختبار أول:** جرّب دائماً على مجلد تجريبي قبل ربطه بمجلد بيانات حقيقي.
