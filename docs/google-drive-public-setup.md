# מדריך: פרסום פיצ'ר הגיבוי של Google Drive למצב ציבורי

> **מצב נוכחי**: הקוד מוכן מלא ופועל! נדרש רק שינוי הגדרות ב-Google Cloud Console.

---

## 📋 סיכום מהיר

הפיצ'ר של גיבוי ושחזור דרך Google Drive **מיושם מלא** וכולל:

- ✅ אוטנטיקציה OAuth2 (Serverless)
- ✅ גיבוי ידני ואוטומטי
- ✅ שחזור מגיבויים קיימים
- ✅ זיהוי קונפליקטים (lastModified)
- ✅ Hydration/Dehydration של תמונות מ-IndexedDB

**מה חסר?** שינוי סטטוס הפרסום ב-Google Cloud Console מ-"Testing" ל-"In production".

## 🔴 תיקון 2026-11-02 — זו אינה לחיצה אחת

המדריך למטה נכתב כאילו די ללחוץ **PUBLISH APP** ולאשר. **זה אינו מדויק.**
נמדד בקונסולה: הכפתור **חסום** עד שממלאים שני שדות, והסיבה אינה מופיעה
בטקסט הגלוי.

### מה גוגל דורשת כדי לעבור ל-production

| שדה (בדף Branding) | נדרש? |
|---|---|
| App name | ✅ |
| User support email | ✅ |
| **Application home page** | ✅ **זה מה שחסם** |
| **Application privacy policy link** | ✅ **זה מה שחסם** |
| Application Terms of Service link | ❌ אינו נדרש לפרסום |
| Authorised domains | ✅ הדומיין של שתי הכתובות חייב להיות רשום כאן |

‏🛑 **דף Branding אינו מסמן שום שדה כחסר.** השדות הריקים נראים רגילים,
בלי סימון שגיאה. ‏⇒ קל להסיק שהכול מוכן ושהבעיה היא בכפתור.

### 🔑 איך למצוא את הסיבה המדויקת

הכפתור נראה לחיץ אבל נושא `aria-disabled="true"` (תבנית Material של כפתור
מושבת שמקבל קליק רק כדי להציג טולטיפ). **ההסבר יושב ב-`aria-describedby`:**

```js
const b = [...document.querySelectorAll('button')]
  .find(x => /publish app/i.test(x.innerText));
document.getElementById(b.getAttribute('aria-describedby')).innerText
```

מחזיר:

> Valid app name, support email, homepage URL and privacy policy URL are
> required for switching the app to external production mode. You must enter
> the missing information to proceed. Please visit the Branding page to finish
> configuring your app.

### ⚠️ הקונסולה זזה — הנתיב בשלב 2 מיושן

‏`APIs & Services → OAuth consent screen` הוחלף ב-**Google Auth Platform**,
עם לשוניות נפרדות: **Audience** (סטטוס הפרסום, ‏user cap, ‏test users) ו-
**Branding** (שם, מיילים, הכתובות, Authorised domains).

### 🔴 ולמה בכל זאת שווה לפרסם — מגבלת 7 הימים

מתועד אצל גוגל ב-`developers.google.com/identity/protocols/oauth2`:

> A Google Cloud Platform project with an OAuth consent screen configured for
> an external user type and a publishing status of **Testing** is issued a
> refresh token expiring in **7 days**, unless the only OAuth scopes requested
> are a subset of name, email address, and user profile.

‏⇒ כל עוד האפליקציה ב-Testing וסקופ Drive מבוקש, **ה-refresh token פג כל
7 ימים** וצריך לאשר מחדש. זו התנהגות מתוכננת של גוגל, לא תקלה — וזו הסיבה
המרכזית לעבור ל-production.

‏🛑 **אבל "לא פג מעצמו" אינו "נצחי".** אותו עמוד מונה עוד סיבות, ושתיים
רלוונטיות לאפליקציה כזאת: **שינוי סיסמת גוגל** (חל כשמבקשים סקופים של
Gmail), ו-**מעל 100 refresh tokens לחשבון לכל client** — *"creating a new
refresh token automatically invalidates the oldest without warning"*.

### 🔴 ועוד דבר שהמקטע "האם צריך Verification" לא אומר

אפליקציה לא-מאומתת **עובדת** — כולל סקופים sensitive ו-restricted. מה שכן
מתלווה: מסך האזהרה, ותקרת **100 משתמשים חדשים לכל חיי הפרויקט**, ש‏**אינה
ניתנת לאיפוס או שינוי** (לשון הקונסולה: *"cannot be reset or changed"*).

‏⚠️ **ובמסך ההסכמה עצמו גוגל מוסיפה אזהרה שאינה בתיעוד הפומבי:**
*"אחרת, יכול להיות שיתבטלו חלק מהרשאות הגישה של האפליקציה לנתונים שלך"*.
‏⇒ "עובד בלי verification" נכון להיום ואינו הבטחה לעתיד. אם האפליקציה
מתחילה לקבל `403` על **שירות בודד** ולא על כולם — זה החשד הראשון, ולא
טוקן פג.

‏✅ **והפעולה הפיכה** — בדף Audience יש `Back to testing`.

---

---

## 🔧 שלבים לפרסום

### שלב 1: כניסה ל-Google Cloud Console

1. היכנס ל-[Google Cloud Console](https://console.cloud.google.com/)
2. בחר את הפרויקט שבו יצרת את ה-OAuth Client ID (בדרך כלל הפרויקט שבו הפעלת את ה-Google Drive API)

### שלב 2: מסך OAuth Consent Screen

1. בתפריט הצד, עבור אל:

   ```
   APIs & Services → OAuth consent screen
   ```

2. תראה את המסך הנוכחי עם הסטטוס:
   - **Publishing status: Testing** ⚠️
   - **User type: External**

### שלב 3: שינוי סטטוס הפרסום

‏🛑 **לפני הלחיצה — לוודא שדף Branding מלא** (ראו התיקון מ-2026-11-02 למעלה).
בלי `Application home page` ו-`Application privacy policy link` הכפתור חסום.

1. לחץ על כפתור **"PUBLISH APP"** (או "Publish to Production")
2. תופיע אזהרה:

   ```
   Publishing your app will make it available to any user with a Google Account.

   Your app will still show an unverified app screen until you complete
   the verification process.
   ```

3. **אשר** את הפרסום

### שלב 4: הבנת המצב "Unverified"

לאחר הפרסום, המשתמשים יראו מסך אזהרה:

```
Google hasn't verified this app
This app hasn't been verified by Google yet. Only proceed if you
know and trust the developer.
```

**זה נורמלי לחלוטין!** המשתמשים יכולים:

1. ללחוץ על "Advanced"
2. לבחור "Go to [שם האפליקציה] (unsafe)"
3. להמשיך להתחברות

---

## 🔒 האם צריך Verification?

### מתי **לא** צריך:

- ✅ אפליקציה פרטית/משפחתית (המקרה שלנו!)
- ✅ מספר מצומצם של משתמשים מוכרים
- ✅ אין שימוש מסחרי

### מתי **כן** צריך:

- ❌ אפליקציה ציבורית עם אלפי משתמשים
- ❌ שימוש מסחרי
- ❌ רוצים להסיר לחלוטין את מסך האזהרה

---

## 📝 תהליך Verification (אופציונלי)

אם בעתיד תרצה להסיר את מסך האזהרה:

1. **הכנת מסמכים:**
   - Privacy Policy (מדיניות פרטיות)
   - Terms of Service (תנאי שימוש)
   - הסבר על השימוש ב-Google Drive API

2. **הגשת בקשה:**
   - ב-OAuth consent screen, לחץ "Start Verification"
   - מלא את כל הפרטים
   - צרף מסמכים
   - זמן אישור: 4-6 שבועות

---

## ✅ מה קורה אחרי הפרסום?

1. **משתמשים חדשים:**
   - יכולים להתחבר לאפליקציה ללא הגבלה
   - יראו מסך "Unverified" - אבל יכולים לעבור אותו

2. **גיבויים:**
   - פועלים מלא: ידני + אוטומטי
   - שמירה ב-Google Drive בתיקייה `AppData` (מוסתר)
   - שחזור עובד מכל מכשיר

3. **סנכרון:**
   - זיהוי קונפליקטים אוטומטי
   - בחירה בין גרסה מקומית לגרסה בענן

---

## 🧪 בדיקה לאחר הפרסום

### בדיקה 1: משתמש חדש

1. פתח את האפליקציה במצב Incognito
2. נסה להתחבר ל-Google Drive
3. ודא שהמסך "Unverified" מופיע אבל ניתן לעבור אותו

### בדיקה 2: גיבוי ושחזור

1. בצע גיבוי מכלל אחד
2. שחזר באותו כלי או בכלי אחר
3. ודא שהכל עובד

### בדיקה 3: קונפליקטים

1. פתח את האפליקציה בשני טאבים שונים
2. ערוך נתונים בכל טאב
3. בצע גיבוי מטאב אחד
4. בטאב השני, בצע גיבוי - ודא שמופיע מסך קונפליקט

---

## 🎯 סיכום

**מה יש:**

- ✅ קוד מוכן מלא
- ✅ OAuth2 פועל
- ✅ כל הפיצ'רים מיושמים

**מה חסר:**

- ⏸️ לחיצה על "PUBLISH APP" ב-Google Cloud Console

**זמן ביצוע:** ~2 דקות

---

## 📚 משאבים נוספים

- [OAuth Consent Screen Documentation](https://support.google.com/cloud/answer/10311615)
- [Google Drive API Scopes](https://developers.google.com/drive/api/guides/api-specific-auth)
- [Verification Process](https://support.google.com/cloud/answer/9110914)

---

## 🆘 פתרון בעיות נפוצות

### בעיה: "Access blocked: Authorization Error"

**פתרון:** ודא שה-scope `https://www.googleapis.com/auth/drive.appdata` מאושר ב-OAuth consent screen.

### בעיה: "Redirect URI mismatch" (שגיאה 400)

**סיבה:** הכתובת הנוכחית (למשל `https://dev.daily-schedule.pages.dev`) אינה מאושרת במסוף של גוגל.
**פתרון:**

1. כנס ל-Console -> Credentials -> OAuth 2.0 Client IDs.
2. הוסף תחת **Authorized JavaScript origins** את הכתובת המלאה:
   - `https://daily-schedule.pages.dev` (פרודקשן)
   - `https://dev.daily-schedule.pages.dev` (סביבת פיתוח - שים לב ל-dev בהתחלה!)
   - `http://localhost:5173` (לפיתוח מקומי)
3. הוסף תחת **Authorized redirect URIs** את אותן כתובות (ליתר ביטחון, למרות שב-Popup ה-Origin הוא הקובע).
4. שמור והמתן מספר דקות.

### בעיה: משתמשים לא יכולים להתחבר

**פתרון:** ודא שעברת ל-"In production" ולא "Testing" (במצב Testing רק 100 משתמשים test מאושרים יכולים להיכנס).

---

**נכתב:** 2026-01-20  
**גרסה:** 1.0
