# zerro-ai-control

CLI-инструмент для управления транзакциями [ZenMoney](https://zenmoney.ru) через API. Один файл на Python, без внешних зависимостей.

Задуман как набор строительных блоков для AI-агента (Claude Code): пользователь даёт команду на естественном языке, агент вызывает нужные `zm`-команды.

## Установка

```bash
# Клонировать
git clone git@github.com:OdNairy/zerro-ai-control.git
cd zerro-ai-control

# Положить API-токен ZenMoney
mkdir -p data
echo "YOUR_TOKEN" > data/token

# Первая синхронизация
./zm sync
```

Требуется Python 3.7+. Внешние пакеты не нужны.

## Использование

### Чтение

```bash
./zm sync                    # инкрементальная синхронизация
./zm sync --full             # полная перезагрузка

./zm list --limit 20         # последние 20 транзакций
./zm list --from 2026-01-01 --to 2026-01-31 --payee Prisma
./zm list --untagged --type expense
./zm list --tag Продукты --account "Revolut EUR"
./zm list --min 100 --max 500

./zm show <id>               # детали транзакции (достаточно 4+ символов id)
./zm accounts                # список счетов
./zm tags                    # список категорий
```

### Изменение

Все изменения сначала сохраняются локально, на сервер отправляются командой `push`.

```bash
./zm tag-add <id> --tags "Продукты"           # добавить тег
./zm tag-set <id> --tags "Кафе" "Досуг"       # заменить теги
./zm tag-remove <id> --tags "Продукты"         # убрать тег
./zm tag-create "Новая категория"              # создать категорию
./zm tag-create "Подкат" --parent "Еда"        # создать подкатегорию

./zm edit <id> payee="Name" comment="заметка"  # редактировать поля
./zm delete <id>                               # мягкое удаление
./zm merge <id1> <id2>                         # объединить (суммы в первую)
./zm make-transfer <expense> <income>          # превратить в перевод
```

### Управление изменениями

```bash
./zm pending          # показать несохранённые изменения
./zm undo             # отменить последнее
./zm undo --all       # отменить всё
./zm push --dry-run   # предпросмотр
./zm push             # отправить на сервер
```

## Формат вывода

Весь вывод — JSON в stdout, статусные сообщения — в stderr. UUID-ы автоматически разрешаются в читаемые имена.

```json
{"id": "571e1176", "date": "2026-02-07", "type": "expense", "amount": "-23.11 EUR", "account": "Revolut EUR Joint", "payee": "Prisma Olari", "tags": ["Продукты"], "comment": null}
```

## Структура

```
zm                  — CLI-инструмент (единственный файл кода)
data/               — локальные данные (в .gitignore)
  token             — API-токен
  raw.json          — снимок данных с сервера
  state.json        — serverTimestamp для инкрементального sync
  pending.json      — журнал локальных изменений
GUIDE.md            — краткая справка для загрузки в AI-агент
CLAUDE.md           — инструкции для Claude Code
```
