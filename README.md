הדרך המומלצת והיעילה ביותר ליצור קבצי Word (בפורמט `.docx`) גם ב-Node.js וגם ב-React היא באמצעות הספרייה **`docx`**. ספרייה זו מאפשרת בנייה של מסמכים בצורה מבנית (Declarative) ותומכת בצורה מעולה בכותרות, פסקאות וטבלאות דינמיות.

מכיוון שהלוגיקה של בניית המסמך זהה לחלוטין בשרת (Node.js) ובצד הלקוח (React), אדגים כיצד לממש זאת ב-React, עם הערה קטנה בסוף לגבי ההתאמה ל-Node.js.

### שלב 1: התקנת ספריות

בפרויקט שלך, התקיני את הספרייה `docx` וכן את `file-saver` שיעזור לנו להוריד את הקובץ בדפדפן:

```bash
npm install docx file-saver

```

### שלב 2: הכנת מבנה הנתונים

נניח שמבנה הנתונים שלך (ה-Content) נראה בערך כך (מערך של אובייקטים):

```javascript
const myContent = [
  {
    title: "כותרת פסקה ראשונה",
    text: "זהו התוכן המילולי של הפסקה הראשונה. כאן נסביר על הנושא.",
    // הטבלה היא מערך של מערכים (שורות שבתוכן עמודות)
    tableData: [
      ["שם המוצר", "כמות", "מחיר"],
      ["מקלדת", "2", "150"],
      ["עכבר", "5", "80"]
    ]
  },
  {
    title: "כותרת פסקה שנייה",
    text: "פסקה זו מכילה טבלה בגודל שונה, כדי להדגים דינמיות.",
    tableData: [
      ["מזהה", "סטטוס"],
      ["1", "פעיל"],
      ["2", "לא פעיל"],
      ["3", "בטיפול"],
      ["4", "הושלם"]
    ]
  }
];

```

### שלב 3: כתיבת פונקציית הייצוא (React)

כאן אנו בונים את המסמך. הטריק לטבלה דינמית הוא להשתמש בפונקציית `map` על מערך הנתונים כדי לייצר את ה-`TableRow` (שורות) וה-`TableCell` (תאים/עמודות) בהתאם לכמות הנתונים בפועל.

```javascript
import React from 'react';
import { Document, Packer, Paragraph, TextRun, Table, TableRow, TableCell, HeadingLevel, WidthType } from "docx";
import { saveAs } from "file-saver";

const ExportToWord = ({ content }) => {

  const generateDocument = async () => {
    const documentChildren = [];

    // עוברים על כל חלקי התוכן
    content.forEach(section => {
      
      // 1. הוספת הכותרת
      documentChildren.push(
        new Paragraph({
          text: section.title,
          heading: HeadingLevel.HEADING_1,
          spacing: { before: 400, after: 200 }, // רווחים לפני ואחרי הכותרת
        })
      );

      // 2. הוספת התוכן המילולי
      documentChildren.push(
        new Paragraph({
          children: [new TextRun(section.text)],
          spacing: { after: 300 },
        })
      );

      // 3. יצירת טבלה דינמית (אם קיימים נתונים)
      if (section.tableData && section.tableData.length > 0) {
        
        // יצירת השורות באופן דינמי
        const tableRows = section.tableData.map(rowData => {
          
          // יצירת העמודות (התאים) עבור כל שורה באופן דינמי
          const tableCells = rowData.map(cellValue => {
            return new TableCell({
              children: [new Paragraph(cellValue.toString())],
              margins: { top: 100, bottom: 100, left: 100, right: 100 }, // ריווח פנימי בתא
            });
          });

          return new TableRow({
            children: tableCells,
          });
        });

        // יצירת הטבלה עצמה
        const table = new Table({
          rows: tableRows,
          width: {
            size: 100,
            type: WidthType.PERCENTAGE, // הטבלה תתפרס על כל רוחב העמוד
          },
        });

        documentChildren.push(table);
      }
    });

    // יצירת אובייקט המסמך המרכזי
    const doc = new Document({
      sections: [
        {
          properties: {},
          children: documentChildren,
        },
      ],
    });

    // אריזת המסמך לקובץ והורדה שלו (בצד הלקוח)
    const blob = await Packer.toBlob(doc);
    saveAs(blob, "Exported_Content.docx");
  };

  return (
    <button onClick={generateDocument} style={{ padding: '10px 20px', cursor: 'pointer' }}>
      ייצא לקובץ Word
    </button>
  );
};

export default ExportToWord;

```

---

### איך זה עובד?

1. **לולאת `forEach`:** עוברת על כל פסקה ומייצרת עבורה אובייקטים של Word (`Paragraph` לכותרות ולטקסט).
2. **טבלה דינמית:** השימוש ב-`section.tableData.map` מאפשר לנו לא להיות תלויים במספר קבוע של שורות או עמודות. אם במערך יש 5 שורות ו-2 עמודות, הלולאה תייצר בדיוק את המספר הזה של אובייקטי `TableRow` ו-`TableCell`.
3. **`Packer`:** לוקח את כל המבנה הזה והופך אותו לקובץ בינארי שניתן לקריאה על ידי תוכנת Word.

### התאמה ל-Node.js (צד שרת)

אם את מעדיפה לייצר את הקובץ בשרת (Node.js) ולשלוח אותו בחזרה ללקוח, קוד בניית המסמך נשאר **זהה לחלוטין**. ההבדל היחיד הוא בסוף התהליך: במקום להשתמש ב-`Packer.toBlob` ולהוריד עם `file-saver`, את משתמשת ב-`Packer.toBuffer` ושולחת אותו בתגובה (Response) מהשרת:

```javascript
// קוד שמבוצע בשרת (לדוגמה ב-Express)
const buffer = await Packer.toBuffer(doc);

res.setHeader('Content-Disposition', 'attachment; filename=Exported_Content.docx');
res.setHeader('Content-Type', 'application/vnd.openxmlformats-officedocument.wordprocessingml.document');
res.send(buffer);

```

האם תרצי שאראה לך כיצד להוסיף עיצובים מתקדמים לטבלה, כמו שורת כותרת מודגשת או צבעי רקע לתאים?
