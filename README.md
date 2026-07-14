# Splunk Learning Pack

A self-contained set of beginner Splunk resources.

## Contents

| File | What it is |
|------|------------|
| `splunk-search-for-beginners.md` | The core 16-section lesson — SPL fundamentals, fields, the pipe, `stats`, `timechart`, `eval`/`where`, time ranges, search modes, troubleshooting, and performance. |
| `splunk-search-cheatsheet.pdf` | One-page printable quick reference. Print at 100% on US Letter. |
| `splunk-search-for-beginners.lia.md` | The same lesson as an interactive **LiaScript** course with inline quizzes. |
| `splunk-home-lab-setup.md` | Step-by-step guide to a free Splunk home lab (install, Free license, Universal Forwarder, Sysmon). |

## How to use each file

- **Markdown files** (`.md`) open in any text editor or Markdown viewer.
- **The cheat-sheet** (`.pdf`) is ready to print.
- **The LiaScript course** (`.lia.md`) renders as an interactive course in the
  browser. Preview it by pasting the contents into the live editor at
  <https://liascript.github.io/LiveEditor/>, or, once the file is hosted at a
  public raw URL, open it via
  `https://liascript.github.io/course/?<RAW_URL_TO_THE_FILE>`.

## Suggested learning path

1. Read `splunk-search-for-beginners.md` (or run the interactive `.lia.md`).
2. Keep `splunk-search-cheatsheet.pdf` next to you while practicing.
3. Stand up a real environment with `splunk-home-lab-setup.md` and search your
   own data.

---

*Independent study summaries compiled from Splunk's official Search Tutorial and
documentation. Splunk and SPL are trademarks of their respective owner; these are
not official Splunk publications.*
