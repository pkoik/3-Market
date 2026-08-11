# 3 — Market · Rive integration handoff

Дата передачи: 11 августа 2026

Репозиторий: https://github.com/pkoik/3-Market

## Состав поставки

| Файл | Назначение |
| --- | --- |
| `index.html` | Самодостаточная демонстрационная страница с placeholder перед секцией, адаптивной Rive-интеграцией и встроенными данными `.riv` |
| `3-Market_v01.riv` | Исходный Rive-файл для дальнейшей разработки и повторного экспорта |
| `HANDOFF.md` | Этот документ |

В текущем `index.html` данные Rive встроены в HTML как base64. Поэтому для запуска демо отдельная загрузка `3-Market_v01.riv` не требуется. Исходный `.riv` оставлен рядом для разработки.

## Быстрый запуск

Демо можно открыть напрямую через `index.html` или разместить как статический сайт.

Настройки Netlify:

- production branch: `main`;
- build command: не требуется;
- publish directory: корень репозитория (`/`);
- entry point: `/index.html`.

Rive WebGL2 runtime загружается с CDN:

```text
https://unpkg.com/@rive-app/webgl2@2.38.4
```

Для работы страницы необходимо интернет-соединение к CDN. Версия runtime зафиксирована и не должна обновляться без повторной проверки анимации, View Model и размытия.

## Что является рабочей секцией

- `.page-placeholder` — только имитация контента страницы перед секцией. В продукте его нужно удалить или заменить реальным контентом.
- `.market-section` — контейнер секции.
- `.rive-frame` — видимая область дизайна с базовым размером `1136 × 564`.
- `#rive-canvas` — WebGL2 canvas с Rive-анимацией.

## Rive contract

Не переименовывать эти сущности без синхронных изменений в Rive-файле и коде:

| Параметр | Значение |
| --- | --- |
| Rive file | `3-Market_v01.riv` |
| Artboard | `3 - Market` |
| State Machine | `State Machine 1` |
| View Model | `MarketVM` |
| View Model Instance | `Instance` |
| Entrance trigger | `isIn` |
| Renderer | WebGL2 |
| Runtime | `@rive-app/webgl2@2.38.4` |

State Machine запускается с `autoplay: true`. View Model подключается через `autoBind: true`; при необходимости код явно находит `MarketVM`, выбирает экземпляр `Instance` и вызывает `bindViewModelInstance()`.

## Scroll trigger

Входная анимация запускается один раз, когда в viewport стало видно не менее 18% блока `.rive-frame`:

```js
const TRIGGER_THRESHOLD = 0.18;
```

`IntersectionObserver` вызывает trigger `MarketVM.isIn`. Для браузеров без `IntersectionObserver` предусмотрен fallback через пассивные обработчики `scroll` и `resize`.

Условия запуска защищены от гонки: trigger срабатывает только когда одновременно загружен Rive-файл, подключён View Model и секция достигла порога видимости.

## Адаптивность

Базовый размер макета:

```text
1136 × 564 px при viewport 1440 × 1080 px
```

`.rive-frame` сохраняет пропорцию `1136 / 564` и ограничен шириной viewport с внешними отступами:

- desktop/tablet: по 24 px;
- до 640 px: по 16 px;
- минимальная поддерживаемая ширина страницы: 320 px.

Масштаб рассчитывается пропорционально:

```js
scale = frame.clientWidth / 1136;
```

Проверенные ориентиры:

| Ширина viewport | Масштаб |
| --- | --- |
| 1440 px | 100% |
| 900 px | 75% |
| 600 px | 50% |

После изменения размеров вызывается `resizeDrawingSurfaceToCanvas()`. `ResizeObserver` отслеживает контейнер, а отдельный listener — изменение `devicePixelRatio`, чтобы canvas оставался чётким на Retina/HiDPI-дисплеях.

## Выход графики за границы блока

В Rive персонаж намеренно выходит за базовую область `1136 × 564`. Его размер менять не нужно.

Для сохранения этого поведения:

- canvas имеет рабочий размер `1238 × 620`;
- `.market-section` и `.rive-frame` используют `overflow: visible`;
- Rive выровнен по `TopLeft`;
- `body` использует только `overflow-x: hidden`, чтобы выступающий объект не создавал горизонтальный скролл;
- нельзя добавлять `overflow: hidden`, `clip-path` или `contain: paint` на `.market-section`, `.rive-frame` или их продуктовые аналоги.

`contain: layout size` на `.rive-frame` допустим: он не обрезает отрисовку.

## WebGL2 и blur

Используется пакет `@rive-app/webgl2`, поскольку Rive-файл содержит размытия. После загрузки код дополнительно проверяет наличие контекста `canvas.getContext("webgl2")`.

Если WebGL2 недоступен, показывается явное сообщение об ошибке. Автоматического перехода на Canvas renderer нет, поскольку он может визуально отличаться и некорректно отображать blur.

## Интеграция в продуктовую страницу

1. Перенести разметку `.market-section` и `.rive-frame` из `index.html`.
2. Перенести связанные стили, сохранив адаптивную пропорцию и `overflow: visible`.
3. Подключить зафиксированный WebGL2 runtime.
4. Создать Rive instance с указанными Artboard, State Machine и `Fit.Layout` / `TopLeft`.
5. Подключить `MarketVM.Instance` и получить trigger `isIn`.
6. Настроить `IntersectionObserver` с порогом `0.18`.
7. Сохранить `ResizeObserver`, DPR watcher и вызов `resizeDrawingSurfaceToCanvas()`.
8. При размонтировании компонента отключить observers/listeners и вызвать `player.cleanup()`.

В продукте `.riv` можно загружать отдельным статическим asset вместо base64. Для этого заменить запуск из встроенного buffer на `src: "./3-Market_v01.riv"`, сохранив остальную конфигурацию без изменений.

## Диагностика

Добавьте `?dev=1` к URL, чтобы показать компактный индикатор состояния. В консоль после успешной загрузки выводится объект `[Xsolla Market Rive]` с именами Artboard, State Machine, View Model, trigger и renderer.

## Проверено

- загрузка встроенного Rive-файла без ручного выбора файла;
- WebGL2 renderer и отображение blur;
- подключение `MarketVM.Instance`;
- однократный запуск `MarketVM.isIn` при достижении секции;
- пропорциональное уменьшение на desktop, tablet и mobile;
- Retina/HiDPI resize;
- отсутствие обрезания персонажа за границами базового блока;
- очистка Rive instance, observers и listeners при закрытии страницы;
- отсутствие ошибок в консоли в проверенной конфигурации.

## Критерии приёмки после переноса

- при viewport `1440 × 1080` основной блок визуально равен `1136 × 564`;
- при уменьшении окна вся композиция уменьшается пропорционально и не появляется горизонтальный скролл;
- выступающий персонаж остаётся исходного относительного размера и не обрезается;
- анимация не стартует до появления секции и запускается при видимости около 18%;
- trigger срабатывает только один раз за загрузку страницы;
- blur соответствует Rive Preview;
- в консоли нет load, WebGL2, View Model или trigger errors.
