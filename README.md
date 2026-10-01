# Databáza úloh

Backendová aplikácia na správu, vyhľadávanie a kategorizáciu matematických a logických úloh pre vzdelávacie aktivity a semináre organizované združením **STROM**.

Projekt je postavený na frameworku **Django** a využíva **Django REST Framework** na poskytovanie REST API. Dáta sú modelované tak, aby bolo možné prepájať úlohy so seminármi, typmi aktivít, náročnosťou, tagmi, médiami a konkrétnymi aktivitami.

> **Poznámka:** Tento README bolo vytvorené s pomocou AI na základe aktuálnej štruktúry a zdrojového kódu repozitára. Popisuje stav projektu v commite [`c084752`](https://github.com/ZdruzenieSTROM/databaza-uloh/tree/c08475251499c7c9d692d45cc63dc08820d39a94).

## Obsah

- [Prehľad projektu](#prehľad-projektu)
- [Použité technológie](#použité-technológie)
- [Štruktúra repozitára](#štruktúra-repozitára)
- [Dátový model](#dátový-model)
- [REST API](#rest-api)
- [Lokálne spustenie](#lokálne-spustenie)
- [Testovanie](#testovanie)
- [ Django administrácia](#django-administrácia)
- [Konfigurácia a bezpečnosť](#konfigurácia-a-bezpečnosť)
- [Známe nedostatky](#známe-nedostatky)
- [Ďalší rozvoj](#ďalší-rozvoj)
- [Licencia](#licencia)

## Prehľad projektu

Aplikácia slúži ako centrálne API pre databázu úloh. Umožňuje uchovávať:

- samotné zadanie úlohy,
- výsledok a riešenie,
- typ úlohy a jej popis,
- priradenie úlohy ku konkrétnej aktivite,
- náročnosť úlohy v rámci aktivity,
- tematické tagy,
- obrázkové alebo iné priložené médiá,
- informáciu o seminári a type aktivity.

API je navrhnuté najmä na čítanie dát. Väčšina viewsetov dedí z `ReadOnlyModelViewSet`, pričom vytváranie seminárov a úloh je v aktuálnej implementácii podporované samostatnou úpravou metódy `create`.

## Použité technológie

- **Python**
- **Django 4.2.5** – webový framework
- **Django REST Framework 3.14.0** – REST API
- **SQLite** – predvolená databáza pre lokálny vývoj
- **Django ORM** – práca s databázovými modelmi
- **Django Filter** – filtrovanie záznamov podľa query parametrov
- **django-typomatic** – generovanie TypeScript rozhraní zo serializerov

Priame verzie základných balíkov sú uvedené v súbore [`requirements.txt`](requirements.txt).

## Štruktúra repozitára

```text
.
├── manage.py
├── requirements.txt
├── README.md
├── problem_database/
│   ├── __init__.py
│   ├── asgi.py
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
└── problems/
    ├── __init__.py
    ├── admin.py
    ├── apps.py
    ├── migrations/
    │   └── __init__.py
    ├── models.py
    ├── serializers.py
    ├── tests.py
    ├── urls.py
    └── views.py
```

### Hlavné súbory

| Súbor | Účel |
| --- | --- |
| `manage.py` | Vstupný bod pre Django príkazy |
| `problem_database/settings.py` | Konfigurácia Django projektu a databázy |
| `problem_database/urls.py` | Koreňová konfigurácia URL adries |
| `problems/models.py` | Definícia databázových modelov |
| `problems/serializers.py` | Serializácia modelov do JSON |
| `problems/views.py` | API viewsety a filtrovanie |
| `problems/urls.py` | Router a URL adresy API |
| `problems/tests.py` | Testy modelov a API endpointov |
| `requirements.txt` | Python závislosti projektu |

## Dátový model

### Seminár – `Seminar`

Reprezentuje vzdelávací seminár, napríklad Malynár, Matik alebo Strom.

- `name` – názov seminára

### Typ aktivity – `ActivityType`

Určuje druh aktivity patriaci ku konkrétnemu semináru.

- `name` – názov typu aktivity
- `seminar` – väzba na seminár

### Aktivita – `Activity`

Konkrétna udalosť alebo realizácia aktivity.

- `date` – dátum aktivity vo formáte `YYYY-MM-DD`
- `activity_type` – typ aktivity
- `description` – popis aktivity
- `soft_deleted` – príznak logického vymazania

### Náročnosť – `Difficulty`

Úroveň náročnosti úlohy v rámci typu aktivity.

- `name` – názov náročnosti
- `activity_type` – súvisiaci typ aktivity

### Typ úlohy – `ProblemType`

Kategória alebo typ príkladu.

- `seminar` – seminár, ku ktorému typ patrí
- `name` – názov typu
- `description` – popis typu

### Úloha – `Problem`

Základný objekt databázy.

- `problem` – text zadania
- `result` – výsledok úlohy
- `solution` – riešenie
- `problem_type` – väzba many-to-many na typy úloh
- `soft_deleted` – príznak logického vymazania

### Médium – `Media`

Priložený súbor alebo obrázok súvisiaci s úlohou.

- `data` – súbor uložený cez `ImageField`
- `problem` – súvisiaca úloha
- `soft_deleted` – príznak logického vymazania

### Väzba úlohy na aktivitu – `ProblemActivity`

Prepája úlohu s aktivitou a určuje jej náročnosť.

- `problem` – úloha
- `activity` – aktivita
- `difficulty` – náročnosť

### Tag – `Tag`

Voľná tematická značka, napríklad `Indukcia` alebo `Aritmetika`.

- `name` – názov tagu

### Väzba úlohy na tag – `ProblemTag`

Prepája úlohu s jedným tagom.

- `problem` – úloha
- `tag` – tag

## REST API

Router v `problems/urls.py` definuje tieto zdroje:

| Zdroj | Endpoint | Podporované filtrovanie |
| --- | --- | --- |
| Semináre | `/problem-database/seminars/` | – |
| Typy aktivít | `/problem-database/activity-types/` | `?seminar=<id>` |
| Aktivity | `/problem-database/activities/` | `?activity_type=<id>` |
| Náročnosti | `/problem-database/difficulty/` | – |
| Úlohy | `/problem-database/problems/` | `?problem_type=<id>`, `?search=<text>` |
| Médiá | `/problem-database/media/` | `?problem=<id>` |
| Úlohy a aktivity | `/problem-database/problem-activities/` | – |
| Typy úloh | `/problem-database/problem-types/` | – |
| Tagy | `/problem-database/tags/` | – |
| Úlohy a tagy | `/problem-database/problem-tags/` | `?problem=<id>`, `?tag=<id>` |

Konkrétny endpoint podporuje štandardné DRF akcie pre čítanie, napríklad:

```bash
curl http://127.0.0.1:8000/problem-database/problems/
curl "http://127.0.0.1:8000/problem-database/problems/?search=geometria"
curl "http://127.0.0.1:8000/problem-database/activity-types/?seminar=1"
```

Serializeri používajú `fields = '__all__'`, takže API vracia všetky polia príslušného modelu.

## Lokálne spustenie

### 1. Klonovanie repozitára

```bash
git clone https://github.com/ZdruzenieSTROM/databaza-uloh.git
cd databaza-uloh
```

### 2. Vytvorenie virtuálneho prostredia

Linux/macOS:

```bash
python3 -m venv .venv
source .venv/bin/activate
```

Windows PowerShell:

```powershell
py -m venv .venv
.venv\Scripts\Activate.ps1
```

### 3. Inštalácia závislostí

```bash
python -m pip install --upgrade pip
pip install -r requirements.txt
```

> Aktuálny zdrojový kód importuje aj `django-filter` a `django-typomatic`. Ak nie sú dostupné v prostredí, doinštalujte ich príkazom `pip install django-filter django-typomatic` a následne ich doplňte do `requirements.txt`.

### 4. Inicializácia databázy

```bash
python manage.py makemigrations
python manage.py migrate
```

### 5. Vytvorenie administrátora

```bash
python manage.py createsuperuser
```

### 6. Spustenie vývojového servera

```bash
python manage.py runserver
```

Server bude štandardne dostupný na adrese:

```text
http://127.0.0.1:8000/
```

Administrácia je dostupná na:

```text
http://127.0.0.1:8000/admin/
```

## Testovanie

Testy je možné spustiť pomocou Django test runnera:

```bash
python manage.py test
```

Testy v `problems/tests.py` overujú najmä:

- vytváranie modelov `Seminar`, `ActivityType`, `Activity` a `ProblemTag`,
- textovú reprezentáciu modelov,
- načítanie zoznamu seminárov,
- načítanie a filtrovanie typov aktivít,
- načítanie a filtrovanie väzieb medzi úlohami a tagmi.

## Django administrácia

Projekt obsahuje základnú integráciu s Django administráciou. V aktuálnom stave je vlastná konfigurácia vytvorená pre `ActivityType`, kde sa v zozname zobrazuje názov a príslušný seminár.

Po vytvorení superusera je možné administráciu rozšíriť registráciou ďalších modelov, napríklad:

- `Problem`,
- `ProblemType`,
- `Activity`,
- `Difficulty`,
- `Tag`.

## Konfigurácia a bezpečnosť

Aktuálne nastavenia sú vhodné iba na lokálny vývoj. Pred nasadením do produkcie je potrebné minimálne:

1. presunúť `SECRET_KEY` do environmentálnej premennej,
2. nastaviť `DEBUG = False`,
3. nakonfigurovať `ALLOWED_HOSTS`,
4. použiť produkčnú databázu,
5. nastaviť statické a používateľské súbory,
6. nakonfigurovať HTTPS a bezpečnostné Django nastavenia,
7. doplniť autentifikáciu a oprávnenia pre zapisovacie operácie.

Súbor `settings.py` obsahuje vývojový secret key, preto by sa nemal používať v produkcii ani zdieľať ako dôveryhodný tajný údaj.

## Známe nedostatky

README vychádza priamo z aktuálneho stavu repozitára. Pred nasadením alebo ďalším rozvojom odporúčame skontrolovať najmä:

- `problems` nie je v aktuálnom `INSTALLED_APPS`,
- koreňové URL konfigurácie zatiaľ explicitne nepripájajú `problems.urls`,
- v `problems/urls.py` je route `media` zaregistrovaná s `ProblemViewSet` namiesto `MediaViewSet`,
- závislosti `django-filter` a `django-typomatic` používané v kóde nie sú uvedené v `requirements.txt`,
- v repozitári zatiaľ nie sú databázové migrácie obsahujúce modely aplikácie,
- `Media` používa `ImageField`, preto bude pravdepodobne potrebné doplniť `Pillow`,
- nie všetky modely sú zaregistrované v administrácii,
- niektoré viewsety sú označené ako read-only, hoci vybrané triedy implementujú vlastné vytváranie záznamov,
- logické mazanie je reprezentované poľom `soft_deleted`, ale aktuálne querysety ho automaticky nefiltrujú.

Tieto body sú dokumentačné upozornenia, nie automaticky vykonané opravy.

## Ďalší rozvoj

Možné ďalšie kroky rozvoja projektu:

- doplniť a skontrolovať migrácie databázy,
- zjednotiť konfiguráciu aplikácie a URL routingu,
- doplniť chýbajúce závislosti do `requirements.txt`,
- pridať autentifikáciu, autorizáciu a používateľské roly,
- dokončiť CRUD operácie pre administrátorov,
- pridať stránkovanie, ordering a pokročilé filtrovanie,
- automaticky vylúčiť soft-deleted záznamy z API,
- rozšíriť testy o vytváranie, aktualizáciu, mazanie a validačné chyby,
- pridať OpenAPI/Swagger dokumentáciu,
- pripraviť Docker konfiguráciu a CI workflow,
- oddeliť vývojové a produkčné nastavenia,
- doplniť správu uploadovaných médií.

## Licencia

V repozitári sa aktuálne nenachádza licenčný súbor. Podmienky používania, kopírovania a úprav projektu preto nie sú v tomto momente formálne definované.

Pred verejným alebo produkčným použitím odporúčame doplniť vhodný licenčný súbor, napríklad `LICENSE`, podľa rozhodnutia vlastníka projektu.

## Kontakt a príspevky

Projekt spravuje organizácia [ZdruzenieSTROM](https://github.com/ZdruzenieSTROM). Návrhy na zlepšenie, opravy a nové funkcie je možné riešiť prostredníctvom GitHub Issues a Pull Requestov.
