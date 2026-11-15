# פרסום ושדרוג הפלאג-אין – מה למדנו ואיך לא לחזור על השגיאות

מסמך זה מתעד את המעבר של Legal-Massadit-All בבדיקת ה-Directory של Anthropic (נובמבר 2026, גרסה 1.0.4).
**לקרוא לפני כל שדרוג גרסה או שינוי ב-`plugin.json`, ב-`hooks/hooks.json` או בדרך שבה השרת מופעל.**

## 1. מה השתנה בגרסה 1.0.4

| נושא | עד 1.0.3 | מ-1.0.4 |
|---|---|---|
| שרת ה-MCP (Legal-Connector) | חבילה בינארית `servers/Legal-Connector.mcpb` | מוגדר ישירות ב-`plugin.json`, רץ מ-`mcp/legal-mcp.ps1` |
| הפעלת PowerShell | `powershell -NoProfile -ExecutionPolicy Bypass -File …` | `powershell -NoProfile -File ${CLAUDE_PLUGIN_ROOT}/…` |
| עקיפת מדיניות ההרצה בשרת | דגל בשורת הפקודה | `"env": { "PSExecutionPolicyPreference": "Bypass" }` |
| `allowed-tools` של `/legal-setup` | כולל `Bash`, `PowerShell`, `device_bash` | רק `AskUserQuestion`, `ToolSearch` וכלי ההגדרה של הפלאג-אין |

שמות הכלים (`mcp__plugin_legal-massadit-all_Legal-Connector__*`) **לא השתנו**, ולכן הסקיל, הפקודה וה-hooks ממשיכים לעבוד כמו קודם.

## 2. הכללים – חובה בכל שדרוג

1. **אין ערכים אחרי דגלים בשורת ההפעלה.** ב-`args` של hook או של שרת MCP מותרים רק דגלים בודדים (`-NoProfile`, `-File`) ונתיב שמתחיל ב-`${CLAUDE_PLUGIN_ROOT}/`.
   הבודק קורא כל ארגומנט שאינו דגל כנתיב. `-ExecutionPolicy Bypass` → הבודק רואה את `Bypass` כ"נתיב מחושב" → **חסימה** (`COMMAND_PATH_COMPUTED`).
2. **עקיפת מדיניות PowerShell עוברת דרך `env`, לא דרך הדגל.** לשרת MCP יש שדה `env`; שם מגדירים `PSExecutionPolicyPreference=Bypass`.
   ל-hooks אין `env` – הם נשענים על מדיניות המחשב (ראה סעיף 4).
3. **לא להשתמש ב-`"shell": "powershell"` ב-hooks.** הצורה הזו עוברת את הבודק, אבל בבדיקה ב-Claude Code היא **לא רצה בכלל**.
4. **`-ExecutionPolicy:Bypass` (בטוקן אחד) לא קיים** ב-Windows PowerShell 5.1 – הפקודה נכשלת.
5. **לא לחזור לחבילת `.mcpb` בתוך הפלאג-אין.** הבודק לא פותח אותה והיא מעוכבת לבדיקה. תוסף Claude Desktop עצמאי, אם נדרש – כקובץ ב-GitHub Releases, לא בתוך העץ.
6. **לא להרחיב את `allowed-tools`** ל-`Bash`/`PowerShell` כלליים.
7. **לא להדפיס שורות פקודה עם משתנים** בתוך טקסט שה-hooks או השרת מחזירים (הבודק סורק גם את תוכן קובצי ה-`.ps1`). להפנות לקובץ "ביחס לתיקיית הפלאג-אין" במקום לבנות פקודה.
8. **לעדכן גרסה בשני מקומות:** `version` ב-`.claude-plugin/plugin.json` ו-`$serverVersion` ב-`mcp/legal-mcp.ps1`.

## 3. סדר עבודה לשדרוג

1. לעבוד בענף נפרד, **בשם בלי `/`** (טופס הפורטל מפצל את השם ב-`/` לענף ותיקייה). לא לגעת ב-`main` עד שהכול עובר.
2. בדיקה מקומית: `claude plugin validate . --profile directory`.
   ⚠️ **מעבר מקומי אינו מבטיח מעבר בפורטל** – הפורטל מריץ חוקים נוספים.
3. בדיקה אמיתית ב-Claude Code, כולל תחת מדיניות מגבילה:
   ```powershell
   $env:PSExecutionPolicyPreference = 'Restricted'
   claude -p "Call the tool mcp__plugin_legal-massadit-all_Legal-Connector__legal_setup_status and reply with only the first line of its output." --plugin-dir . --allowedTools "mcp__plugin_legal-massadit-all_Legal-Connector__legal_setup_status" --max-turns 3
   Remove-Item Env:PSExecutionPolicyPreference
   ```
   צפוי: תשובה מהשרת (למשל `SETUP NOT COMPLETE.` במחשב בלי הגדרה).
4. בדיקת ה-guard:
   ```powershell
   '{"tool_input":{"command":"notepad %APPDATA%\\MasTmd\\config.json"}}' | powershell -NoProfile -File hooks/guard.ps1
   ```
   צפוי: הודעת חסימה ו-exit 2.
5. הרצת הבודק בפורטל על הענף (Repository = כתובת הריפו, Plugin path ריק, Branch = שם הענף).
6. רק אחרי 0 חסימות – PR ומיזוג ל-`main`.

## 4. מגבלה ידועה

ה-hooks (`check-setup` ו-`guard`) רצים לפי מדיניות ה-PowerShell של המחשב. ברוב המחשבים (`RemoteSigned`) הם רצים.
במחשב עם `Restricted` הם לא ירוצו – **בלי לחסום שום דבר**: הכלים עובדים, ושרת ה-MCP עצמו מבקש הגדרה ראשונית.
אם זה קורה, המשתמש יכול להריץ פעם אחת: `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned`.

## 5. עיכובים (policy holds) שנשארים – וזה תקין

אלה **לא חוסמים**. הם מחכים לבודק אנושי, כי הארגון אינו Verified:
- hooks מסוג command ושרת MCP מקומי (הפלאג-אין חייב לרוץ מקומית – הוא פותח מסד נתונים על המחשב).
- קובצי `.ps1` (סוג קובץ שהבודק לא קורא).
- `allowed-tools` מעבר ל-Read/Grep/Glob/LS.
- סיסמת המסד (`ConnStr`) – לציין ב-listing שהיא סיסמת Jet מקומית ואינה יוצאת מהמחשב.
- המשתנה `PSExecutionPolicyPreference` ב-`env` של השרת – לציין ב-listing שזה ערך קבוע שמאפשר ל-PowerShell להריץ את סקריפט השרת.

בעתיד ה-Directory עשוי לקבל hooks ושרתים מקומיים רק מארגונים Verified.

## 6. איך מצאנו את הבעיה (לפעם הבאה שמשהו לא ברור)

- הדוח של הפורטל **לא ציין קובץ** לחסימה, והקוד `COMMAND_PATH_COMPUTED` לא מופיע בתיעוד. החוק המקביל בתיעוד: "Command path can't be followed" (https://claude.com/docs/plugins/pre-submission-checklist).
- שני ניחושים ראשונים היו שגויים (חבילת ה-`.mcpb`, ואחר כך טקסט שמודפס ב-`check-setup.ps1`).
- **מה שעבד: פיצול (bisect) בענפי בדיקה** – ענף בלי hooks, ענף בלי שרת, ענף בלי שניהם, ואז ענף לכל צורת הפעלה. כל הרצה בפורטל צמצמה את האפשרויות, עד שנמצא `Bypass`.
- הכלל: **קודם תיעוד, אחר כך פיצול בענפי בדיקה, ורק אז תיקון.** לא לדחוף ניחושים.
