---
name: submission-check
description: Pre-submission anonymity & desk-reject risk check for paper PDFs, supplementary PDFs (incl. annotated/reMarkable exports), and videos. Scans metadata, body text, hyperlinks, page visuals, video frames, and format compliance — fully local, nothing is uploaded. Usage: /submission-check <files or directory...> [venue, e.g. CHI/CHIPlay/TEI]
---

# Submission Anonymity & Desk-Reject Check

Run an exhaustive check over the submission materials the user provides and produce a tiered report. **Every check runs locally — never send file contents to any external service.** Write the report in the language the user is conversing in.

Required local tools: `pdfinfo`, `pdftotext` (poppler), `ffprobe`/`ffmpeg`, `python3` + `pypdf`. If one is missing, say so and degrade gracefully (e.g. `pypdf` can replace `pdfinfo`).

## Step 0: Confirm inputs

- If given a directory, `ls -R` it and list every file (PDFs, videos, zips, images all count).
- If the user named a venue (CHI, CHIPlay, TEI, UIST, DIS, …), note its page limits and template rules; do NOT invent limits you are not sure of — mark them "please verify against the CFP" in the report.
- Build a checklist of all files, check each one, then merge into a single report.

## Step 0.5: Load identity terms

Read `identity.local.md` in this skill's directory. It lists the author-identifying strings to scan for (name variants, email, affiliation, GitHub/website handles, project-identifying locations).

- If the file does not exist, ask the user for these items (see `identity.example.md` for the expected shape) and offer to create it for them.
- At the start of every run, **ask the user for this paper's co-authors** and add their names + affiliations to the scan list for this run.

All scans below are case-insensitive and use every term from this list.

## Step 1: Filenames

No filename — including paths inside zips (`unzip -l` or Python `zipfile`) — may contain an author name (e.g. `JaneDoe_CHI.pdf`).

## Step 2: PDFs (main paper and every supplementary PDF, incl. reMarkable/annotated exports)

### 2a. Metadata (🔴 high-risk zone)
```bash
pdfinfo "$f"
python3 - "$f" <<'EOF'
import sys
from pypdf import PdfReader
r = PdfReader(sys.argv[1])
print("DocInfo:", dict(r.metadata or {}))
xmp = r.xmp_metadata
if xmp:
    for a in ("dc_creator","dc_title","dc_description","xmp_creator_tool","pdf_producer"):
        try: print(a, "=", getattr(xmp, a))
        except Exception: pass
# Annotations (comments / sticky notes often carry the author's name;
# reMarkable exports are especially prone to this)
for i, page in enumerate(r.pages):
    for an in (page.get("/Annots") or []):
        obj = an.get_object()
        sub = str(obj.get("/Subtype",""))
        if sub not in ("/Link",):
            print(f"p{i+1} annot {sub}: author={obj.get('/T')} content={str(obj.get('/Contents'))[:80]}")
# Embedded attachments
names = r.trailer["/Root"].get("/Names")
if names and names.get("/EmbeddedFiles"): print("⚠️ has embedded attachments — unpack and check them too")
EOF
```
On macOS also run `mdls -name kMDItemAuthors -name kMDItemWhereFroms "$f"` — `kMDItemWhereFroms` records the download URL, which can leak identity.

Check: Author / Creator / Producer / Title fields, XMP `dc:creator`, annotation author (`/T`), download provenance.

### 2b. Full-text scan
Extract with `pdftotext "$f" -` and scan for:
- Every identity term from Step 0.5.
- Author block on page 1 — an anonymized submission should read `Anonymous Author(s)` or be empty.
- **Acknowledgments**: must be removed or anonymized. `grep -in 'acknowledg\|funding\|grant'`.
- Grant numbers (NSF #…, IRB protocol number + real institution name).
- IRB phrasing: `approved by the IRB of <real institution>` → 🔴; should be `our institution's IRB`.
- First-person self-citation: `our previous work`, `we previously`, `our prior study`, `in our earlier paper` followed by a citation → 🔴 (ACM requires citing your own work in the third person). Check `Anonymous` placeholder entries in the references are consistent.
- Email addresses (regex `[\w.+-]+@[\w-]+\.[\w.]+`).
- Location/institution clues: campus names, city + lab combinations, deployment sites only this team has used (🟡 — reviewers can search these).

### 2c. Hyperlinks (very easy to miss)
```bash
python3 - "$f" <<'EOF'
import sys
from pypdf import PdfReader
for i, page in enumerate(PdfReader(sys.argv[1]).pages):
    for an in (page.get("/Annots") or []):
        obj = an.get_object()
        a = obj.get("/A")
        if a and a.get("/URI"): print(f"p{i+1}: {a['/URI']}")
EOF
```
Judge each link: personal GitHub account, personal homepage, non-anonymous Drive/OSF/YouTube → 🔴. Data/code should use anonymous.4open.science, OSF anonymized view links, etc. **For every "anonymized" link, verify it actually is** — open it (or have the user confirm) and check the page does not reveal an account name.

### 2d. Visual inspection (Read the PDF pages directly)
- Page 1: author block, teaser figure.
- Go through every page, focusing on figures: browser bookmarks / avatars / menu-bar usernames in screenshots, recognizable faces and campus landmarks in photos, figure watermarks, file paths visible in screenshots (e.g. `/Users/<name>/`), logos on posters or materials.
- Annotated (reMarkable) PDFs: any signature or name in the handwriting.

### 2e. Format compliance (common desk-reject causes)
- Page count: `pdfinfo` Pages vs. the venue limit (body-page vs. references rules differ per venue).
- Template: CHI-family venues use the ACM template, usually single column at submission time; TEI Pictorials use a dedicated landscape template — don't mix up the genre.
- `Page size` letter (612 x 792 pts) vs. A4, per the template.
- Broken citations: `??` or `[?]` left by LaTeX.
- Figure alt-text requirements (recent CHI venues) — mark "please verify against the CFP".

## Step 3: Videos

### 3a. Metadata
```bash
ffprobe -v quiet -show_format -show_streams -print_format json "$v" | python3 -m json.tool
```
Check `tags`: artist / author / comment / encoder / copyright, plus `creation_time`; run `mdls` too on macOS.

### 3b. Frame sampling + visual check
```bash
d=$(mktemp -d)
ffmpeg -v error -i "$v" -vf "select='gt(scene,0.3)',showinfo" -vsync vsync_drop "$d/scene_%03d.jpg" 2>/dev/null
ffmpeg -v error -i "$v" -vf fps=1/10 "$d/t_%03d.jpg"   # one frame every 10 s as a fallback
```
Read every frame: title/credit cards (names, institutions, logos), faces on camera, recognizable venues, account names / menu bars / file paths in screen recordings, burned-in subtitles. Report findings with **timestamps** (frame index x interval).

### 3c. Audio
If no local transcription tool is available, detect speech segments from the waveform and list their time ranges as 🟡 "listen manually: does anyone state a name/institution; does the voice need disguising".

## Step 4: Report

Group by file, three tiers, every finding with an exact location (page / timestamp / field name):

- 🔴 **Must fix (desk-reject or direct identity leak)** — give a concrete fix for each (e.g. clear PDF metadata via pypdf; `\documentclass[...,anonymous]{acmart}`).
- 🟡 **Should review (risky, needs human judgment)** — location clues, audio to listen to, "anonymous" links to verify.
- 🟢 **Passed checks** — brief list so the user knows what was confirmed clean.

At the end, offer to run the automatable fixes (clear PDF metadata; strip video tags via `ffmpeg -i in.mp4 -map_metadata -1 -c copy out.mp4`), but **always ask before modifying anything, and always write to a new file — never overwrite the original**.
