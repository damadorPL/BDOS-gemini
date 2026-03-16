---
description: Set up Gemini CLI for BDOS by generating GEMINI.md, syncing BDOS skills into .gemini/skills, and fixing missing skill synchronization
---

# Setup Gemini CLI dla BDOS

Skill robi dwie rzeczy:
1. Generuje/aktualizuje `GEMINI.md` — plik kontekstowy dla Gemini CLI (odpowiednik `CLAUDE.md`)
2. Tworzy natywne skille Gemini CLI w `.gemini/skills/` — auto-discovery zamiast `@ścieżka`

## Jak działają skille w Gemini CLI

Gemini CLI ma natywny system skilli — inny niż Claude Code:

| | Claude Code | Gemini CLI |
|--|-------------|-----------|
| Lokalizacja | `.claude/skills/<name>/SKILL.md` | `.gemini/skills/<name>/SKILL.md` |
| Aktywacja | Slash komenda np. `/bdos-check` | Automatyczna — Gemini dobiera skill po opisie |
| Lista skilli | wbudowana | `/skills` lub `/skills list` |
| Skrypty | brak | opcjonalny podkatalog `scripts/` |

**Struktura skilla Gemini:**
```
.gemini/
└── skills/
    └── bdos-check/
        ├── SKILL.md       # opis + instrukcje
        └── scripts/       # opcjonalne skrypty pomocnicze
```

**Format SKILL.md** (identyczny jak w Claude Code):
```markdown
---
name: bdos-check
description: Diagnostyka konta Google Ads — kondycja, wyniki, problemy, rekomendacje
---

# Treść skilla...
```

## Krok 1 — Zbierz listę skilli BDOS

```python
import sys; sys.stdout.reconfigure(encoding='utf-8')
from pathlib import Path

# Źródła — READ ONLY, nigdy nie zapisuj do tych folderów
skill_dirs = [
    Path("bdos/data/claude/skills"),  # skille systemowe (core)
    Path("my/skills"),                # skille użytkownika
]
skills = []
for base in skill_dirs:
    if not base.exists():
        continue
    for d in sorted(base.iterdir()):
        if d.is_dir():
            skill_file = d / "SKILL.md"
            if skill_file.exists():
                content = skill_file.read_text(encoding="utf-8")
                desc = ""
                if content.startswith("---"):
                    for line in content.split("\n")[1:]:
                        if line.startswith("---"):
                            break
                        if line.startswith("description:"):
                            desc = line.replace("description:", "").strip()
                skills.append((d.name, desc, skill_file))

for name, desc, path in skills:
    print(f"{name}: {desc}")
    print(f"  path: {path}")
```

## Krok 2 — Utwórz natywne skille Gemini CLI

Skopiuj skille BDOS do `.gemini/skills/` — wtedy Gemini CLI auto-wykrywa je i aktywuje gdy pasują do intencji użytkownika.

```python
import sys; sys.stdout.reconfigure(encoding='utf-8')
import shutil
from pathlib import Path

# Źródła — READ ONLY, nigdy nie zapisuj do tych folderów
skill_dirs = [
    Path("bdos/data/claude/skills"),  # skille systemowe (core)
    Path("my/skills"),                # skille użytkownika
]
# Cel — JEDYNE miejsce zapisu
gemini_skills_dir = Path(".gemini/skills")
gemini_skills_dir.mkdir(parents=True, exist_ok=True)

copied = []
skipped = []

for base in skill_dirs:
    if not base.exists():
        continue
    for src_dir in sorted(base.iterdir()):
        if not src_dir.is_dir():
            continue
        skill_file = src_dir / "SKILL.md"
        if not skill_file.exists():
            continue

        dst_dir = gemini_skills_dir / src_dir.name
        dst_skill = dst_dir / "SKILL.md"

        # Sprawdź czy już istnieje i jest aktualny
        if dst_skill.exists():
            src_mtime = skill_file.stat().st_mtime
            dst_mtime = dst_skill.stat().st_mtime
            if dst_mtime >= src_mtime:
                skipped.append(src_dir.name)
                continue

        dst_dir.mkdir(parents=True, exist_ok=True)

        # Skopiuj SKILL.md
        content = skill_file.read_text(encoding="utf-8")

        # Upewnij się że frontmatter zawiera 'name'
        if content.startswith("---"):
            lines = content.split("\n")
            has_name = any(l.startswith("name:") for l in lines[1:lines.index("---", 1)] if "---" in lines[1:])
            if not has_name:
                # Dodaj name po pierwszym ---
                end_idx = content.index("---", 3)
                content = content[:end_idx] + f"name: {src_dir.name}\n" + content[end_idx:]

        dst_skill.write_text(content, encoding="utf-8")

        # Skopiuj scripts/ jeśli istnieje
        scripts_src = src_dir / "scripts"
        if scripts_src.exists():
            shutil.copytree(scripts_src, dst_dir / "scripts", dirs_exist_ok=True)

        copied.append(src_dir.name)

print(f"Skopiowano: {len(copied)} skilli -> .gemini/skills/")
if copied:
    for name in copied:
        print(f"  + {name}")
if skipped:
    print(f"Pominięto (aktualne): {len(skipped)}")
```

## Krok 3 — Wygeneruj GEMINI.md

`GEMINI.md` to kontekst systemowy — odpowiednik `CLAUDE.md`. Gemini CLI wczytuje go automatycznie.

```python
import sys; sys.stdout.reconfigure(encoding='utf-8')
from pathlib import Path

claude_md = Path("CLAUDE.md").read_text(encoding="utf-8")

# Zamień nagłówek
gemini_content = claude_md.replace(
    "Edytuj: my/CLAUDE.md (dodatkowe instrukcje)",
    "Edytuj: my/GEMINI.md (dodatkowe instrukcje dla Gemini CLI)"
)

# Zbierz skille z .gemini/skills/ (natywna lokalizacja)
gemini_skills_dir = Path(".gemini/skills")
skills_lines = []
if gemini_skills_dir.exists():
    for d in sorted(gemini_skills_dir.iterdir()):
        if d.is_dir():
            skill_file = d / "SKILL.md"
            if skill_file.exists():
                content = skill_file.read_text(encoding="utf-8")
                desc = ""
                if content.startswith("---"):
                    for line in content.split("\n")[1:]:
                        if line.startswith("---"):
                            break
                        if line.startswith("description:"):
                            desc = line.replace("description:", "").strip()
                skills_lines.append(f"| `{d.name}` | {desc} |")

skills_section = """

---

## Skille BDOS w Gemini CLI

Skille są zainstalowane w `.gemini/skills/` i **aktywują się automatycznie** gdy Gemini CLI rozpozna intencję.
Sprawdź dostępne skille: `/skills` lub `/skills list`

### Dostępne skille

| Skill | Co robi |
|-------|---------|
""" + "\n".join(skills_lines) + """

### Jak używać

Gemini CLI auto-aktywuje właściwy skill na podstawie opisu — wystarczy napisać naturalnie:

```
sprawdź konto X
```
→ Gemini aktywuje `bdos-check`

```
pokaż zmiany z ostatnich 7 dni na koncie X
```
→ Gemini aktywuje `bdos-changes`

```
utwórz kampanię Search dla konta X
```
→ Gemini aktywuje `bdos-create`

### Ręczna aktywacja przez @ścieżka (fallback)

Jeśli auto-aktywacja nie zadziała, możesz nadal załadować skill ręcznie:

```
@.gemini/skills/bdos-check/SKILL.md sprawdź konto X
```

### Aktualizacja skilli

Po dodaniu nowego skilla lub `bdos update` uruchom ponownie `/gemini-setup`
żeby zsynchronizować `.gemini/skills/` z `.claude/skills/`.
"""

final = gemini_content.rstrip() + skills_section

Path("GEMINI.md").write_text(final, encoding="utf-8")
print(f"GEMINI.md zaktualizowany — {len(skills_lines)} skilli")
```

## Krok 4 — Utwórz my/GEMINI.md (jeśli nie istnieje)

```python
import sys; sys.stdout.reconfigure(encoding='utf-8')
from pathlib import Path

gemini_user = Path("my/GEMINI.md")
if not gemini_user.exists():
    gemini_user.write_text(
        "# Moje instrukcje dla Gemini CLI\n\n"
        "<!-- Wpisz tu dodatkowe instrukcje specyficzne dla Gemini CLI.\n"
        "     Ten plik NIGDY nie jest nadpisywany przez bdos update.\n"
        "     Po edycji uruchom: /gemini-setup -->\n\n"
        "## Moje preferencje\n\n",
        encoding="utf-8"
    )
    print("Utworzono my/GEMINI.md")
else:
    print("my/GEMINI.md już istnieje — pomijam")
```

## Krok 5 — Weryfikacja

Uruchom Gemini CLI i sprawdź:

```
/skills list
```

Powinny pojawić się wszystkie skille BDOS. Jeśli nie — sprawdź czy `.gemini/skills/` istnieje w katalogu projektu.

## Krok 6 — Podsumuj

Po wykonaniu powiedz użytkownikowi:

> **Gemini CLI skonfigurowany.**
>
> Skille BDOS są zainstalowane w `.gemini/skills/` i aktywują się automatycznie.
> Sprawdź listę: `/skills list`
>
> Przykład: napisz *"sprawdź konto X"* — Gemini sam dobierze właściwy skill.
>
> Kontekst systemowy: `GEMINI.md` | Twoje instrukcje: `my/GEMINI.md`

## Kiedy ponowić

Uruchom `/gemini-setup` ponownie gdy:
- dodajesz nowy skill do `my/skills/`
- robisz `bdos update` (`bdos/data/claude/skills/` się zmienia → `.gemini/skills/` do synchronizacji)
- edytujesz `my/GEMINI.md`
- skille w `.gemini/skills/` są nieaktualne (Krok 2 sprawdza mtimes i kopiuje tylko zmienione)
