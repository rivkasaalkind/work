https://github.com/mateocallec/exceljs-hardened

exceljs-hardened
Warning

This is an unofficial, unaffiliated fork of exceljs. It exists to maintain the security of a widely-used library whose upstream appears unmaintained: no response channel currently works (Discord is gone, GitHub private vulnerability reporting is disabled on the upstream repo), and several real vulnerabilities have gone unpatched as a result.

This fork is not endorsed by, and has no relationship with, the original exceljs maintainers. It is not a general-purpose continuation of the project — it exists specifically to carry security fixes for known, reported issues that upstream cannot currently receive.

Why this exists
exceljs is depended on by a large number of Node.js applications for reading/writing .xlsx/.csv files. Several vulnerabilities in the upstream code have real, demonstrated impact (prototype pollution, arbitrary local file read, unbounded decompression) and no way to be responsibly disclosed to, or fixed by, the original project. Rather than let users choose between "stay vulnerable" and "stop using the library," this fork patches those specific issues and republishes under a different package name so they don't collide with the official one on npm.

What's patched
ID	Vulnerability	Fixed in	Status
VULN-01	Prototype pollution in _.deepMerge() (lib/utils/under-dash.js)	5.0.0	✅ Patched
VULN-02	Arbitrary local file read via addImage({filename}) (lib/doc/workbook.js, lib/xlsx/xlsx.js)	5.0.0	✅ Patched
VULN-03	CSV / formula injection in CSV.write() (lib/csv/csv.js)	5.0.0	✅ Patched
VULN-04	Decompression bomb / memory exhaustion in Workbook#load() (lib/xlsx/xlsx.js)	5.0.0	✅ Patched
All four issues identified in the original audit are now patched and verified against live proof-of-concept exploits (see ../../poc/exceljs — the same PoC tools that demonstrated each vulnerability were re-run against a demo app wired to this fork to confirm the fixes hold).

See SECURITY.md for how to report new issues, and each patch's inline [exceljs-hardened] comments in the source for the technical rationale.

VULN-01 — Prototype pollution
_.deepMerge() now refuses to assign into __proto__, constructor, or prototype keys, closing the path from a JSON-parsed cell.note object to the global Object.prototype.

VULN-02 — Arbitrary local file read
addImage({filename}) and the internal media writer reject any raw path containing a .. segment (assertSafeMediaPath) — but that alone does not fix the realistic exploit chain, and testing against a live target confirmed it: the typical vulnerable pattern is

const resolvedPath = path.join(baseDir, userInput);   // e.g. userInput = '../../../.env'
workbook.addImage({filename: resolvedPath});
path.join() already resolves and strips .. segments before addImage() ever sees the string (path.join('/app/assets/logos', '../../../.env') → '/app/.env', with no .. left in it) — so a check performed after the join has nothing left to catch.

The actual fix is ExcelJS.utils.safeJoin(baseDir, userInput), which resolves the path and checks containment as a single operation instead of two separate steps:

const ExcelJS = require('exceljs-hardened');

// throws: Refusing to resolve "../../../.env" against base directory
// "/app/assets/logos": it resolves to "/app/.env", which is outside the
// base directory.
const safePath = ExcelJS.utils.safeJoin('/app/assets/logos', '../../../.env');
If your application builds an addImage({filename}) path from user input, you must use safeJoin() instead of path.join() — this is not optional defense in depth, it's the fix. See the demo app's ../../poc/exceljs/webapp-hardened/server.js for the before/after, and lib/utils/media-path-guard.js for the full rationale (including why the naive assertSafeMediaPath-only version of this patch shipped in an earlier draft and failed against a live PoC before this fix was added).

VULN-04 — Decompression bomb
Workbook#load() now reads each zip entry's declared uncompressed size from the archive's central directory before decompressing it, and refuses to proceed if a single entry (default cap: 128MB) or the archive as a whole (default cap: 512MB) would exceed a sane limit — without ever materializing the bomb in memory.

// throws: Refusing to decompress "xl/worksheets/sheet1.xml": declared
// uncompressed size (1610612736 bytes) exceeds the per-entry limit ...
await workbook.xlsx.load(maliciousBuffer);

// Both limits are configurable if your legitimate files are larger:
await workbook.xlsx.load(buffer, {
  maxEntryUncompressedSize: 256 * 1024 * 1024,
  maxTotalUncompressedSize: 1024 * 1024 * 1024,
});
VULN-03 — CSV / formula injection
CSV.write() now prefixes any value beginning with =, +, -, @, tab, or CR with a leading apostrophe before handing it to fast-csv. Every mainstream spreadsheet application treats a leading ' as "force this cell to be text" — the apostrophe itself isn't shown, so the cell still displays exactly as written, it just never gets evaluated as a formula.

const rows = [['Item', '=cmd|"/c calc"!A0']];
rows.forEach(r => sheet.addRow(r));
await workbook.csv.writeBuffer();
// -> Item,'=cmd|"/c calc"!A0
//    (the payload is now inert text, not a live formula)
This is applied to the default mapper and to any custom options.map you supply — the wrapping happens after your mapper runs, so it protects custom export logic too. If you need the raw, unescaped behavior (e.g. a purely numeric export you've already validated), opt out explicitly:

await workbook.csv.writeBuffer({escapeFormulas: false});
Installing
npm install exceljs-hardened
The API is otherwise unchanged from upstream exceljs@4.4.0 — this is a drop-in replacement:

- const ExcelJS = require('exceljs');
+ const ExcelJS = require('exceljs-hardened');
What this fork does not do
It does not track every upstream change — it starts from exceljs@4.4.0 plus the specific security patches above. If upstream ever becomes active again, migrating back to the official package is recommended.
It makes no claim of a full, independent security audit beyond the four issues listed above.
Upgrading from 4.x
5.0.0 deliberately breaks a few edge cases that upstream 4.4.0 used to allow silently, because those edge cases were the exploitable behavior:

addImage({filename}) now throws on a raw path containing .., and Workbook.utils.safeJoin() (new) is the supported way to build such a path from user input — see the VULN-02 section above.
Workbook#load() now throws by default on any zip entry declaring more than 128MB uncompressed, or an archive totalling more than 512MB — both configurable via options.maxEntryUncompressedSize / options.maxTotalUncompressedSize if your legitimate files are larger.
CSV.write() now prefixes formula-trigger characters with ' by default — opt out per-call with {escapeFormulas: false} if you rely on the raw output.
If you hit one of these in an existing app, it means the input in question matches the exact shape that was exploitable — that's the point. Adjust the input, raise the relevant limit deliberately, or opt out for that call site if you've already validated it's safe.

License
MIT, same as upstream — see LICENSE. Original copyright (c) 2014-2019 Guyon Roche is preserved as required by the license; the security patches in this fork are additional contributions on top of that original work.

------

Role: You are a Senior Financial Analyst.
Objective: Provide a professional, concise, and data-driven response to the analysis question based ONLY on the provided internal context (Financial Reports and Carmel System data).

Instructions for Internal Evidence:

For data from Financial Reports: You must provide the Report Name, the specific Page Number, and a direct relevant Quote.

For data from the Carmel System: You must specify the Field Name and the exact Value/Data point.

Do not list or mention any internet-based sources or URLs in the sources section below. Focus exclusively on the provided internal documents and system data.

Strict Output Format:
[Provide your professional analysis here]

--- SOURCES ---

[Source Name] | Page: [Number] | Quote: "[Exact Quote]"

Carmel System | Field: [Field Name] | Value: [Value]

Analysis Question: {question}


‫Gemini אמר

קודם כל, תיקון קטן וידידותי: נכון לזמן שאנחנו מדברים בו (2026), המודל שאת משתמשת בו הוא כנראה Gemini 3 Flash (או גרסה מתקדמת של סדרת ה-1.5 אם טרם שדרגת). השם פחות קריטי, העיקרון הטכני נשאר דומה – המטא-דאטה הוא המפתח.

כדי לבנות את הקישורים מה-groundingMetadata, את לא צריכה "להמציא" אותם. המודל מחזיר לך אובייקט מובנה שבו הקישורים כבר קיימים. התפקיד שלך הוא רק "לדוג" אותם משם ולחבר אותם לטקסט.

להלן השלבים הטכניים לביצוע השליפה והחיבור:

1. הבנת מבנה ה-Metadata
כשהתשובה חוזרת מה-API, היא מכילה שדה שנקרא grounding_metadata. בתוכו יש רשימה של grounding_chunks. כל "צ'אנק" כזה מייצג מקור שהמודל השתמש בו.

אם המקור הוא אינטרנט: יהיה לו שדה web שמכיל uri (הקישור) ו-title (כותרת האתר).

אם המקור הוא קובץ שהעלית: הוא יופיע כ-retrieved_context.

2. שלבי העבודה בקוד (אלגוריתם)
פיצול הטקסט: חלקי את התשובה לפי המפריד --- SOURCES ---.

חילוץ מקורות פנימיים: קחי את החלק השני של הטקסט (הדוחות ומערכת כרמל) כפי שהם.

חילוץ קישורים חיצוניים: עברי בלולאה על ה-grounding_chunks במטא-דאטה וחלצי את ה-URIs.

איחוד תצוגה: הציגי למשתמש את הניתוח, ומתחתיו את שתי הרשימות מאוחדות.

3. דוגמת קוד (Python) למימוש השליפה
כך נראה המימוש ששואב את הקישורים האמיתיים שהמודל מצא באינטרנט:

Python
def process_gemini_response(response):
    # 1. חילוץ הטקסט המלא מהמודל
    full_text = response.text
    
    # 2. פיצול בין התשובה למקורות הפנימיים (שהגדרנו בפרומפט)
    parts = full_text.split("--- SOURCES ---")
    analysis_part = parts[0].strip()
    internal_sources = parts[1].strip() if len(parts) > 1 else ""

    # 3. שליפת קישורים מה-Grounding Metadata (מקורות אינטרנט)
    web_sources = []
    metadata = response.candidates[0].grounding_metadata
    
    if hasattr(metadata, 'grounding_chunks'):
        for chunk in metadata.grounding_chunks:
            if chunk.web: # בודק אם זה מקור מהאינטרנט
                source_info = f"* {chunk.web.title}: {chunk.web.uri}"
                if source_info not in web_sources: # מניעת כפילויות
                    web_sources.append(source_info)

    # 4. בניית התצוגה הסופית
    final_sources_section = internal_sources + "\n" + "\n".join(web_sources)
    
    return analysis_part, final_sources_section
למה הדרך הזו עדיפה?
אמינות: הקישור ב-uri מגיע ישירות ממנוע החיפוש של גוגל שמחובר למודל. זה מבטיח שהקישור תקין וקיים.

ניקיון: המשתמש לא רואה כתובות URL ארוכות ומסורבלות בתוך הניתוח הפיננסי עצמו, אלא רשימה מסודרת בסוף.

גמישות: אם מחר תרצי להציג את הקישורים כ"כפתורים" בממשק המשתמש שלך (UI), יהיה לך קל מאוד לעשות זאת כי הם כבר נמצאים אצלך במערך (Array) נפרד.

הערה חשובה: אם את משתמשת ב-Vertex AI או ב-Google AI Studio, ודאי שסימנת ב-Safety Settings או ב-Tools את האפשרות של Google Search Grounding, אחרת המטא-דאטה הזה פשוט יחזור ריק.

האם תרצי שאעזור לך לעצב את הפורמט הסופי שבו המקורות יוצגו למשתמש (למשל כטבלה או כרשימת "הערות שוליים")?


----------------------------------------------------

SYSTEM ROLE: You are a Senior Financial Analyst. Your output is used by a machine-parsing script; therefore, any deviation from the structure will break the system.

OBJECTIVE: Analyze the internal context (Financial Reports/Carmel System) to answer the user's question.

STRICT FORMATTING RULES:

NO CONVERSATIONAL FILLER: Do not start with "Here is the analysis" or "Based on the data". Start directly with the analysis.

MANDATORY SEPARATOR: You MUST include the exact string --- SOURCES --- on a new line after your analysis.

NO EXTERNAL LINKS: Do not mention internet sources or URLs. Only internal data.

EMPTY STATE: If no internal sources are found, you MUST still write the separator followed by "No internal sources used."

INTERNAL EVIDENCE GUIDELINES:

Financial Reports: [Report Name] | Page: [Number] | Quote: "[Exact Quote]"

Carmel System: Carmel System | Field: [Field Name] | Value: [Value]

USER QUESTION: {question}

OUTPUT STRUCTURE (FOLLOW EXACTLY):
[Detailed Professional Analysis]

--- SOURCES ---
[List each source on a new line using the guidelines above]
---------------------------
<p dir="RTL" style="margin: 0px;color: rgb(34, 34, 34);font-family: Arial, Helvetica, sans-serif;font-size: small;font-style: normal;font-weight: 400;text-align: start;text-indent: 0px;background-color: rgb(255, 255, 255);"><span style="font-size:16px;font-family: David, sans-serif;color: blue;">שלום רב,</span></p>
<p dir="RTL" style="margin: 0px;color: rgb(34, 34, 34);font-family: Arial, Helvetica, sans-serif;font-size: small;font-style: normal;font-weight: 400;text-align: start;text-indent: 0px;background-color: rgb(255, 255, 255);"><span dir="LTR" style="font-size:16px;font-family: David, sans-serif;color: blue;">&nbsp;</span></p>
<p dir="RTL" style="margin: 0px;color: rgb(34, 34, 34);font-family: Arial, Helvetica, sans-serif;font-size: small;font-style: normal;font-weight: 400;text-align: start;text-indent: 0px;background-color: rgb(255, 255, 255);"><span style="font-size:16px;font-family: David, sans-serif;color: blue;">אני פונה אליכם בנוגע לבטוחה מסוג [סוג בטוחה] - [תיאור בטוחה] (מספר בטוחה: [מספר בטוחה] בתיקכם (מספר חשבון: [מספר חשבון] על שם [שם לקוח].</span></p>
<p dir="RTL" style="margin: 0px;color: rgb(34, 34, 34);font-family: Arial, Helvetica, sans-serif;font-size: small;font-style: normal;font-weight: 400;text-align: start;text-indent: 0px;background-color: rgb(255, 255, 255);"><span style="font-size:16px;font-family: David, sans-serif;color: blue;">על פי רישומינו, <strong>תוקף השמאות/הפטור משמאות לבטוחה זו יפוג</strong> ביום [תוקף פטור משמאות / תאריך תוקף ביטוח].</span></p>
<p dir="RTL" style="margin: 0px;color: rgb(34, 34, 34);font-family: Arial, Helvetica, sans-serif;font-size: small;font-style: normal;font-weight: 400;text-align: start;text-indent: 0px;background-color: rgb(255, 255, 255);"><span dir="LTR" style="font-size:16px;font-family: David, sans-serif;color: blue;">&nbsp;</span></p>
<p dir="RTL" style="margin: 0px;color: rgb(34, 34, 34);font-family: Arial, Helvetica, sans-serif;font-size: small;font-style: normal;font-weight: 400;text-align: start;text-indent: 0px;background-color: rgb(255, 255, 255);"><u><span style="font-size:16px;font-family: David, sans-serif;color: blue;">לשמירה על תקינות הבטוחה ותנאי האשראי, נדרש לחדש את השמאות בהקדם</span></u><span style="font-size:16px;font-family: David, sans-serif;color: blue;">.</span></p>
<p dir="RTL" style="margin: 0px;color: rgb(34, 34, 34);font-family: Arial, Helvetica, sans-serif;font-size: small;font-style: normal;font-weight: 400;text-align: start;text-indent: 0px;background-color: rgb(255, 255, 255);"><span dir="LTR" style="font-size:16px;font-family: David, sans-serif;color: blue;">&nbsp;</span></p>
<p dir="RTL" style="margin: 0px;color: rgb(34, 34, 34);font-family: Arial, Helvetica, sans-serif;font-size: small;font-style: normal;font-weight: 400;text-align: start;text-indent: 0px;background-color: rgb(255, 255, 255);"><span style="font-size:16px;font-family: David, sans-serif;color: blue;">אנא פעלו להזמנת שמאות חדשה לנכס על ידי שמאי/ת מקרקעין המאושר/ת על ידי הבנק.</span></p>
<p dir="RTL" style="margin: 0px;color: rgb(34, 34, 34);font-family: Arial, Helvetica, sans-serif;font-size: small;font-style: normal;font-weight: 400;text-align: start;text-indent: 0px;background-color: rgb(255, 255, 255);"><span dir="LTR" style="font-size:16px;font-family: David, sans-serif;color: blue;">&nbsp;</span></p>
<p dir="RTL" style="margin: 0px;color: rgb(34, 34, 34);font-family: Arial, Helvetica, sans-serif;font-size: small;font-style: normal;font-weight: 400;text-align: start;text-indent: 0px;background-color: rgb(255, 255, 255);"><span style="font-size:16px;font-family: David, sans-serif;color: blue;">את השמאות המעודכנת יש להעביר אלינו עד לתאריך [תאריך יעד לחידוש].</span></p>
<p dir="RTL" style="margin: 0px;color: rgb(34, 34, 34);font-family: Arial, Helvetica, sans-serif;font-size: small;font-style: normal;font-weight: 400;text-align: start;text-indent: 0px;background-color: rgb(255, 255, 255);"><span dir="LTR" style="font-size:16px;font-family: David, sans-serif;color: blue;">&nbsp;</span></p>
<p dir="RTL" style="margin: 0px;color: rgb(34, 34, 34);font-family: Arial, Helvetica, sans-serif;font-size: small;font-style: normal;font-weight: 400;text-align: start;text-indent: 0px;background-color: rgb(255, 255, 255);"><span style="font-size:16px;font-family: David, sans-serif;color: blue;">לשאלות וסיוע, אשמח לענות לכל שאלה.</span></p>
<p dir="RTL" style="margin: 0px;color: rgb(34, 34, 34);font-family: Arial, Helvetica, sans-serif;font-size: small;font-style: normal;font-weight: 400;text-align: start;text-indent: 0px;background-color: rgb(255, 255, 255);"><span dir="LTR" style="font-size:16px;font-family: David, sans-serif;color: blue;">&nbsp;</span></p>
<p dir="RTL" style="margin: 0px;color: rgb(34, 34, 34);font-family: Arial, Helvetica, sans-serif;font-size: small;font-style: normal;font-weight: 400;text-align: start;text-indent: 0px;background-color: rgb(255, 255, 255);"><span style="font-size:16px;font-family: David, sans-serif;color: blue;">תודה על שיתוף פעולה.</span></p>
<p dir="RTL" style="margin: 0px;color: rgb(34, 34, 34);font-family: Arial, Helvetica, sans-serif;font-size: small;font-style: normal;font-weight: 400;text-align: start;text-indent: 0px;background-color: rgb(255, 255, 255);"><span dir="LTR">&nbsp;</span></p>


----------------
element.style {
    /* position: relative; */
    height: 100%;
    background-color: transparent;
}
#app {
    font-family: Segoe UI, Helvetica Neue, Roboto, sans-serif;
    -ms-text-size-adjust: 100%;
    -webkit-text-size-adjust: 100%;
    line-height: normal;
}
.pyramid-app-container {
   position: static !important; 
}

11 Vulnerabilities

Last Scan Status 
9.8
v3
XRAY-1060233
Locked
exceljs:4.4.0
N/A
CWE-1321
9.3
v4
CVE-2026-78207
Locked
exceljs:4.4.0
N/A
CWE-1321
7.5
v3
CVE-2026-77078
Locked
multer:2.2.0
2.3.0
CWE-248
7.5
v3
CVE-2026-77037
Locked
multer:2.2.0
2.3.0
CWE-459 (+1)
7.5
v3
CVE-2026-82333
Locked
multer:2.2.0
2.3.0
CWE-400
7.5
v3
XRAY-1060232
Locked
exceljs:4.4.0
N/A
CWE-73 (+1)
8.2
v3
XRAY-1060231
Locked
exceljs:4.4.0
N/A
CWE-1236
7.5
v3
XRAY-1060230
Locked
exceljs:4.4.0
N/A
CWE-409
8.4
v4
CVE-2026-78209
Locked
exceljs:4.4.0
N/A
CWE-1236
8.7
v4
CVE-2026-78208
Locked
exceljs:4.4.0
N/A
CWE-73
8.7
v4
CVE-2026-78206
Locked
exceljs:4.4.0
N/A
CWE-409
