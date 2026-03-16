---
description: Setup Codex for BDOS by installing BDOS skills into .agents/skills in the project root so Codex can discover them natively from .agents/skills/**/SKILL.md
---

# Setup Codex dla BDOS

Skill robi trzy rzeczy:
1. Kopiuje lub odswieza skille BDOS do `.agents/skills/` w root katalogu projektu
2. Tworzy lub odswieza `AGENTS.md` bez recznie dopinanej listy skilli
3. Tworzy `my/AGENTS.md` - miejsce na prywatne instrukcje uzytkownika dla Codexa

## Jak dziala konfiguracja w Codex

Codex ma natywny system skilli oparty o katalog z plikami `SKILL.md`. W tej konfiguracji skille BDOS sa instalowane do `.agents/skills/<name>/SKILL.md` w root katalogu projektu, zeby Codex mogl je wykrywac natywnie bez dopisywania recznej listy do `AGENTS.md`.

| | Claude Code | Gemini CLI | Codex |
|--|-------------|-----------|-------|
| Glowny plik instrukcji | `CLAUDE.md` | `GEMINI.md` | `AGENTS.md` |
| Skille projektu | `my/skills/<name>/SKILL.md` | `.gemini/skills/<name>/SKILL.md` | `.agents/skills/<name>/SKILL.md` |
| Aktywacja | Auto po `description` | Auto po `description` | Auto po `description` lub przez `$skill` |
| Lista skilli | wbudowana | `/skills` | `/skills` |

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

## Krok 2 - Zainstaluj skille do .agents/skills

Skopiuj wszystkie skille BDOS do natywnego katalogu Codexa: `.agents/skills/` w katalogu projektu. Kazdy skill powinien trafic do osobnego folderu z plikiem `SKILL.md`.

```python
import sys; sys.stdout.reconfigure(encoding='utf-8')
import shutil
from pathlib import Path

skill_dirs = [
    Path("bdos/data/claude/skills"),
    Path("my/skills"),
]

agents_skills_dir = Path(".agents/skills")
agents_skills_dir.mkdir(parents=True, exist_ok=True)

copied = []
for base in skill_dirs:
    if not base.exists():
        continue
    for src_dir in sorted(base.iterdir()):
        if not src_dir.is_dir():
            continue

        skill_file = src_dir / "SKILL.md"
        if not skill_file.exists():
            continue

        dst_dir = agents_skills_dir / src_dir.name
        dst_dir.mkdir(parents=True, exist_ok=True)

        for item in src_dir.iterdir():
            dst_path = dst_dir / item.name
            if item.is_dir():
                if dst_path.exists():
                    shutil.rmtree(dst_path)
                shutil.copytree(item, dst_path)
            else:
                shutil.copy2(item, dst_path)

        copied.append(dst_dir)

print(f"Skopiowano: {len(copied)} skilli -> {agents_skills_dir}")
for path in copied:
    print(f" - {path}")
```

## Krok 3 - Wygeneruj AGENTS.md

`AGENTS.md` nadal jest glownym plikiem instrukcji projektu dla Codexa, ale nie trzeba juz dopinac do niego recznej listy skilli. Wystarczy informacja, ze prywatne instrukcje sa w `my/AGENTS.md`, a skille sa wykrywane natywnie z `.agents/skills/`.

```python
import sys; sys.stdout.reconfigure(encoding='utf-8')
from pathlib import Path

claude_md = Path("CLAUDE.md").read_text(encoding="utf-8")

agents_content = claude_md.replace(
    "Edytuj: my/CLAUDE.md (dodatkowe instrukcje)",
    "Edytuj: my/AGENTS.md (dodatkowe instrukcje dla Codexa)"
)

native_skills_section = """

---

## Skills

Codex uzywa natywnych skilli z `.agents/skills/**/SKILL.md`.

- Discovery: Codex wykrywa skille natywnie z katalogu `.agents/skills/` w root katalogu projektu.
- Trigger rules: Skill uruchamia sie po dopasowaniu `description` albo po jawnym wywolaniu przez `$skill`.
- Maintenance: Po dodaniu lub zmianie skilla uruchom ponownie `codex-setup`, zeby zsynchronizowac katalog `.agents/skills/`.
- Safety: Nie dopisuj recznie listy skilli do `AGENTS.md`, bo zrodlem prawdy jest katalog `.agents/skills/`.
"""

user_agents = Path("my/AGENTS.md")
user_section = ""
if user_agents.exists():
    user_section = "\n\n---\n\n## Dodatkowe instrukcje\n\n" + user_agents.read_text(encoding="utf-8").rstrip() + "\n"

final = agents_content.rstrip() + native_skills_section + user_section
Path("AGENTS.md").write_text(final, encoding="utf-8")
print("AGENTS.md zaktualizowany")
```

## Krok 4 - Utworz my/AGENTS.md (jesli nie istnieje)

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

## Krok 5 - Wygeneruj AGENTS.md ponownie po utworzeniu my/AGENTS.md

Jesli `my/AGENTS.md` zostal dopiero utworzony, uruchom ponownie Krok 3, zeby jego tresc byla dolaczona do `AGENTS.md`.

## Krok 6 - Weryfikacja

Sprawdz:

1. Czy istnieje `.agents/skills/`
2. Czy w `.agents/skills/` sa foldery skilli z plikiem `SKILL.md`
3. Czy istnieje `AGENTS.md`
4. Czy istnieje `my/AGENTS.md`
5. Czy `AGENTS.md` zawiera informacje o natywnych skillach z `.agents/skills/`

Przyklad weryfikacji:

```python
import sys; sys.stdout.reconfigure(encoding='utf-8')
from pathlib import Path

agents_skills_dir = Path(".agents/skills")
print(f"{agents_skills_dir}: {'OK' if agents_skills_dir.exists() else 'BRAK'}")

if agents_skills_dir.exists():
    for skill_dir in sorted(agents_skills_dir.iterdir()):
        if skill_dir.is_dir():
            print(f"{skill_dir / 'SKILL.md'}: {'OK' if (skill_dir / 'SKILL.md').exists() else 'BRAK'}")

for path in [Path("AGENTS.md"), Path("my/AGENTS.md")]:
    print(f"{path}: {'OK' if path.exists() else 'BRAK'}")
```

## Krok 7 - Podsumuj

Po wykonaniu powiedz uzytkownikowi:

> **Codex skonfigurowany.**
>
> Instrukcje projektu sa w `AGENTS.md`, a Twoje prywatne ustawienia w `my/AGENTS.md`.
> Skille BDOS zostaly zsynchronizowane do `.agents/skills/`, wiec Codex moze wykrywac je natywnie z `.agents/skills/**/SKILL.md`.
>
> Gdy dodasz nowy skill do `my/skills/` albo zaktualizujesz BDOS, uruchom ponownie `codex-setup`.

## Kiedy ponowic

Uruchom `codex-setup` ponownie gdy:
- dodajesz nowy skill do `my/skills/`
- robisz `bdos update` i zmienia sie `bdos/data/claude/skills/`
- edytujesz `my/AGENTS.md`
- chcesz odswiezyc `AGENTS.md` po zmianach w `CLAUDE.md`
