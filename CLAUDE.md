# amanu

A local-first meeting recorder for macOS. One SwiftPM package that is both the
application and its command line; `make app` wraps the binary in `Amanu.app`.

## Форк poznik

Этот клон — форк `gsamat/amanu` в `poznik/amanu`, заведён 2026-11-10 ради одной
правки: зависание главного потока при смене аудиомаршрута во время записи
(issue [gsamat/amanu#63](https://github.com/gsamat/amanu/issues/63)). Проект живёт
локально на MBA; Amanu на Windows остаётся официальным. Раздел ведётся по-русски,
остальной файл — upstream, его не править.

**Ветки.** `master` повторяет `upstream/master` и своих коммитов не несёт. `mba` —
рабочая ветка форка: из неё собирается приложение, которое стоит на Маке. Правки
для upstream — отдельными ветками от `master`, в стиле коммитов upstream
(английская фраза, см. «Commit messages»), затем мерж в `mba`. Коммиты только
для форка — по-русски, «область: что сделано».

**Отличия `mba` от upstream.**

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
