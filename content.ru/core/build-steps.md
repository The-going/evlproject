---
menuTitle: "Компиляция EVL"
title: "Сборка EVL"
weight: 1
original_path: "content/core/build-steps.md"
original_hash: "1e166ad"
---

### Построение EVL из исходного кода

{{% mixedgrid src="/images/overview-build-process.png" %}}

**Процесс сборки.** Сборка EVL из исходного кода - это процесс в два этапа:
мы должны собрать ядро Linux включив EVL ядро, и библиотеку реализующую API
пользователя для этого ядра - т.е. [libevl]({{< relref
"core/user-api/_index.md" >}}) - с использованием соответствующего инструментария.
Эти шаги могут выполняться в любом порядке. Результатом этого процесса является:

- образ ядра Linux с изображением [Dovetail]({{< relref
  "dovetail/_index.md" >}}) и [ядром EVL]({{< relref
  "core/_index.md" >}}) поверх него.

- общая библиотека<sup>*</sup> `libevl.so`, которая позволяет приложениям
  запрашивать службы из ядра EVL, а также несколько [основных утилит]({{< relref
  "core/commands.md" >}}) и [тестовых программ]({{< relref "core/testing.md" >}}).

  <sup>*</sup> Также создается статический архив `libevl.a`.

{{% /mixedgrid %}}

### Получение источников

Источники EVL хранятся в репозиториях [GIT](https://git-scm.com). В качестве
предварительного шага вы можете взглянуть на [процесс разработки EVL]({{< relref
"devprocess.md" >}}), чтобы определить, какие ветви GIT могут вас заинтересовать
в этих репозиториях:

- Дерево ядра с ядром EVL:

  * `git@git.xenomai.org:Xenomai/xenomai4/linux-evl.git`
  * https://git.xenomai.org/xenomai4/linux-evl.git

- Дерево libevl, которое предоставляет пользовательский интерфейс для ядра:

  * `git@git.xenomai.org:Xenomai/xenomai4/libevl.git`
  * https://git.xenomai.org/xenomai4/libevl.git

### Другие предварительные условия {#building-evl-prereq}

В дополнение к исходному коду нам понадобится:

- набор инструментов GCC для целевой архитектуры процессора.

- заголовки UAPI из целевого ядра Linux соответствуют ядру EVL. Каждый файл UAPI
экспортирует набор определений и типов интерфейсов, которые совместно
используются с `libevl.so` работающей в пространстве пользователя, чтобы последняя
могла отправлять хорошо сформированные системные вызовы первому. Другими словами,
при сборке `libevl.so`, нам нужен доступ к содержимому `include/uapi/asm/` и
`include/uapi/evl/` из исходного дерева ядра, которое содержит ядро EVL,
которое будет обрабатывать системные вызовы.

{{% notice warning %}}
libevl полагается на поддержку локального хранилища потоков (TLS), которая может
быть нарушена в некоторых устаревших цепочках инструментов (ARM).
Обязательно используйте текущий.
{{% /notice %}}

### Сборка ядра {#building-evl-core}

Как только появится ваш любимый инструмент настройки ядра, вы должны увидеть
блок конфигурации EVL где-то внутри меню **Общая настройка**. Этот блок
конфигурации выглядит следующим образом:

![Alt text](/images/core_xconfig.png "Конфигурирование ядра EVL")

Включения `CONFIG_EVL` должно быть достаточно для начала работы, значения
по умолчанию для других параметров EVL безопасны в использовании.
Вы должны убедиться, что также включены `CONFIG_EVL_LATMUS` и `CONFIG_EVL_HECTIC`;
это драйверы, необходимые для запуска утилит `latmus` и `hectic`,
доступных с `libevl`, которые измеряют задержку и проверяют правильность
переключения контекста.

{{% notice tip %}}
Если вы не знакомы с созданием ядер, [этот документ](https://kernelnewbies.org/KernelBuild)
может помочь. Если вы сталкиваетесь с препятствиями, возникающими непосредственно
в дереве исходных текстов ядра, как показано в упомянутом документе,
вы можете проверить, может ли работать построение вне дерева,
поскольку именно так разработчики Dovetail/EVL обычно перестраивают ядра.
Если что-то пойдет не так при построении внутри дерева или вне дерева,
пожалуйста, отправьте заметку в [список рассылки
EVL](https://xenomai.org/mailman/listinfo/xenomai/) с соответствующей информацией.
{{% /notice %}}

#### Все основные параметры конфигурации {#core-kconfig}

<div>
<style>
#kconfig {
       width: 100%;
}
#kconfig th {
       text-align: center;
}
#kconfig td {
       text-align: left;
}
#kconfig tr:nth-child(even) {
       background-color: #f2f2f2;
}
#kconfig td:nth-child(2) {
       text-align: center;
}
</style>

<table id="kconfig">
  <col width="10%">
  <col width="5%">
  <col width="85%">
  <tr>
    <th>Название символа</th>
    <th>По умолчанию</th>
    <th>Назначение</th>
  </tr>
  <tr>
    <td><a href="{{% relref "/core/" %}}" target="_blank">CONFIG_EVL</a></td>
    <td>N</td>
    <td>Включите ядро EVL</td>
  </tr>
  <tr>
    <td><a href="{{% relref "/core/user-api/scheduling/#SCHED_QUOTA" %}}" target="_blank">CONFIG_EVL_SCHED_QUOTA</a></td>
    <td>N</td>
    <td>Включите политику планирования на основе квот</td>
  </tr>
  <tr>
    <td><a href="{{% relref "/core/user-api/scheduling/#SCHED_TP" %}}" target="_blank">CONFIG_EVL_SCHED_TP</a></td>
    <td>N</td>
    <td>Включите политику планирования с разделением по времени</td>
  </tr>
  <tr>
    <td><a href="{{% relref "/core/user-api/scheduling/#SCHED_TP" %}}" target="_blank">CONFIG_EVL_SCHED_TP_NR_PART</a></td>
    <td>N</td>
    <td>Количество временных разделов для CONFIG_EVL_SCHED_TP</td>
  </tr>
  <tr>
    <td>CONFIG_EVL_HIGH_PERCPU_CONCURRENCY</td>
    <td>N</td>
    <td>Оптимизирует реализацию для приложений с множеством потоков реального времени, работающих одновременно на любом заданном ядре процессора</td>
  </tr>
  <tr>
    <td><a href="{{% relref "/core/user-api/thread/#thread-stats" %}}" target="_blank">CONFIG_EVL_RUNSTATS</a></td>
    <td>Y</td>
    <td>Сбор статистики времени выполнения о потоках</td>
  </tr>
  <tr>
    <td>CONFIG_EVL_COREMEM_SIZE</td>
    <td>2048</td>
    <td>Размер кучи основной памяти (в килобайтах)</td>
  </tr>
  <tr>
    <td><a href="{{% relref "/core/user-api/thread/" %}}" target="_blank">CONFIG_EVL_NR_THREADS</a></td>
    <td>256</td>
    <td>Максимальное количество потоков EVL</td>
  </tr>
  <tr>
    <td>CONFIG_EVL_NR_MONITORS</td>
    <td>512</td>
    <td>Максимальное количество мониторов EVL (т.е. мьютексы + семафоры + флаги + события)</td>
  </tr>
  <tr>
    <td><a href="{{% relref "/core/user-api/clock/" %}}" target="_blank">CONFIG_EVL_NR_CLOCKS</a></td>
    <td>8</td>
    <td>Максимальное количество часов EVL</td>
  </tr>
  <tr>
    <td><a href="{{% relref "/core/user-api/xbuf/" %}}" target="_blank">CONFIG_EVL_NR_XBUFS</a></td>
    <td>16</td>
    <td>Максимальное количество кросс-буферов EVL</td>
  </tr>
  <tr>
    <td><a href="{{% relref "/core/user-api/proxy/" %}}" target="_blank">CONFIG_EVL_NR_PROXIES</a></td>
    <td>64</td>
    <td>Максимальное количество прокси-серверов EVL</td>
  </tr>
  <tr>
    <td><a href="{{% relref "/core/user-api/observable/" %}}" target="_blank">CONFIG_EVL_NR_OBSERVABLES</a></td>
    <td>64</td>
    <td>Максимальное количество наблюдаемых EVL (не включает потоки)</td>
  </tr>
  <tr>
    <td><a href="{{% relref "/core/runtime-settings/" %}}" target="_blank">CONFIG_EVL_LATENCY_USER</a></td>
    <td>0</td>
    <td>Предварительно установленное значение силы тяжести таймера ядра для пользовательских потоков (0 означает использование предварительно откалиброванного значения)</td>
  </tr>
  <tr>
    <td><a href="{{% relref "/core/runtime-settings/" %}}" target="_blank">CONFIG_EVL_LATENCY_KERNEL</a></td>
    <td>0</td>
    <td>Предварительно установленное значение силы тяжести таймера ядра для потоков ядра (0 означает использование предварительно откалиброванного значения)</td>
  </tr>
  <tr>
    <td><a href="{{% relref "/core/runtime-settings/" %}}" target="_blank">CONFIG_EVL_LATENCY_IRQ</a></td>
    <td>0</td>
    <td>Предварительно установленное значение силы тяжести основного таймера для обработчиков прерываний (0 означает использование предварительно откалиброванного значения)</td>
  </tr>
  <tr>
    <td>CONFIG_EVL_DEBUG</td>
    <td>N</td>
    <td>Включить функции отладки</td>
  </tr>
  <tr>
    <td>CONFIG_EVL_DEBUG_CORE</td>
    <td>N</td>
    <td>Включить утверждения отладки ядра</td>
  </tr>
  <tr>
    <td>CONFIG_EVL_DEBUG_CORE</td>
    <td>N</td>
    <td>Enable core debug assertions</td>
  </tr>
  <tr>
    <td>CONFIG_EVL_DEBUG_MEMORY</td>
    <td>N</td>
    <td>Включите проверки отладки в распределителе основной памяти.
        <strong>Этот параметр добавляет значительные накладные расходы, влияющие на показатели задержки</strong></td>
  </tr>
  <tr>
    <td><a href="{{% relref "/core/user-api/thread/#health-monitoring" %}}" target="_blank">CONFIG_EVL_DEBUG_WOLI</a></td>
    <td>N</td>
    <td>Включить контрольные точки предупреждения о несогласованности при блокировке</td>
  </tr>
  <tr>
    <td>CONFIG_EVL_WATCHDOG</td>
    <td>Y</td>
    <td>Включить сторожевой таймер</td>
  </tr>
  <tr>
    <td>CONFIG_EVL_WATCHDOG_TIMEOUT</td>
    <td>4</td>
    <td>Значение тайм-аута сторожевого таймера (в секундах).</td>
  </tr>
  <tr>
    <td><a href="{{% relref "/core/oob-drivers/gpio/" %}}" target="_blank">CONFIG_GPIOLIB_OOB</a></td>
    <td>n</td>
    <td>Включите поддержку внеполосных запросов на обработку линий GPIO.</td>
  </tr>
  <tr>
    <td><a href="{{% relref "/core/oob-drivers/spi/" %}}" target="_blank">CONFIG_SPI_OOB, CONFIG_SPIDEV_OOB</a></td>
    <td>n</td>
    <td>Включите поддержку внеполосных передач SPI.</td>
  </tr>
</table>

#### Включение 32-разрядной поддержки в 64-разрядном ядре (`CONFIG_COMPAT`) {#enable-kernel-compat-mode}

Начиная с [EVL ABI]({{< relref "core/under-the-hood/abi.md" >}})
20 серии v5.6, ядро EVL обычно позволяет 32-разрядным приложениям
выполнять системные вызовы 64-разрядного ядра, когда поддерживаются как 32-,
так и 64-разрядные архитектуры ЦП, такие как код ARM (он же Aarch32),
работающий поверх ядра arm64 (Aarch64). Для arm64 вам необходимо включить
`CONFIG_COMPAT` и `CONFIG_COMPAT_VDSO` в конфигурации ядра.
Чтобы иметь возможность изменять последнее, переменная среды
`CROSS_COMPILE_COMPAT` должна быть установлена в префикс 32-разрядной цепочки
инструментов ARMv7, которая должна использоваться для компиляции vDSO
(да, это довольно запутанно). Например:

```sh
$ make <your-make-args> ARCH=arm64 \
	CROSS_COMPILE=aarch64-linux-gnu- \
	CROSS_COMPILE_COMPAT=arm-linux-gnueabihf- (x|g|menu)config
```

{{% notice tip %}}
Например, если вы планируете запускать EVL на любом из 64-разрядных компьютеров
[Raspberry PI](https://raspberrypi.org), вам может оказаться полезным
использовать 32-разрядные дистрибутивы Linux, ориентированные на PI,
такие как [Raspbian](https://www.raspberrypi.org/downloads/raspbian/).
Для этого обязательно включите `CONFIG_COMPAT` и `CONFIG_COMPAT_VDSO`
для вашего ядра с поддержкой EVL, создав 32-разрядный vDSO, как упоминалось ранее.
{{% /notice %}}

### Создание libevl {#building-libevl}

Общая команда для построения libevl - это:

```sh
$ make [-C $SRCDIR] [ARCH=$cpu_arch] \
	   [CROSS_COMPILE=$toolchain] UAPI=$uapi_dir \
	   [OTHER_BUILD_VARS] [goal...]
```

> Основные переменные построения

| Переменная   |  Описание
| --------   |    -------
| $SRCDIR    |  Путь к этому исходному дереву
| $cpu_arch  |  Архитектура процессора, которую вы создаете для ('arm', 'arm64', 'x86')
| $toolchain |  Необязательный префикс имени файла binutils (например, 'arm-linux-gnueabihf-', 'aarch64-linux-gnu-')

> Другие переменные построения

| Переменная      |  Описание   |  По умолчанию
| --------      |    -------     |  -------
| D={0\|1}      |  Disable or enable debug build, i.e. -g -O0 vs -O2    | 0
| O=$output_dir |  Генерировать двоичные выходные файлы в $output_dir   | .
| V={0\|1}      |  Установите уровень детализации сборки, 0 - краткость | 0
| DESTDIR=$install_dir | Установите библиотеку и двоичные файлы в $install_dir | /usr/evl

> Установить цель

| Цель    |     Действие
| ---     |     ---
| all     |     создайте все двоичные файлы (библиотеку, утилиты и тесты)
| clean   |     удалите файлы сборки
| install |     сделайте всё, скопировав сгенерированные системные двоичные файлы в $DESTDIR в процессе
| install_all | установите, скопировав все сгенерированные двоичные файлы, включая tidbits

#### Перекрестная компиляция EVL

Допустим, исходный код библиотеки находится в ~/git/libevl, а исходные тексты
ядра с ядром EVL расположены в ~/git/linux-evl.

Перекрестная компиляция EVL и установка полученной библиотеки и утилит
в промежуточный каталог, расположенный по адресу
/nfsroot/\<machine\>/usr/evl , будет означать следующее:

> Перекрестная компиляция из отдельного каталога сборки

```sh
# Сначала создайте каталог сборки, в который должны помещаться выходные файлы
$ mkdir /tmp/build-imx6q && cd /tmp/build-imx6q
# Затем запустите процесс сборки+установки
$ make -C ~/git/libevl O=$PWD ARCH=arm \
		CROSS_COMPILE=arm-linux-gnueabihf- \
		UAPI=~/git/linux-evl \
		DESTDIR=/nfsroot/imx6q/usr/evl install
```

или,

> Перекрестная компиляция из дерева исходных текстов библиотеки EVL

```sh
$ mkdir /tmp/build-hikey
$ cd ~/git/libevl
$ make O=/tmp/build-hikey ARCH=arm64 \
		CROSS_COMPILE=aarch64-linux-gnu- \
		UAPI=~/git/linux-evl \
		DESTDIR=/nfsroot/hikey/usr/evl install
```

{{% notice note %}}
Рекомендуется всегда создавать выходные файлы сборки в отдельный каталог сборки
с помощью директивы O= в командной строке _make_, чтобы не загромождать ими
дерево исходных текстов. Создание вывода в отдельный каталог также создает
удобные файлы создания на лету в дереве вывода, которые вы можете использовать
для запуска последующих сборок без необходимости снова упоминать весь ряд
переменных и параметров в командной строке _make_.
{{% /notice %}}

#### Сборка EVL в родной среде

И наоборот, вы можете захотеть создать EVL изначально в целевой системе.
Установка полученной библиотеки и утилит непосредственно в их конечный каталог,
расположенный, например в /usr/evl, может быть выполнена следующим образом:

> Построение изначально из каталога сборки

```sh
$ mkdir /tmp/build-native && cd /tmp/build-native
$ make -C ~/git/libevl O=$PWD UAPI=~/git/linux-evl DESTDIR=/usr/evl install
```

или,

> Построение изначально из дерева исходных текстов библиотеки EVL

```sh
$ mkdir /tmp/build-native
$ cd ~/git/libevl
$ make O=/tmp/build-native UAPI=~/git/linux-evl DESTDIR=/usr/evl install
```

### Тестирование установки

На этом этапе вы действительно захотите [протестировать установку EVL]({{<
relref "core/testing.md" >}}).

---

{{<lastmodified>}}
