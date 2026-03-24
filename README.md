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
