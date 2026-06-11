# submission-check

A [Claude Code](https://claude.com/claude-code) skill that checks your paper submission materials for **anonymity violations and desk-reject risks** before you hit submit.

Point it at your paper PDF, supplementary PDFs (including annotated/reMarkable exports), and video figures. Claude then runs an exhaustive, **fully local** sweep — nothing is uploaded anywhere:

- 🔴 **Metadata leaks** — PDF Author/Creator/XMP fields, annotation author names (reMarkable exports love to embed these), video `artist`/`encoder` tags, macOS download-provenance (`kMDItemWhereFroms`)
- 🔴 **Body-text leaks** — acknowledgments, grant numbers, IRB institution names, first-person self-citations ("our previous work [12]"), email addresses, the page-1 author block
- 🔴 **Hyperlinks** — every link extracted from the PDF and judged: personal GitHub, homepage, non-anonymous Drive/OSF/YouTube links
- 👁 **Visual inspection** — Claude actually looks at every PDF page and at sampled video frames: usernames in screenshots, file paths, faces, title cards, recognizable locations, logos
- 📋 **Format compliance** — page count vs. venue limit, template/page size, broken `??` citations
- 🟡 **Human-judgment flags** — deployment sites only your team uses, speech segments to listen to, "anonymized" links to verify

The report is tiered (🔴 must fix / 🟡 should review / 🟢 passed) with exact page numbers, timestamps, and field names — and Claude offers to apply the automatable fixes (always to a new file, never overwriting your original).

## Install

```bash
git clone https://github.com/WeiWu0115/submission-check.git ~/.claude/skills/submission-check
cd ~/.claude/skills/submission-check
cp identity.example.md identity.local.md   # then edit with your own name/affiliation/handles
```

`identity.local.md` holds the identity terms the scan looks for. It is gitignored and never leaves your machine. (If you skip this step, the skill asks for your details on first run and creates it for you.)

### Requirements

- Claude Code
- `poppler` (`pdfinfo`, `pdftotext`), `ffmpeg`/`ffprobe`, `python3` with `pypdf`

```bash
brew install poppler ffmpeg && python3 -m pip install pypdf   # macOS
```

## Usage

In any Claude Code session:

```
/submission-check ~/Desktop/CHI2027/paper.pdf ~/Desktop/CHI2027/video.mp4 CHI
```

or just point it at a folder:

```
/submission-check ~/Desktop/CHI2027/ CHIPlay
```

The skill works in whatever language you converse in.

## Caveats

- It's a safety net, not a guarantee — venue rules change; always check the current CFP.
- Audio is not transcribed locally; the skill locates speech segments and asks you to listen.
- Designed around ACM SIGCHI-family venues (CHI, CHI PLAY, TEI, UIST, DIS) but the checks are generic.

## License

MIT
