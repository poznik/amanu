# amanu

A local-first meeting recorder for macOS. One SwiftPM package that is both the
application and its command line; `make app` wraps the binary in `Amanu.app`.

## Форк poznik

Этот клон — форк `gsamat/amanu` в `poznik/amanu`, заведён 2026-11-10 ради одной
правки: зависание главного потока при смене аудиомаршрута во время записи
(issue [gsamat/amanu#63](https://github.com/gsamat/amanu/issues/63)). Проект живёт
локально на MBA; Amanu на Windows остаётся официальным. Раздел ведётся по-русски,
остальной файл — upstream, его не править.

**Состояние на 2026-11-11.** Правка зависания отправлена автору:
[gsamat/amanu#65](https://github.com/gsamat/amanu/pull/65) из ветки
`fix/route-change-deadlock`; находка про «Изоляцию голоса» — комментарием в #63.
На MBA решено остаться на штатном 0.6.5 со стандартным режимом микрофона для
Amanu; сборка форка с машины убрана, локальный клон удалён. Этот форк на GitHub
живёт, пока открыт PR. Чтобы снова собрать — клонировать `poznik/amanu`, ветка
`mba`, дальше по разделу «Сборка на MBA». Правка тишины многоканального входа
(#57) в форк не вносилась — решение Николая: он на встречах в наушниках.

**Состояние на 2026-11-12.** Клон возвращён ради второй правки: дорожка микрофона
молча встаёт на границе звонков (ветка `fix/mic-stall-restart`, см. ниже). На MBA
с 14:38 того же дня стоит сборка из `mba` (build 465), штатный 0.6.5 лежит архивом
`~/.local/state/amanu-mic-watch/stock/Amanu-0.6.5-b440-stock.zip`. Разрешения для
подписи форка остались с 2026-11-11: пробная запись дала микрофон, системный звук
и название из календаря. PR автору по этой правке не отправлен.

**Ветки.** `master` повторяет `upstream/master` и своих коммитов не несёт. `mba` —
рабочая ветка форка: из неё собирается приложение, которое стоит на Маке. Правки
для upstream — отдельными ветками от `master`, в стиле коммитов upstream
(английская фраза, см. «Commit messages»), затем мерж в `mba`. Коммиты только
для форка — по-русски, «область: что сделано».

**Отличия `mba` от upstream.**

- Правка зависания (ветка `fix/route-change-deadlock`, 2026-11-11, PR в upstream
  не отправлен): наблюдатель `AVAudioEngineConfigurationChange` без `queue: .main`
  (`MicRecorder.observeConfigurationChanges`), прежний движок отпускается вне
  главного потока с ожиданием не дольше 2 с (`MicRecorder.Retiring`). Разбор —
  `docs/pitfalls.md`, «Nothing the engine posts may wait for the main thread».
  Живая проверка 2026-11-11 (сборка 450, встреча в Elseconf, AirPods Max сняты
  в 08:59:52): зависания нет — перезапуск захвата записан в `mic_restarts`,
  `record stop` сработал, `meta.json` написан, встреча расшифрована. Но цикл
  `coreaudiod` начался так же (146 % уже через 5 с, затем 200–240 %, load
  average 104) и шёл после остановки записи; `mic.caf` с момента снятия до
  остановки (26 с) не вырос ни на байт — новый движок на встроенном микрофоне
  не выдал ни одного буфера (`gap_ms` в записи о перезапуске нет), `system.caf`
  при этом рос. Вывод: зависание было следствием или спутником цикла, но не его
  единственной причиной. Замер —
  `~/projects/nikvps/mba-perf/data/2026-11-11/coreaudio-watch-fork/`
  (`cpu.tsv`, `tracks.tsv`, `events.log`, `amanu-after-fix.sample.txt`).
  Не поставлен контрольный опыт: та же встреча и снятие наушников без Amanu.
- Гипотеза Николая от 2026-11-11: цикл запускает режим микрофона «Изоляция голоса»,
  включённый в macOS для Amanu (в журнале coreaudiod при старте записи —
  `SIMULATE [me.samat.amanu]`, `semantics = voice isolation`). Второй опыт
  (09:04:57–09:06:24, режим переключён на стандартный уже во время записи) её не
  проверил: после `killall coreaudiod` звонок шёл через встроенные микрофон и
  динамики (`mic_capture.input_device`), снятие AirPods в 09:06:02 маршрут не
  меняло, `mic_restarts` пуст. Цикла не было, дорожки росли до конца, но обе —
  цифровая тишина (−91 дБ), расшифровка: «No speech was recognized». Amanu в этом
  опыте не перезапускался после `killall coreaudiod`; связано ли с этим молчание
  микрофона, не выяснено. Чистый повтор: перед стартом убедиться, что ввод и
  вывод — AirPods, режим микрофона для Amanu — стандартный.
- Третий опыт 2026-11-11 (09:12:46–09:14:21), чистый: старт на AirPods Max (HFP
  24 кГц), режим микрофона для Amanu — стандартный, для Elseconf — «Изоляция
  голоса», строк `SIMULATE [me.samat.amanu]` в журнале нет. Наушники сняты в
  09:13:40, перезапуск захвата в 09:13:41 пришёл через уведомление движка
  («input device reconfigured»), `gap_ms` 891, `mic.caf` рос до конца, `record
  stop` сработал, расшифровка и саммари готовы. coreaudiod: 7–8 % до, 20 % в
  момент смены, 10–13 % после, цикла нет. Счёт по режиму Amanu: «Изоляция
  голоса» — цикл в 3 записях из 3 (две на 0.6.5, одна на форке), стандартный —
  1 из 1 без цикла. Не выяснено: в расшифровке нет ни слова после снятия
  наушников (40 с), аудио удалено (`keep_audio` выключен) — шёл ли со
  встроенного микрофона звук, неизвестно; во втором опыте он дал цифровую тишину.
- Четвёртый опыт 2026-11-11 (09:23:40–09:25:12) — **штатный 0.6.5** из
  `/Applications`, режим микрофона для Amanu стандартный (строк `SIMULATE` нет),
  `keep_audio` включён. Наушники сняты в 09:24:54, перезапуск захвата через
  уведомление движка, `gap_ms` 899, зависания нет, запись закрылась сама
  (`call-ended`), coreaudiod 7–9 % до и 11–16 % после, цикла нет. То есть без
  «Изоляции голоса» для Amanu штатная версия блокировку не ловит (1 из 1); счёт
  по режиму: изоляция — цикл 3 из 3, стандартный — чисто 2 из 2 (форк и штатная).
- **Встроенный микрофон пишется нулями — и в штатной версии, и в форке.** В
  `audio.m4a` четвёртого опыта дорожка микрофона до снятия наушников −38 дБ в
  среднем, после перехода на «Микрофон MacBook Air» (с 76-й секунды) — ровно
  −91 дБ до конца. То же во втором опыте (вся запись) и в 10-секундной пробе форка
  без звонка. Вход в этих случаях — 3 канала 48 кГц, Float32, deinterleaved; с
  AirPods (1 канал 24 кГц) звук есть. Громкость входа в системе 29, не заглушён.
  **Причина найдена 2026-11-11** и воспроизведена без микрофона:
  `AVAudioConverter(from:to:)` для входа с дискретной раскладкой больше двух
  каналов не назначает моно-выходу ни одного входного канала (`channelMap == [-1]`)
  и пишет нули; `downmix = true` не помогает, `channelMap = [0]` даёт звук.
  Проверено на 3 и 9 каналах, с пересчётом частоты и без. Это upstream-issue
  [gsamat/amanu#57](https://github.com/gsamat/amanu/issues/57) (Flamefork,
  2026-10-26, без ответа автора; там RME Babyface, 14 каналов). Готовая правка с
  тестом — в форке Flamefork, коммит `eba46e8` от того же `c5950c5`: если карта
  пуста, берётся вход 1. В наш форк не внесена. Не проверено на устройстве, несёт
  ли голос именно первый из трёх каналов «Микрофона MacBook Air».
- Журнал приложения при проверках: `open --stderr <файл> .build/Amanu.app` —
  без этого строки `mic: …` теряются.
- Sparkle выключен: `AppUpdates.updatesAreOffInThisFork` не даёт создать
  контроллер, пункта «Проверить обновления…» в меню нет, в `Amanu-Info.plist`
  `SUEnableAutomaticChecks` = `false`. Иначе лента upstream заменит сборку
  официальной, подписанной другой командой.
- `MicRecorder.listenForDefaultInputChanges`: вложенное замыкание захватывает
  `[weak self]` само. Без этого Swift 6.2.4 (Xcode 26.3) отвергает строку с
  `sending 'self' risks causing data races`; upstream собирает более новым Swift.
- Bundle ID прежний, `me.samat.amanu`: конфиг `~/.config/amanu`, симлинк
  `~/.local/bin/amanu`, хук `on_stop` и агент `me.poznik.amanu-push` работают без
  правок. Подпись другая (Apple Development, команда `GGZ7MD65VF`), поэтому
  разрешения macOS, выданные официальной сборке, к этой не относятся.

- Перезапуск захвата при молчащем движке (ветка `fix/mic-stall-restart` от
  `upstream/master`, 2026-11-12; первым коммитом в ней — правка сборки под Swift 6.2
  из ветки зависания, без неё `master` на Xcode 26.3 не собирается). Что поймано
  2026-11-12 в 11:28 наблюдателем `nikvps/mba-perf/amanu-mic-watch.sh`: звонок
  закончился, запись продолжалась; вход движка пять раз за шесть секунд получил
  `HALC_ProxyIOContext::_StartIO(): Start failed - StartAndWaitForState returned
  error 35`, уведомления о смене конфигурации не было, `isRunning` остался `true`, а
  `mic.caf` больше не рос. Улики — `~/.local/state/amanu-mic-watch/data/2026-11-12/`,
  разбор — `RULES/hosts/mba.md`, «Amanu → Second Me». Правка: тик сессии (15 с)
  зовёт `MicRecorder.checkAlive`; если буферов нет 5 с и дольше
  (`MicRecorder.Stall`), захват пересобирается тем же путём, что при смене маршрута,
  пауза между бесплодными пересборками удваивается до 120 с. Повтор пересборки
  через 2 с теперь держит `restartPending`, чтобы тик не начал вторую поверх.
  Тесты — `MicRestartTests` (правило `Stall`) и `RecordingSessionTests` (тик
  спрашивает рекордер). Живьём не проверено: нужна запись, которая переживает
  конец звонка (ручная или две встречи встык), в `meta.json` должен появиться
  `mic_restarts`, а своя речь во втором звонке — попасть в `mic.caf`.

**Сборка на MBA.** Нужны Xcode 26.3 и `cmake` (Homebrew). Связку `elseconf-build`
разблокирует скрипт elseconf: без него `codesign` открывает окно с запросом
пароля связки и `make` висит на нём молча.

```sh
source ~/projects/elseconf/ui/shared/apple/tools/keychain.sh && build_keychain ~/projects/elseconf
make SIGN_ID="Apple Development: pnik@me.com (E2297K96J4)"
```

Первая сборка качает зависимости и собирает LocalVQE под обе архитектуры —
десятки минут на 8 ГБ; повторная — около минуты. `swift test` и `make` не
запускать одновременно. Журналы сборки удобно писать в `.build/`.

**Тесты на MBA.** Базовый прогон `swift test` 2026-11-10 (ветка `mba`, macOS 15.8,
до правки зависания): 767 тестов, падают два — «The rest follow by name, each
language offered once» (`MeetingLanguagesTests`, порядок сортировки языков) и
«Every row the status window can show sits under the row above it»
(`StatusWindowLayoutTests`, расхождение раскладки на 1–12 pt). Те же два падают
и на ветке правки от `master` (771 тест, 2026-11-11), то есть от отличий форка
не зависят; причина не выяснена. Новые падения сверять с этим списком.

**Разрешения macOS** после смены подписи выдаёт Николай: микрофон, запись
системного звука, календарь, уведомления. Если окно запроса не появляется —
сбросить прежнюю запись: `tccutil reset Microphone me.samat.amanu` (так же
`AudioCapture`, `Calendar`).

**Обновление с upstream.**

```sh
git fetch upstream
git switch master && git merge --ff-only upstream/master && git push
git switch mba && git merge master
```

**Материалы разбора ошибки** лежат вне репозитория: `~/projects/RULES/hosts/mba.md`,
раздел «coreaudiod зацикливается», и
`~/projects/nikvps/mba-perf/data/2026-11-10/coreaudio-watch/` (стек зависшего
Amanu — `amanu-hang.sample.txt`). Наблюдатель — `nikvps/mba-perf/coreaudio-watch.sh`.

## Start here

- `docs/pitfalls.md` — the constraints that are not visible from the code that
  depends on them, and the places believed correct on reasoning alone. Read it
  before changing anything about signing, the bundle, or where keys live.
- `.issues/` — tracked problems and post-mortems; `status:` in the frontmatter
  says whether each one is still open. The `rca-*` files are worth reading for
  the failure modes even where they are closed.
- `docs/releasing.md` — how to cut a release. **Read it in full before running
  `make release`.** Publishing is one command, and the failures are all in the
  parts that are not.
- `docs/specs/2026-08-18-native-macos-app-design.md` — the design,
  including the places the code deliberately went its own way.
- `docs/old-macs.md` — what runs on Intel and on older macOS, what sets each
  limit, and which of it has actually been measured. Read it before answering
  any question that starts "does amanu work on…".

## Working on it

```sh
make app                                 # build + sign .build/Amanu.app
cp -R .build/Amanu.app /Applications/    # replace the installed copy
swift test                               # full suite; screenshot tests are opt-in,
                                         # LocalVQE tests skip until `make localvqe`
```

Quit the running app before replacing `/Applications/Amanu.app` — deleting a
bundle does not stop the process using it, and the survivor answers the
doorbell the new copy is ringing.

Audio paths can only be tested by making a recording by hand. Everything else
is covered by tests, and `docs/testing/setup-window-manual-checklist.md` covers
the setup window — most of it can be driven from a script, and the items that
genuinely need a person are marked. Before changing that window, read
`docs/testing/window-shots.md`: it renders both windows to PNG in both
appearances, and every layout defect this project has found was found by
looking at one.

## Two standing decisions

- **The Unix socket is cancelled.** The interface is the distributed-
  notification doorbell (`SetupRequest`, `RecordRequest`, `SingleInstance`) plus
  the `~/.local/bin/amanu` symlink into the bundle. If something seems to need
  structured IPC, that is a new argument to make, not a plan to finish.
- **The bundle is assembled by `make app`**, not by an Xcode target. A second
  build system would have to be kept in step with the first.

## Commit messages

One prose sentence saying what changed and, where it is not obvious, why —
imperative or declarative, no `feat:`/`fix:` prefixes, no bullet lists in the
subject. A body is welcome when the reason is worth more than the diff. Read
`git log` before writing one; the voice is consistent and it is meant to be.

Comments in the code follow the same rule: they explain why, in full sentences.
