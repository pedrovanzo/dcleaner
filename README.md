# dcleaner

This repo will hold an agent that scans and cleans temporary files safely, leaving warnings and logs on each sweep.

The first run will be AI assisted (Claude), but subsequent runs will be shaped by an agent that will come from the dasdk app.

This repo enforces line endings through `.gitattributes`.[^eol]

---

[^eol]: **Line endings (LF vs CRLF).** *Temporary home: this note will move to its proper documentation location.*

    Each line of a text file ends with an invisible marker. **LF** (`\n`, byte `0x0A`) is used by Linux, macOS and most servers. **CRLF** (`\r\n`, bytes `0x0D 0x0A`) is used by Windows.

    **Why it matters:**
    - Shell scripts (`.sh`) with CRLF fail on Linux, for example with `/bin/bash^M: bad interpreter`.
    - Scripts or config copied into Docker containers or CI runners can break the same way.
    - If endings flip, Git shows every line as changed. This causes noisy diffs and false merge conflicts.
    - Mixed Windows and Mac/Linux teams keep overwriting each other's endings.
    - Some Windows-only files (`.bat`, `.cmd`) expect CRLF.

    **Git's `core.autocrlf` setting (per machine):**
    - `true` (Git for Windows default): converts CRLF to LF on commit and LF to CRLF on checkout. This causes the warning `LF will be replaced by CRLF the next time Git touches it`.
    - `input`: converts CRLF to LF on commit and never converts on checkout.
    - `false`: no conversion.

    **This repo's approach:** a committed `.gitattributes` file. Its rules travel with the repo and override each machine's `core.autocrlf`:

    ```gitattributes
    * text=auto eol=lf
    *.bat text eol=crlf
    *.cmd text eol=crlf
    *.ps1 text eol=crlf
    ```

    All text files are stored and checked out as LF, except Windows scripts, which are checked out as CRLF.

    **Optional global default** for other repos: `git config --global core.autocrlf input`.

    **After changing rules** in an existing repo, run `git add --renormalize .` and commit to fix files already tracked.
