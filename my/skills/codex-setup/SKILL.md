---
description: Setup Codex for BDOS by generating AGENTS.md with available BDOS skills and creating my/AGENTS.md for user-specific instructions
---

# Setup Codex dla BDOS

Skill robi dwie rzeczy:
1. Generuje/aktualizuje `AGENTS.md` - plik instrukcji projektowych dla Codexa
2. Tworzy `my/AGENTS.md` - miejsce na prywatne instrukcje uzytkownika dla Codexa

## Jak dziala konfiguracja w Codex

Codex nie korzysta tutaj z natywnego katalogu skilli projektu jak Gemini CLI. Zamiast tego czyta instrukcje z `AGENTS.md`, a dostepne skille trzeba mu jawnie opisac z nazwa, opisem i sciezka do `SKILL.md`.

| | Claude Code | Gemini CLI | Codex |
|--|-------------|-----------|-------|
| Glowny plik instrukcji | `CLAUDE.md` | `GEMINI.md` | `AGENTS.md` |
| Skille projektu | `my/skills/<name>/SKILL.md` | `.gemini/skills/<name>/SKILL.md` | `my/skills/<name>/SKILL.md` |
| Aktywacja | Auto po `description` | Auto po `description` | Po zasadach opisanych w `AGENTS.md` |
| Lista skilli | wbudowana | `/skills` | sekcja w `AGENTS.md` |

## Krok 1 - Zbierz liste skilli BDOS

```python
import sys; sys.stdout.reconfigure(encoding='utf-8')
from pathlib import Path

# Zrodla - READ ONLY, nigdy nie zapisuj do tych folderow
skill_dirs = [
    Path("bdos/data/claude/skills"),  # skille systemowe (core)
    Path("my/skills"),                # skille uzytkownika
]

skills = []
for base in skill_dirs:
    if not base.exists():
        continue
    for d in sorted(base.iterdir()):
        if not d.is_dir():
            continue
        skill_file = d / "SKILL.md"
        if not skill_file.exists():
            continue

        content = skill_file.read_text(encoding="utf-8")
        desc = ""
        if content.startswith("---"):
            for line in content.split("\n")[1:]:
                if line.startswith("---"):
                    break
                if line.startswith("description:"):
                    desc = line.replace("description:", "").strip()

        skills.append((d.name, desc, skill_file.as_posix()))

for name, desc, path in skills:
    print(f"{name}: {desc}")
    print(f"  path: {path}")
```

## Krok 2 - Wygeneruj AGENTS.md

`AGENTS.md` jest odpowiednikiem instrukcji projektowych dla Codexa. Najbezpieczniej wygenerowac go z `CLAUDE.md`, a na koncu dopiac sekcje specyficzne dla Codexa: liste skilli i reguly ich uzycia.

```python
import sys; sys.stdout.reconfigure(encoding='utf-8')
from pathlib import Path

claude_md = Path("CLAUDE.md").read_text(encoding="utf-8")

# Zbierz skille
skill_dirs = [
    Path("bdos/data/claude/skills"),
    Path("my/skills"),
]

skills = []
for base in skill_dirs:
    if not base.exists():
        continue
    for d in sorted(base.iterdir()):
        if not d.is_dir():
            continue
        skill_file = d / "SKILL.md"
        if not skill_file.exists():
            continue

        content = skill_file.read_text(encoding="utf-8")
        desc = ""
        if content.startswith("---"):
            for line in content.split("\n")[1:]:
                if line.startswith("---"):
                    break
                if line.startswith("description:"):
                    desc = line.replace("description:", "").strip()

        skills.append((d.name, desc, skill_file.resolve()))

skills_lines = []
for name, desc, path in skills:
    skills_lines.append(
        f"- `{name}`: {desc} (file: {path.as_posix()})"
    )

codex_header = (
    "Edytuj: my/CLAUDE.md (dodatkowe instrukcje)"
    if "Edytuj: my/CLAUDE.md (dodatkowe instrukcje)" in claude_md
    else None
)

agents_content = claude_md
if codex_header:
    agents_content = agents_content.replace(
        "Edytuj: my/CLAUDE.md (dodatkowe instrukcje)",
        "Edytuj: my/AGENTS.md (dodatkowe instrukcje dla Codexa)"
    )

skills_section = """

---

## Skills

A skill is a set of local instructions stored in `SKILL.md`. Codex should use a listed skill when the user names it or when the task clearly matches its description.

### Available skills
""" + "\n".join(skills_lines) + """

### How to use skills

- Discovery: Skills listed above are available in this project.
- Trigger rules: If the user names a skill directly or the request clearly matches its description, use that skill for the turn.
- Missing/blocked: If a listed file is missing, say so briefly and continue with the best fallback.
- Context hygiene: Read only the files needed to execute the selected skill.
- Safety: Do not bulk-load unrelated references just because a skill exists.
"""

user_agents = Path("my/AGENTS.md")
user_section = ""
if user_agents.exists():
    user_section = "\n\n---\n\n## Dodatkowe instrukcje\n\n" + user_agents.read_text(encoding="utf-8").rstrip() + "\n"

final = agents_content.rstrip() + skills_section + user_section

Path("AGENTS.md").write_text(final, encoding="utf-8")
print(f"AGENTS.md zaktualizowany - {len(skills)} skilli")
```

## Krok 3 - Utworz my/AGENTS.md (jesli nie istnieje)

```python
import sys; sys.stdout.reconfigure(encoding='utf-8')
from pathlib import Path

agents_user = Path("my/AGENTS.md")
if not agents_user.exists():
    agents_user.write_text(
        "# Moje instrukcje dla Codexa\n\n"
        "<!-- Wpisz tu dodatkowe instrukcje specyficzne dla Codexa.\n"
        "     Ten plik NIGDY nie jest nadpisywany przez aktualizacje.\n"
        "     Po edycji uruchom ponownie skill codex-setup. -->\n\n"
        "## Moje preferencje\n\n",
        encoding="utf-8"
    )
    print("Utworzono my/AGENTS.md")
else:
    print("my/AGENTS.md juz istnieje - pomijam")
```

## Krok 4 - Wygeneruj AGENTS.md ponownie po utworzeniu my/AGENTS.md

Jesli `my/AGENTS.md` zostal dopiero utworzony, uruchom ponownie Krok 2, zeby jego tresc byla dolaczona do `AGENTS.md`.

## Krok 5 - Weryfikacja

Sprawdz:

1. Czy istnieje `AGENTS.md`
2. Czy istnieje `my/AGENTS.md`
3. Czy `AGENTS.md` zawiera sekcje `## Skills`
4. Czy na liscie sa skille z `bdos/data/claude/skills/` oraz `my/skills/`

Przyklad weryfikacji:

```python
import sys; sys.stdout.reconfigure(encoding='utf-8')
from pathlib import Path

for path in [Path("AGENTS.md"), Path("my/AGENTS.md")]:
    print(f"{path}: {'OK' if path.exists() else 'BRAK'}")

agents = Path("AGENTS.md")
if agents.exists():
    content = agents.read_text(encoding="utf-8")
    print("Sekcja Skills:", "OK" if "## Skills" in content else "BRAK")
    print("Sekcja my/AGENTS:", "OK" if "Moje instrukcje dla Codexa" in content else "BRAK")
```

## Krok 6 - Podsumuj

Po wykonaniu powiedz uzytkownikowi:

> **Codex skonfigurowany.**
>
> Instrukcje projektu sa w `AGENTS.md`, a Twoje prywatne ustawienia w `my/AGENTS.md`.
> Lista skilli BDOS jest dopieta do `AGENTS.md`, wiec Codex wie, jakie skille sa dostepne i kiedy ma ich uzywac.
>
> Gdy dodasz nowy skill do `my/skills/` albo zaktualizujesz BDOS, uruchom ponownie `codex-setup`.

## Kiedy ponowic

Uruchom `codex-setup` ponownie gdy:
- dodajesz nowy skill do `my/skills/`
- robisz `bdos update` i zmienia sie `bdos/data/claude/skills/`
- edytujesz `my/AGENTS.md`
- chcesz odswiezyc `AGENTS.md` po zmianach w `CLAUDE.md`
