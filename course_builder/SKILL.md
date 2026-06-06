---
name: moodle-course-builder
description: >
  Build a complete, importable Moodle course (.mbz backup file) from a source document or URL,
  optionally using an existing .mbz file as a structural template. Supports embedding H5P
  interactive activities (Flashcards, Dialog Cards, Drag the Words, Accordion, Word Search)
  alongside standard Moodle activities. Use this skill whenever the user wants to: create a
  Moodle course from content, convert a document or webpage into a Moodle course, package
  activities into an .mbz file, generate Moodle XML questions from source material, create H5P
  interactive elements, or automate course building for Moodle. Triggers on phrases like "build
  a Moodle course", "turn this into a course", "create an .mbz", "package this as a Moodle
  backup", "generate questions for Moodle", "create H5P activities", "add interactive elements",
  or any request to convert content into a structured Moodle-importable format.
---

# Moodle Course Builder Skill

Converts a source document or URL into a fully packaged Moodle `.mbz` backup file, with
Moodle XML question bank and optional H5P interactive activities, ready to import into
Moodle 4.x or 5.x.

---

## Inputs

| Input | Required | Description |
|---|---|---|
| Source content | **Yes** | A URL, uploaded PDF, Word doc, or pasted text — the course content |
| `.mbz` template | No | An existing Moodle backup used as a structural/platform reference |
| H5P templates | No | `.h5p` files used as structural templates for interactive activities |

---

## Mandatory Workflow

### 1. Ingest

**Source content:**
- URL → use web fetch tool to retrieve full content
- PDF/DOCX → read with file tools
- Pasted text → use directly

**`.mbz` template (if provided):**
```bash
mkdir -p /tmp/course_builder/mbz_template
cp /path/to/uploads/[filename].mbz /tmp/course_builder/template.mbz
cd /tmp/course_builder/mbz_template && tar -xzf ../template.mbz
```
Inspect:
- `course/course.xml` — custom fields, format, category
- `moodle_backup.xml` — Moodle version, backup version
- One activity XML — HTML encoding style, module conventions

**H5P templates (if provided):**
```bash
mkdir -p /tmp/course_builder/h5p_inspect/[name]
unzip -o /path/to/uploads/[filename].h5p -d /tmp/course_builder/h5p_inspect/[name]
cat /tmp/course_builder/h5p_inspect/[name]/content/content.json
cat /tmp/course_builder/h5p_inspect/[name]/h5p.json
```
Inspect `h5p.json` for library name/version and `content/content.json` for data structure.

---

### 2. Propose Layout (ALWAYS before building)

Present to the user:
- Course full name, short name, summary
- Numbered section list with activity type and one-line description per section
- H5P activities planned (type + section placement + content summary)
- Question count (MCQ + True/False)

**Wait for approval. Do not build until confirmed.**

---

### 3. Build the Course Package

Use Python to generate all XML files, then package with tar.

**Working directory:** `/tmp/course_builder/course_build/[shortname]/`

#### Key XML files to generate:

| File | Purpose |
|---|---|
| `moodle_backup.xml` | Master manifest — lists all sections and activities |
| `course/course.xml` | Course metadata, custom fields, format settings |
| `sections/section_N/section.xml` | Section title, sequence of module IDs |
| `activities/page_N/page.xml` | Page name, intro, HTML body content |
| `activities/book_N/book.xml` | Book name, intro, chapters with HTML content |
| `activities/*/module.xml` | Visibility, completion, section assignment |

#### ID scheme (arbitrary, Moodle remaps on restore):
- Sections: 1001, 1002, 1003…
- Modules: 2001, 2002, 2003…
- Context IDs: 3001, 3002, 3003…
- Activity internal IDs: 4001, 4002, 4003…

#### HTML encoding rule:
All HTML in XML content fields must be entity-encoded:
```python
content_html = content_html.replace('&', '&amp;').replace('<', '&lt;').replace('>', '&gt;')
```

#### Hidden sections (e.g. lecturer notes):
Set `<visible>0</visible>` in both `section.xml` and the corresponding `module.xml`.

#### Packaging:
```bash
cd /tmp/course_builder/course_build/[shortname]
tar -czf /path/to/outputs/[shortname].mbz .
```

---

### 4. Generate Question Bank

Produce a separate `[shortname]-questions.xml` in **Moodle XML format** (not GIFT).

**MCQ structure:**
```xml
<question type="multichoice">
  <name><text>Q: Short title</text></name>
  <questiontext format="html">
    <text><![CDATA[<p>Full question text?</p>]]></text>
  </questiontext>
  <defaultgrade>1</defaultgrade>
  <single>true</single>
  <answer fraction="100">
    <text>Correct answer</text>
    <feedback><text>Correct — brief explanation.</text></feedback>
  </answer>
  <answer fraction="0">
    <text>Distractor 1</text>
    <feedback><text>Incorrect — brief explanation.</text></feedback>
  </answer>
</question>
```

**True/False structure:**
```xml
<question type="truefalse">
  <name><text>TF: Short title</text></name>
  <questiontext format="html">
    <text><![CDATA[<p>Statement to evaluate.</p>]]></text>
  </questiontext>
  <defaultgrade>1</defaultgrade>
  <answer fraction="100">
    <text>true</text>
    <feedback><text>Correct — brief explanation.</text></feedback>
  </answer>
  <answer fraction="0">
    <text>false</text>
    <feedback><text>Incorrect — brief explanation.</text></feedback>
  </answer>
</question>
```

---

### 5. Build H5P Interactive Activities

H5P files are separate `.h5p` uploads — they are NOT embedded inside the `.mbz`.
Create them as companion files for the lecturer to upload via Content Bank > H5P > Upload.

#### Supported H5P Types

| Type | Library | Use Case |
|---|---|---|
| Flashcards | `H5P.Flashcards 1.7` | Term-definition recall (typed answer) |
| Dialog Cards | `H5P.Dialogcards 1.9` | Concept pairs — flip to reveal (no typing) |
| Drag the Words | `H5P.DragText 1.10` | Fill-in-the-blank with draggable words |
| Accordion | `H5P.Accordion 1.0` | Expandable panels for reference content |
| Word Search | `H5P.FindTheWords 1.4` | Vocabulary review — find terms in a grid |

#### H5P File Structure

All H5P files are ZIP archives containing:
```
h5p.json              → Library metadata (title, mainLibrary, version)
content/content.json  → Activity data (cards, word lists, text fields, etc.)
content/images/       → Optional images (if applicable)
content/audios/       → Optional audio files (if applicable)
```

#### Building H5P Files (Python)

```python
import json, zipfile, os

def make_h5p(name, h5p_json, content_json, outdir="/path/to/outputs"):
    workdir = f"/tmp/course_builder/h5p_build/{name}"
    os.makedirs(f"{workdir}/content", exist_ok=True)

    with open(f"{workdir}/h5p.json", "w") as f:
        json.dump(h5p_json, f, separators=(",", ":"))
    with open(f"{workdir}/content/content.json", "w") as f:
        json.dump(content_json, f, separators=(",", ":"))

    with zipfile.ZipFile(f"{outdir}/{name}.h5p", "w", zipfile.ZIP_DEFLATED) as zf:
        zf.write(f"{workdir}/h5p.json", "h5p.json")
        zf.write(f"{workdir}/content/content.json", "content/content.json")
```

#### H5P Content Structures

**Flashcards (`H5P.Flashcards 1.7`):**
```json
{
  "description": "Type the correct term for each definition",
  "cards": [
    {"text": "Clue on front", "answer": "TypedAnswer", "tip": "Optional hint"}
  ],
  "caseSensitive": false,
  "randomCards": true,
  "progressText": "Card @card of @total",
  "checkAnswerText": "Check",
  "correctAnswerText": "Correct!",
  "incorrectAnswerText": "Incorrect!",
  "showSolutionText": "Correct answer:",
  "retry": "Retry", "next": "Next", "previous": "Previous"
}
```

**Dialog Cards (`H5P.Dialogcards 1.9`):**
```json
{
  "title": "<p>Title</p>\n",
  "description": "<p>Instructions</p>\n",
  "dialogs": [
    {"text": "<p>Front text</p>", "answer": "<p>Back text</p>", "tips": {}}
  ],
  "behaviour": {"disableBackwardsNavigation": false, "randomCards": false},
  "answer": "Turn", "next": "Next", "prev": "Previous", "retry": "Retry",
  "correctAnswer": "I got it right!", "incorrectAnswer": "I got it wrong"
}
```

**Drag the Words (`H5P.DragText 1.10`):**
```json
{
  "taskDescription": "<p>Drag the correct words into each blank.</p>\n",
  "textField": "The answer is *correct_word:Optional hint*.\nAnother blank: *answer2*.",
  "distractors": "*wrong1* *wrong2*",
  "behaviour": {"enableRetry": true, "enableSolutionsButton": true, "instantFeedback": false}
}
```
Draggable words are wrapped in `*asterisks*`. Hints follow a colon: `*word:hint text*`.
Distractors are extra words that don't fit any blank.

**Accordion (`H5P.Accordion 1.0`):**
```json
{
  "panels": [
    {
      "title": "Panel Title",
      "content": {
        "params": {"text": "<p>Panel HTML content</p>"},
        "library": "H5P.AdvancedText 1.1",
        "subContentId": "uuid-here",
        "metadata": {"contentType": "Text", "license": "U", "title": "Untitled Text"}
      }
    }
  ]
}
```

**Word Search (`H5P.FindTheWords 1.4`):**
```json
{
  "taskDescription": "Find the terms in the grid below",
  "wordList": "Word1,Word2,Word3",
  "behaviour": {
    "orientations": {
      "horizontal": true, "horizontalBack": true,
      "vertical": true, "verticalUp": true,
      "diagonal": true, "diagonalBack": true,
      "diagonalUp": true, "diagonalUpBack": true
    },
    "fillPool": "abcdefghijklmnopqrstuvwxyz",
    "preferOverlap": true, "showVocabulary": true,
    "enableShowSolution": true, "enableRetry": true
  },
  "l10n": {"check": "Check", "tryAgain": "Retry", "showSolution": "Show Solution",
           "found": "@found of @totalWords found", "wordListHeader": "Terms"}
}
```

---

## Content Standards

- Reading level: Grade 12 / early undergraduate (adjustable)
- HTML tags allowed: `<h3>`, `<p>`, `<ul>`, `<li>`, `<strong>`, `<em>` — no inline styles
- Page activity: 200–400 words of substantive body content
- Book chapters: 150–300 words each
- H5P Flashcards: 8–15 cards per activity
- H5P Dialog Cards: 6–12 cards per activity
- H5P Drag the Words: 4–8 blanks per activity
- H5P Accordion: 4–8 panels per activity
- H5P Word Search: 8–15 words per activity
- No placeholder content — every field must contain real material from the source

---

## Minimum Valid `.mbz` File Set

```
course/course.xml
course/enrolments.xml
course/inforef.xml
sections/section_NNNN/section.xml
sections/section_NNNN/inforef.xml
activities/[type]_NNNN/[type].xml
activities/[type]_NNNN/module.xml
activities/[type]_NNNN/inforef.xml
activities/[type]_NNNN/grades.xml
activities/[type]_NNNN/roles.xml
activities/[type]_NNNN/competencies.xml
activities/[type]_NNNN/filters.xml
activities/[type]_NNNN/calendar.xml
activities/[type]_NNNN/grade_history.xml
activities/[type]_NNNN/comments.xml
activities/[type]_NNNN/completion.xml
activities/[type]_NNNN/xapistate.xml
moodle_backup.xml
files.xml
gradebook.xml
roles.xml
groups.xml
scales.xml
outcomes.xml
questions.xml
badges.xml
completion.xml
grade_history.xml
.ARCHIVE_INDEX
```

---

## Outputs

Present all files for download:
1. `[shortname].mbz` — import via Site Admin > Restore or Course > Restore
2. `[shortname]-questions.xml` — import via Question Bank > Import > Moodle XML format
3. `*.h5p` files — upload via Content Bank > H5P > Upload, then add as H5P activities in each section

---

## Common Errors to Avoid

| Error | Fix |
|---|---|
| Moodle rejects restore | Check `moodle_backup.xml` lists all activities and sections |
| Broken HTML in activities | Ensure all `<` `>` `&` are entity-encoded in XML fields |
| Missing files cause restore failure | Always include `files.xml`, `gradebook.xml`, `grade_history.xml` |
| Questions don't import | Use Moodle XML format, not GIFT; wrap text in `<![CDATA[...]]>` |
| Wrong section sequence | `<sequence>` in section.xml must list module IDs, comma-separated |
| H5P file won't upload | Ensure `h5p.json` library version matches what's installed on the Moodle site |
| H5P content empty after upload | Check `content/content.json` is valid JSON with correct field names |
| Hidden section still visible | Set `<visible>0</visible>` in BOTH `section.xml` AND `module.xml` |
