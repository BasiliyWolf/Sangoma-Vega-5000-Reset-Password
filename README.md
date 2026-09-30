# Sangoma Vega 5000 (VS0181) — U-Boot Access, Configuration Backup and Factory Reset

> [Русская версия](#русская-версия) | [English version](#english-version)

---

# Русская версия

## Sangoma Vega 5000 (VS0181): доступ к U-Boot, сохранение лицензии и конфигурации перед сбросом

Эта инструкция предназначена для **Sangoma Vega 5000 / VS0181** и описывает способ:

- войти в U-Boot;
- загрузить встроенный Linux с `rdinit=/bin/sh`;
- получить доступ к MTD-разделам;
- смонтировать разделы конфигурации в режиме `read-only`;
- сохранить текущую конфигурацию и лицензионный ключ;
- только после резервного копирования выполнить заводской сброс.

## ⚠️ Важное предупреждение

Команда U-Boot:

```text
factoryreset
```

может удалить конфигурацию и **лицензионный ключ**.

На самом устройстве команда отображается с предупреждением:

```text
factoryreset - reset all configuration to default (CAUTION - will erase license key)
```

Sangoma указывает, что если лицензионный ключ не удалось сохранить до сброса, его восстановление может потребовать обращения в поддержку с серийным номером устройства.

**Не выполняйте `factoryreset`, `sf erase`, `sf write`, `sf update` или другие команды записи во flash до сохранения лицензии и необходимой конфигурации.**

---

## 1. Вход в U-Boot

Подключитесь к консольному порту устройства.

Параметры последовательного порта:

```text
Speed:        115200
Data bits:    8
Parity:       None
Stop bits:    1
Flow control: None
```

Перезагрузите Sangoma Vega 5000.

Во время начала загрузки нажимайте:

```text
Enter
```

до появления приглашения:

```text
U-Boot#
```

---

## 2. Ручная загрузка firmware

Не используйте:

```text
run sys1
```

при попытке загрузить shell через `rdinit=/bin/sh`, потому что переменная `sys1` добавляет дополнительный аргумент `sys1` в командную строку ядра.

Вместо этого повторите загрузочную последовательность вручную:

```text
sf probe
spi_readu 0x80200000 0x0100000
setenv bootargs 'rdinit=/bin/sh console=ttyO1,115200n8'
bootm 0x80200000
```

На устройстве с firmware, на котором тестировалась эта процедура, загружается образ:

```text
VEGAARM_R120S006
```

После запуска ядра должно появиться приглашение shell:

```text
#
```

Сообщение вида:

```text
/bin/sh: can't access tty
```

не мешает дальнейшей работе через консоль.

---

## 3. Подготовка Linux shell

Добавьте стандартные каталоги в `PATH`:

```sh
export PATH=/bin:/sbin:/usr/bin:/usr/sbin
```

Смонтируйте виртуальные файловые системы:

```sh
mount -t proc proc /proc
mount -t sysfs sysfs /sys
```

Проверьте параметры загрузки и структуру системы:

```sh
cat /proc/cmdline
cat /proc/mtd
mount
df -h
ls -la /
```

Дополнительно:

```sh
ls -la /dev/mtd*
ls -la /etc
ls -la /var
ls -la /mnt
ls -la /tmp
```

---

## 4. Создание MTD device nodes

Если `/dev/mtd*` отсутствуют, выполните:

```sh
mdev -s
```

После этого проверьте:

```sh
ls -l /dev/mtd*
```

---

## 5. Разметка flash

На исследованном Sangoma Vega 5000 / VS0181 была обнаружена следующая разметка:

```text
dev:    size      erasesize  name

mtd0:   00010000  00010000   "u-boot-spl"
mtd1:   00040000  00010000   "u-boot"
mtd2:   00020000  00010000   "env"
mtd3:   00090000  00010000   "vega-cfg"
mtd4:   00700000  00010000   "system1"
mtd5:   00700000  00010000   "system2"
mtd6:   00100000  00010000   "config"
```

В адресах SPI flash это соответствует примерно:

```text
0x000000 - 0x00FFFF   u-boot-spl
0x010000 - 0x04FFFF   u-boot
0x050000 - 0x06FFFF   env
0x070000 - 0x0FFFFF   vega-cfg
0x100000 - 0x7FFFFF   system1
0x800000 - 0xEFFFFF   system2
0xF00000 - 0xFFFFFF   config
```

> Разметка может отличаться на другой версии аппаратной платформы или firmware. Всегда сначала проверяйте `cat /proc/mtd`.

---

## 6. Монтирование конфигурации только для чтения

Создайте точки монтирования:

```sh
mkdir -p /mnt/vega-cfg
mkdir -p /mnt/config
```

Смонтируйте разделы **только для чтения**:

```sh
mount -t jffs2 -o ro /dev/mtdblock3 /mnt/vega-cfg
mount -t jffs2 -o ro /dev/mtdblock6 /mnt/config
```

Проверьте результат:

```sh
mount
```

Просмотрите файлы:

```sh
find /mnt/vega-cfg -maxdepth 3 -type f -print
find /mnt/config -maxdepth 3 -type f -print
```

---

## 7. Просмотр основного конфигурационного файла

На исследованном устройстве основной конфигурационный файл находился здесь:

```text
/mnt/vega-cfg/CONFIG
```

Просмотреть его можно командой:

```sh
cat /mnt/vega-cfg/CONFIG
```

Лучший вариант — **сначала выгрузить файл целиком на другой компьютер**, используя любой доступный безопасный способ, и только затем работать с его содержимым.

---

## 8. Поиск лицензионного ключа

В конфигурационном файле найдите секцию наподобие:

```ini
[vega100-v3.9]
[_advanced]
 _diff_protocol_part=0
 _license=001300505820afd4002400002006180000000..............................xxxxx
```

Реальное значение `_license` необходимо сохранить **целиком и без изменений**.

Не публикуйте действующий лицензионный ключ в открытом репозитории.

Рекомендуется сохранить:

- файл `CONFIG`;
- значение `_license`;
- полный дамп раздела `vega-cfg`;
- при необходимости — содержимое раздела `config`.

---

## 9. Резервное копирование MTD-разделов

После появления `/dev/mtd*` можно сделать бинарные копии разделов в RAM:

```sh
dd if=/dev/mtd3ro of=/tmp/vega-cfg.bin bs=64k
dd if=/dev/mtd6ro of=/tmp/config.bin bs=64k
```

Проверьте:

```sh
ls -lh /tmp/vega-cfg.bin /tmp/config.bin
```

Ожидаемый размер:

```text
vega-cfg.bin   ~576 KB
config.bin     ~1 MB
```

> Файлы в `/tmp` находятся в RAM и пропадут после перезагрузки. Их необходимо выгрузить с устройства до reboot/reset.

---

## 10. Что сохранять перед factory reset

Если требуется сохранить только лицензию:

1. Получите доступ к `/mnt/vega-cfg/CONFIG`.
2. Найдите `_license=`.
3. Сохраните полное значение лицензионного ключа вне устройства.

Если необходимо восстановить устройство максимально близко к исходному состоянию:

1. Сохраните `/mnt/vega-cfg/CONFIG`.
2. Сохраните полный `/dev/mtd3ro`.
3. Сохраните `/dev/mtd6ro`.
4. При возможности сохраните полный SPI flash.
5. Только после проверки резервных копий выполняйте сброс.

---

## 11. Factory reset

Заводской сброс следует выполнять **только после сохранения лицензии и необходимых данных**.

В U-Boot присутствует команда:

```text
factoryreset
```

Перед её использованием ещё раз убедитесь, что лицензия и конфигурация сохранены вне устройства.

### Не выполняйте reset, если:

- `_license` ещё не сохранён;
- резервная копия `CONFIG` не выгружена;
- бинарные дампы существуют только в `/tmp`;
- вы не уверены, что сможете восстановить лицензию через Sangoma.

---

## 12. Примечание

Эта инструкция получена в результате работы с конкретным:

```text
Sangoma Vega 5000
Model/Platform: VS0181 / V3K
CPU platform: TI AM335x / ARMv7
SPI flash: N25Q128, 16 MiB
U-Boot: 2013.10
Firmware image: VEGAARM_R120S006
```

Перед использованием на другом устройстве обязательно сравните:

```text
printenv
bdinfo
cat /proc/mtd
```

---

# English version

## Sangoma Vega 5000 (VS0181): U-Boot access, license backup and configuration recovery before factory reset

This guide describes a recovery procedure for **Sangoma Vega 5000 / VS0181** that can be used to:

- enter U-Boot;
- boot the embedded Linux system with `rdinit=/bin/sh`;
- access MTD flash partitions;
- mount configuration partitions read-only;
- save the current configuration and license key;
- perform a factory reset only after a verified backup exists.

## ⚠️ Important warning

The U-Boot command:

```text
factoryreset
```

may erase the configuration and the **license key**.

The device itself displays the following warning:

```text
factoryreset - reset all configuration to default (CAUTION - will erase license key)
```

Sangoma indicates that if the license key cannot be saved before a reset, license recovery may require contacting support and providing the device serial number.

**Do not run `factoryreset`, `sf erase`, `sf write`, `sf update`, or any other flash-writing command before backing up the license and required configuration.**

---

## 1. Entering U-Boot

Connect to the device console port.

Serial settings:

```text
Speed:        115200
Data bits:    8
Parity:       None
Stop bits:    1
Flow control: None
```

Reboot the Sangoma Vega 5000.

At the beginning of the boot process, repeatedly press:

```text
Enter
```

until the following prompt appears:

```text
U-Boot#
```

---

## 2. Booting the firmware manually

Do not use:

```text
run sys1
```

when trying to start a shell with `rdinit=/bin/sh`, because the `sys1` environment script appends an additional `sys1` argument to the kernel command line.

Instead, reproduce the boot sequence manually:

```text
sf probe
spi_readu 0x80200000 0x0100000
setenv bootargs 'rdinit=/bin/sh console=ttyO1,115200n8'
bootm 0x80200000
```

On the unit used to test this procedure, the following firmware image was loaded:

```text
VEGAARM_R120S006
```

After the kernel starts, a shell prompt should appear:

```text
#
```

A message such as:

```text
/bin/sh: can't access tty
```

does not prevent further console operation.

---

## 3. Preparing the Linux shell

Set a usable `PATH`:

```sh
export PATH=/bin:/sbin:/usr/bin:/usr/sbin
```

Mount the virtual filesystems:

```sh
mount -t proc proc /proc
mount -t sysfs sysfs /sys
```

Inspect the boot parameters and filesystem:

```sh
cat /proc/cmdline
cat /proc/mtd
mount
df -h
ls -la /
```

Additional checks:

```sh
ls -la /dev/mtd*
ls -la /etc
ls -la /var
ls -la /mnt
ls -la /tmp
```

---

## 4. Creating MTD device nodes

If `/dev/mtd*` devices are missing, run:

```sh
mdev -s
```

Then verify:

```sh
ls -l /dev/mtd*
```

---

## 5. Flash layout

The tested Sangoma Vega 5000 / VS0181 exposed the following MTD layout:

```text
dev:    size      erasesize  name

mtd0:   00010000  00010000   "u-boot-spl"
mtd1:   00040000  00010000   "u-boot"
mtd2:   00020000  00010000   "env"
mtd3:   00090000  00010000   "vega-cfg"
mtd4:   00700000  00010000   "system1"
mtd5:   00700000  00010000   "system2"
mtd6:   00100000  00010000   "config"
```

Approximate SPI flash address map:

```text
0x000000 - 0x00FFFF   u-boot-spl
0x010000 - 0x04FFFF   u-boot
0x050000 - 0x06FFFF   env
0x070000 - 0x0FFFFF   vega-cfg
0x100000 - 0x7FFFFF   system1
0x800000 - 0xEFFFFF   system2
0xF00000 - 0xFFFFFF   config
```

> The layout may differ on another hardware revision or firmware version. Always verify it first with `cat /proc/mtd`.

---

## 6. Mounting the configuration read-only

Create mount points:

```sh
mkdir -p /mnt/vega-cfg
mkdir -p /mnt/config
```

Mount both partitions **read-only**:

```sh
mount -t jffs2 -o ro /dev/mtdblock3 /mnt/vega-cfg
mount -t jffs2 -o ro /dev/mtdblock6 /mnt/config
```

Verify:

```sh
mount
```

List available files:

```sh
find /mnt/vega-cfg -maxdepth 3 -type f -print
find /mnt/config -maxdepth 3 -type f -print
```

---

## 7. Reading the main configuration file

On the tested unit, the main configuration file was located at:

```text
/mnt/vega-cfg/CONFIG
```

It can be displayed with:

```sh
cat /mnt/vega-cfg/CONFIG
```

The preferred approach is to **copy the complete file to another computer first** using any available safe transfer method.

---

## 8. Finding the license key

Search the configuration for a section similar to:

```ini
[vega100-v3.9]
[_advanced]
 _diff_protocol_part=0
 _license=001300505820afd4002400002006180000000..............................xxxxx
```

The actual `_license` value should be saved **completely and without modification**.

Do not publish a real production license key in a public Git repository.

It is recommended to save:

- the complete `CONFIG` file;
- the complete `_license` value;
- a binary image of the `vega-cfg` partition;
- the `config` partition if required.

---

## 9. Backing up the MTD partitions

Once `/dev/mtd*` exists, binary copies can be created in RAM:

```sh
dd if=/dev/mtd3ro of=/tmp/vega-cfg.bin bs=64k
dd if=/dev/mtd6ro of=/tmp/config.bin bs=64k
```

Verify:

```sh
ls -lh /tmp/vega-cfg.bin /tmp/config.bin
```

Expected sizes:

```text
vega-cfg.bin   ~576 KB
config.bin     ~1 MB
```

> `/tmp` is stored in RAM. These files will be lost after reboot or reset, so transfer them off the device first.

---

## 10. What should be saved before a factory reset

If only the license is required:

1. Access `/mnt/vega-cfg/CONFIG`.
2. Find `_license=`.
3. Save the full license value outside the device.

If the goal is to preserve as much of the original device state as possible:

1. Save `/mnt/vega-cfg/CONFIG`.
2. Save the full `/dev/mtd3ro` partition.
3. Save `/dev/mtd6ro`.
4. If possible, create a full SPI flash backup.
5. Verify the backups before resetting the gateway.

---

## 11. Factory reset

A factory reset should be performed **only after the license and required configuration have been backed up**.

The U-Boot command is:

```text
factoryreset
```

Before running it, verify again that the backups are stored outside the gateway.

### Do not reset the device if:

- `_license` has not been saved;
- the `CONFIG` backup has not been transferred off the device;
- binary dumps exist only in `/tmp`;
- you are not sure the license can be recovered through Sangoma support.

---

## 12. Tested platform

This procedure was derived from work with the following hardware/software combination:

```text
Sangoma Vega 5000
Model/Platform: VS0181 / V3K
CPU platform: TI AM335x / ARMv7
SPI flash: N25Q128, 16 MiB
U-Boot: 2013.10
Firmware image: VEGAARM_R120S006
```

Before using the procedure on another unit, compare at least:

```text
printenv
bdinfo
cat /proc/mtd
```

---

## Disclaimer

This procedure involves direct access to the bootloader and raw flash partitions.

Incorrect use of U-Boot or MTD write/erase commands can make the gateway unbootable or permanently erase configuration/licensing data.

Use it at your own risk and always create verified backups before making changes.
