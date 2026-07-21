# نظام مزامنة/نسخ احتياطي لحظي (Real-time Delta Sync)

سكريبت خفيف يراقب مجلد معيّن ويزامن أي تغيير يحصل فيه فوراً مع مجلد نسخ احتياطي، بدون
الاعتماد على برامج بواجهة رسومية (بديل عملي وخفيف عن Syncthing).

يدعم النسخ الاحتياطي إلى **مجلد محلي على نفس السيرفر**، أو إلى **سيرفر بعيد عبر SSH**.

## الفكرة

- **rsync**: ينقل "الفروقات فقط" بين المصدر والوجهة (مو نسخ الملفات كاملة كل مرة).
- **inotify-tools**: يراقب المجلد لحظياً ويكتشف أي حدث (إضافة/تعديل/حذف) فور حدوثه.
- السكريبت يربط الأداتين مع بعض: أي حدث يلتقطه inotify → يشغّل rsync تلقائياً.

## سياسة الحذف

النسخة الاحتياطية **أرشيف تراكمي** — لو انحذف ملف من المصدر، يبقى محفوظ في النسخة
الاحتياطية ولا ينحذف تلقائياً (لا يوجد `--delete` في أمر rsync). هذا مقصود.

## تجميع الأحداث (Debounce)

لو صار تعديل/نسخ لعدة ملفات دفعة وحدة، السكريبت ينتظر ثانيتين هدوء بعد آخر حدث قبل ما
يشغّل rsync، بدل ما يشغّله مع كل حدث لحاله (يوفر موارد ويمنع تكدس العمليات).

## 1) تثبيت الأدوات المطلوبة

على توزيعات Debian/Ubuntu، على **السيرفر المصدر**:
```bash
sudo apt update
sudo apt install -y rsync inotify-tools
```

تأكد من التثبيت:
```bash
which rsync inotifywait
```

> **ملاحظة مهمة للنسخ عبر SSH:** لازم `rsync` يكون مثبت على **الطرفين** (السيرفر المصدر
> والسيرفر الوجهة)، مو بس على المصدر.

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

# لو الوجهة سيرفر بعيد عبر SSH، فعّل هذا وعدّل القيم (شوف قسم "الإعداد للنسخ إلى سيرفر بعيد" بالأسفل)
REMOTE_MODE=false                   # true = الوجهة سيرفر بعيد عبر SSH / false = مجلد محلي
SSH_KEY="$HOME/.ssh/backup_key"     # مسار مفتاح SSH الخاص (مستخدم فقط لو REMOTE_MODE=true)
SSH_PORT=22                         # بورت SSH (مستخدم فقط لو REMOTE_MODE=true)
# ---------------------------------------------------------------------

# التأكد من وجود المصدر، وإنشاء الوجهة تلقائياً لو ناقصة (محلي فقط)
if [ ! -d "$SOURCE" ]; then
    echo "❌ خطأ: مجلد المصدر غير موجود: $SOURCE"
    exit 1
fi

if [ "$REMOTE_MODE" = false ]; then
    mkdir -p "$TARGET"
fi

# بناء أمر rsync حسب النوع (محلي أو عبر SSH)
if [ "$REMOTE_MODE" = true ]; then
    RSYNC_CMD=(rsync -av -e "ssh -p $SSH_PORT -i $SSH_KEY" "$SOURCE" "$TARGET")
else
    RSYNC_CMD=(rsync -av "$SOURCE" "$TARGET")
fi

echo "🚀 بدأنا المراقبة اللحظية لمجلد: $SOURCE"
echo "تعديل الملفات الحاصل هنا سينتقل فوراً إلى: $TARGET"
echo "------------------------------------------------"

# مزامنة أولية عند بدء التشغيل (لضمان التطابق من البداية)
"${RSYNC_CMD[@]}"

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
    "${RSYNC_CMD[@]}"
    echo "✅ تمت المزامنة بنجاح!"
    echo "------------------------------------------------"
done
```

عدّل هذه الأسطر في أعلى الملف حسب الحالة عندك:

```bash
SOURCE=/full/path/to/source/
TARGET=/full/path/to/backup/
REMOTE_MODE=false
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

## 4) الإعداد للنسخ إلى سيرفر بعيد (Remote via SSH)

لو تبي الوجهة تكون سيرفر ثاني بدل مجلد محلي، اتبع هذه الخطوات **قبل** تشغيل السكريبت أو
ربطه بـ systemd.

### أ) توليد مفتاح SSH مخصص بدون كلمة مرور

على **السيرفر المصدر** (اللي فيه السكريبت):
```bash
ssh-keygen -t ed25519 -f ~/.ssh/backup_key -N ""
```

لازم بدون passphrase (`-N ""`) لأن السكريبت يشتغل أوتوماتيك بدون تدخل بشري.

### ب) نسخ المفتاح العام إلى السيرفر الوجهة

```bash
ssh-copy-id -i ~/.ssh/backup_key.pub user@remote_host
```

### ج) اختبار الاتصال يدويًا أول مرة (مهم جدًا)

```bash
ssh -p 22 -i ~/.ssh/backup_key user@remote_host
```

هذه الخطوة ضرورية لتسجيل بصمة السيرفر (`known_hosts`) يدويًا مرة وحدة. لو ما سويتها،
أول اتصال يصير أوتوماتيك من systemd بيعلّق وينتظر تأكيد (`yes/no`) ما حد بيرد عليه.

### د) تعديل إعدادات السكريبت

في أعلى `my_sync.sh`:
```bash
TARGET="user@remote_host:/full/path/to/backup/"
REMOTE_MODE=true
SSH_KEY="$HOME/.ssh/backup_key"
SSH_PORT=22
```

### هـ) تأكد إن rsync مثبت على السيرفر الوجهة أيضًا

```bash
ssh -i ~/.ssh/backup_key user@remote_host "which rsync"
```

لو ما طلعت نتيجة، ثبته على الوجهة:
```bash
sudo apt update && sudo apt install -y rsync
```

بعد هذي الخطوات، شغّل السكريبت يدويًا للاختبار (خطوة 3 بالأعلى) وتأكد إن الملفات
توصل للسيرفر البعيد قبل ما تربطه بـ systemd.

## 5) التشغيل الدائم كخدمة (systemd)

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

> **ملاحظة لو الوضع Remote:** الخدمة تشتغل بمستخدم `root` افتراضيًا — تأكد إن مفتاح
> SSH (`~/.ssh/backup_key`) موجود ومقروء من نفس المستخدم اللي تشغّل الخدمة بصلاحيته
> (لو الخدمة تعمل بـ `root`، لازم المفتاح يكون بمسار `/root/.ssh/` أو تحدد المسار الكامل
> الصحيح في `SSH_KEY`).

## ملاحظات مهمة

- **الصلاحيات:** إذا الخدمة تعمل بصلاحية `root`، تأكد إن root فعلاً عنده صلاحية قراءة/كتابة
  على المجلدين (أو صلاحية الاتصال بمفتاح SSH الصحيح في وضع Remote).
- **مجلدات فرعية:** السكريبت يراقب المجلد المحدد وكل المجلدات الفرعية بداخله (`-r`). لو
  عندك عدد ملفات/مجلدات ضخم جداً، تأكد إن `fs.inotify.max_user_watches` كافي:
  `cat /proc/sys/fs/inotify/max_user_watches`.
- **الاختبار أول:** جرّب دائماً على مجلد تجريبي قبل ربطه بمجلد بيانات حقيقي، وبنفس الطريقة
  اختبر وضع Remote يدويًا قبل تفعيله كخدمة دائمة.
