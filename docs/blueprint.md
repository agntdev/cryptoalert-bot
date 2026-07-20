# CryptoAlert Bot — Bot specification

**Archetype:** custom

**Voice:** professional and concise — write every user-facing message, button label, error, and empty state in this voice.

CryptoAlert Bot — это персональный Telegram-бот для отслеживания криптовалютных цен. Пользователи могут создавать собственные списки монет, устанавливать пороговые уведомления и уведомления о процентном изменении, настраивать тихие часы и получать ежедневные уведомления. Владелец получает агрегированную статистику использования бота.

> This is the complete contract for the bot. Implement EVERY entry point, flow, feature, integration, and edge case below. The completeness review checks the bot against this document after each build pass.

## Primary audience

- розничные пользователи Telegram
- непрофессиональные трейдеры и держатели криптовалют

## Success criteria

- пользователи могут добавлять и удалять монеты из списка наблюдения
- бот отправляет точные уведомления о ценах по пороговым значениям и процентным изменениям
- владелец получает агрегированную статистику использования бота

## Entry points

Every feature must be reachable from the bot's command/button surface (button-first; only /start and /help are slash commands).

- **/start** (command, actor: user, command: /start) — Открывает главное меню и начинает процесс настройки
- **/price** (command, actor: user, command: /price) — Показывает текущую цену указанной монеты или всех монет в списке наблюдения
- **Добавить Bitcoin** (button, actor: user, callback: add_coin:BTC) — Добавляет Bitcoin в список наблюдения
- **Добавить Ethereum** (button, actor: user, callback: add_coin:ETH) — Добавляет Ethereum в список наблюдения
- **Добавить Toncoin** (button, actor: user, callback: add_coin:TON) — Добавляет Toncoin в список наблюдения
- **Ввести пользовательский тикер** (button, actor: user, callback: custom_ticker) — Открывает меню для ввода пользовательского тикера монеты

## Flows

### onboarding
_Trigger:_ /start

1. открывает главное меню
2. предлагает добавить Bitcoin, Ethereum, Toncoin или ввести пользовательский тикер
3. спрашивает о часовом поясе, если он не установлен

_Data touched:_ User profile

### adding_removing_coins
_Trigger:_ add_coin:... или custom_ticker

1. пользователь выбирает монету из быстрых кнопок или вводит пользовательский тикер
2. бот проверяет валидность тикера
3. добавляет монету в список наблюдения или возвращает ошибку с подсказками

_Data touched:_ Watchlist item

### configuring_alerts
_Trigger:_ inline menu for watchlist item

1. открывает меню настройки уведомлений для конкретной монеты
2. пользователь выбирает тип уведомления (пороговая цена или процентное изменение)
3. настраивает параметры уведомления

_Data touched:_ Watchlist item, Alert rule types

### price_check
_Trigger:_ /price

1. пользователь запрашивает цену монеты или список
2. бот получает текущую цену из источника данных
3. отображает цену и временную метку

_Data touched:_ Watchlist item, Notification record

### morning_summary
_Trigger:_ scheduled time

1. бот проверяет время пользователя
2. отправляет краткую сводку цен для списка наблюдения
3. пользователь может отключить уведомление

_Data touched:_ User profile, Watchlist item

### quiet_hours
_Trigger:_ user sets quiet hours

1. пользователь настраивает тихие часы
2. бот подавляет уведомления в течение этого времени
3. сохраняет уведомления и отправляет их после окончания тихих часов

_Data touched:_ User profile, Notification record

### error_handling
_Trigger:_ price source failure

1. бот обнаруживает сбой источника цен
2. пытается повторно получить данные
3. отправляет информационное сообщение пользователю, если сбой сохраняется

_Data touched:_ Notification record

## Data entities

Durable data (must survive a restart) uses the toolkit's persistent store, never in-memory maps.

- **User profile** _(retention: persistent)_ — Информация о пользователе, включая Telegram ID, имя пользователя, часовой пояс, тихие часы, время утренней сводки и предпочтения уведомлений
  - fields: Telegram ID, username, timezone, quiet hours window, morning summary time, notification preferences
- **Watchlist item** _(retention: persistent)_ — Монета в списке наблюдения пользователя, включая тикер, отображаемое имя, включенные уведомления, последнее время уведомления и последнюю известную цену
  - fields: ticker, display name, enabled alerts, last-notified timestamp, last-known price
- **Alert rule types** _(retention: persistent)_ — Типы правил уведомлений: пороговая цена (направление + целевая цена) и процентное изменение (направление, процентное значение, временной интервал)
  - fields: price-threshold, percent-change
- **Notification record** _(retention: persistent)_ — Запись о сработавшем уведомлении, включая тип уведомления, монету, старую цену, новую цену, процентное изменение и временную метку
  - fields: alert type, coin, old price, new price, percent change, timestamp
- **Owner analytics** _(retention: persistent)_ — Агрегированная статистика для владельца: общее количество пользователей, активные пользователи (за 30 дней), количество срабатываний уведомлений по типам и монетам
  - fields: total users, active users, alert triggers per type, alert triggers per coin

## Integrations

- **Telegram** (required) — Bot API messaging
- **Price data provider** (required) — Получение актуальных цен криптовалют
Call external APIs against their real contract (correct endpoints, ids, params); credentials from env. Do not fake responses.

## Owner controls

- просмотр агрегированной статистики пользователей
- просмотр количества срабатываний уведомлений по типам и монетам
- получение уведомлений о системных проблемах

## Notifications

- уведомления о пороговых значениях
- уведомления о процентном изменении
- ежедневная утренняя сводка
- уведомления о системных проблемах для владельца

## Permissions & privacy

- данные каждого пользователя хранятся отдельно и не видны другим пользователям
- агрегированная статистика не включает персональные данные пользователей
- уведомления отправляются только в личные чаты пользователей

## Edge cases

- пользователь вводит несуществующий тикер или допускает опечатку
- источник цен временно недоступен
- пользователь настраивает тихие часы, когда срабатывают уведомления
- множественные уведомления срабатывают одновременно
- пользователь отключает уведомления для конкретной монеты

## Required tests

- тестирование добавления и удаления монет из списка наблюдения
- тестирование срабатывания уведомлений по пороговым значениям и процентным изменениям
- тестирование тихих часов и подавления уведомлений
- тестирование ежедневной утренней сводки
- тестирование обработки ошибок источника цен

## Assumptions

- пользователи предпочитают конфиденциальность и не хотят, чтобы их список монет был виден другим
- источник цен надежен и имеет политику повторных попыток
- владелец хочет видеть агрегированную статистику, но не персональные данные
- пользователи хотят настраивать тихие часы, чтобы избежать ночного беспокойства
- пользователи предпочитают получать уведомления о значительных изменениях, а не о незначительных колебаниях
