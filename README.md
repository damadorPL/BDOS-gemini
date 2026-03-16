# BDOS Skills

Zestaw dodatkowych skilli do projektu BDOS AI.

Repo zawiera obecnie:

- `my/skills/gemini-setup/SKILL.md` - przygotowuje konfiguracje Gemini CLI dla BDOS
- `my/skills/codex-setup/SKILL.md` - przygotowuje konfiguracje Codexa dla BDOS

Te skille nie sa samodzielna aplikacja. To gotowe pliki `SKILL.md`, ktore kopiujesz do swojego projektu BDOS AI.

## Zawartosc repo

```text
my/
  skills/
    gemini-setup/
      SKILL.md
    codex-setup/
      SKILL.md
```

## Co robi kazdy skill

### `gemini-setup`

Skill dla Gemini CLI. Jego celem jest:

- wygenerowanie lub odswiezenie `GEMINI.md`
- utworzenie i synchronizacja `.gemini/skills/`
- przygotowanie `my/GEMINI.md` na prywatne instrukcje uzytkownika

Uzywaj go, gdy chcesz, zeby Gemini CLI widzial skille BDOS natywnie przez `/skills`.

### `codex-setup`

Skill dla Codexa. Jego celem jest:

- wygenerowanie lub odswiezenie `AGENTS.md`
- utworzenie `my/AGENTS.md`
- dopiecie do `AGENTS.md` listy dostepnych skilli BDOS i zasad ich uzycia

Uzywaj go, gdy chcesz skonfigurowac projekt BDOS pod Codexa.

## Instalacja

### Wymagania

Potrzebujesz dzialajacej kopii projektu BDOS AI. Odwiedz https://bdos.ai/ aby nabyc kopie.

### Opcja 1 - recznie skopiuj skille

Skopiuj folder `my/skills/` z tego repo do katalogu `my/skills/` w swoim projekcie BDOS.

Przyklad na Windows PowerShell:

```powershell
Copy-Item -Recurse -Force `
  "C:\Users\damador\Documents\Code\BDOS-skills\my\skills\*" `
  "C:\sciezka\do\BDOS-AI\my\skills\"
```

Przyklad na macOS lub Linux:

```bash
cp -R /sciezka/do/BDOS-skills/my/skills/* /sciezka/do/BDOS-AI/my/skills/
```

Po skopiowaniu przejdz do repo BDOS AI i odswiez konfiguracje:

```bash
bdos update --regenerate
```

### Opcja 2 - sklonuj to repo obok BDOS AI i kopiuj z niego

Jesli chcesz trzymac skille w osobnym repo i aktualizowac je niezaleznie:

```bash
git clone https://github.com/TWOJ-LOGIN/BDOS-skills.git
```

Kopie BDOS AI mozna nabyc na https://bdos.ai/

Nastepnie kopiuj wybrane skille z `BDOS-skills/my/skills/` do `BDOS-AI/my/skills/` i uruchamiaj:

```bash
cd BDOS-AI
bdos update --regenerate
```

## Aktywacja po instalacji

Samo skopiowanie plikow nie wystarczy. Po dodaniu skilli do BDOS uruchom w repo projektu:

```bash
bdos update --regenerate
```

Potem uzyj odpowiedniego skilla:

- dla Gemini CLI: uruchom `gemini-setup`
- dla Codexa: uruchom `codex-setup`

## Jak uzywac

Po instalacji mozesz poprosic agenta o konfiguracje odpowiedniego srodowiska.

Przyklady:

```text
Uruchom gemini-setup
```

```text
Uruchom codex-setup
```

Jesli agent obsluguje jawne wywolanie przez sciezke, mozesz wskazac plik bezposrednio:

```text
@my/skills/gemini-setup/SKILL.md
```

```text
@my/skills/codex-setup/SKILL.md
```

## Aktualizacja

Gdy zmienisz zawartosc ktoregos skilla:

1. Podmien pliki w `BDOS-AI/my/skills/`
2. Uruchom `bdos update --regenerate`
3. Ponownie uruchom `gemini-setup` albo `codex-setup`, zaleznie od klienta

## Uwagi

- Repo nie zawiera calego BDOS AI, tylko dodatkowe skille
- Skille sa przeznaczone do wgrania do `my/skills/` w istniejacej instancji BDOS
- `gemini-setup` i `codex-setup` sa skillami konfiguracyjnymi, a nie skillami do analizy kampanii
