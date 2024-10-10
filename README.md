### Systemd. Система инициализация в Linux.

#### <a name='toc'>Содержание</a>

1. [Система инициализации](#initialization_system)
2. [Управление сервисами при использовании systemd](#managing_services)
3. [Дополнительные источники](#%%%%%%%%%%%%%%%%%)   


#### 1. [[⬆]](#toc) <a name='initialization_system'>Система инициализации</a>

После своей загрузки ядро передает управление системе инициализации, Цель этой системы - выполнить дальнейшую инициализацию системы, 
> **Самая главная задача системы инициалиэации - запуск и управление системными службами. Служба (сервис, демон) - специальная программа, выполняющаяся в фоновом режиме и предоставляющая определенные услуги (или, как говорят, сервис - отсюда и второе название).**

**Systemd** - подсистема инициализации и управления службами в Linux. Основная особенность - интенсивное **распараллеливание** запуска служб в процессе загрузки системы, что позволяет существенно ускорить запуск операционной системы.

##### Принцип работы
Система инициализации systemd используется во многих современных дистрибутивах, и на данный момент это самая быстрая система инициализации. Например, upstart - запускает службы параллельно. Но параллельный запуск - не всегда хорошо. Нужно учитывать зависимости служб. Например, сервис d-bus нужен многим другим сервисам. Пока сервис d-bus не будет запушен, нельзя запускать сервисы, которые от него зависят. И если сервис d-bus (или любой другой, от которого зависят какие-то другие сервисы) запускается долго, то все остальные службы будут ждать его загрузки, что скажется на загрузки системы в целом.

В systemd все по другому. При своем запуске службы проверяют, запущена ли необходимая им служба, по наличию файла сокета. Например, в случае с d-bus это файл `/var/run/dbus/system_bus_socket`. Если создать сокеты для всех служб, то можно запускать их параллельно, особо не беспокоясь, что произойдет сбой какой-то службы при запуске из-за отсутствия службы, от которой они зависят. Даже если несколько служб, которым нужен сервис d-bus, запустятся раньше, чем сам сервис d-bus, ничего страшного. Каждая из этих служб отправит в сокет (главное, что он уже открыт!) сообщение, которое обработает сервис d-bus после того, как он запустится. Вот, собственно, и все.

##### Конфигурационные файлы systemd
Systemd запускает сервисы описанные в его конфигурации. Конфигурация состоит из множества файлов, которые называют юнитами или модулями. Все эти юниты/модули разложены в трех каталогах:

| Путь | Описание |
| ------- | ----------- |
| `/etc/systemd/system` | юниты/модули, которые создаёт администратор системы. Обладают самым высоким приоритетом.|
| `/run/systemd/system` | юниты/модули, которые создаются в процессе работы системы. Приоритет этого каталога ниже, чем каталога /etc/systemd/system/, но выше, чем у /usr/lib/systemd/system |
| `/usr/lib/systemd/system` | юниты сервисов, установленных с помощью менеджера пакетов. Самый простой пример — веб-серверы: Apache или Nginx. |

Типичный файл модуля типа service  

![image](https://github.com/user-attachments/assets/0a56b4d5-3834-4be0-8e08-328dddbe27b5)

| Путь | Описание |
| ------- | ----------- |
| `[Unit]` | В секции `Unit` содержится общая информация о сервисе. Эта секция есть и в друrих модулях, а не только в сервисах. |
| `[Service]` | Секция `Service`содержит информацию о сервисе. Параметр `ExecStart` описывает команду, которую нужно запустить. Параметр `Туре` указывает, как сервис будет уведомлять systemd об окончании запуска. |
| `[Install]` | Секция Insta/1 содержит информацию о цели, в которой должен запускаться сервис. |

Таким образом можнл создать собственный сервис, который потом нужно поместить в файл `/etc/systemd/systern/имя_сервиса. service`. После этого нужно перезапустить саму systemd, чтобы она узнала о новом сервисе:
```
sudo systemctl daemon-reload
```

##### Типы модулей системы инициализации systemd

| Тип | Описание |
| ------- | ----------- |
| `service` | Служба (сервис, демон), которую нужно запустить. Пример имени модуля: network.service. Изначально systemd поддерживала сценарии SysV (чтобы управлять сервисами можно service было как при использовании init), но в последнее время в каталоге /etc/init.d систем, которые используют systemd, практически пусто (или вообще пусто), а управление сервисами осуществляется только посредством systemd |
| `target` | Цель. Используется для группировки модулей других типов. В systemd нет уровней запуска, вместо них используются цели. Например, цель multi-user.target описывает, какие модули должны быть запущены в многопользовательском режиме |
| `snapshot` | Точка монтирования. Представляет точку монтирования. Система инициализации systemd контролирует все точки монтирования. При использовании systemd файл /etc/fstab уже не главный, хотя все еще может использоваться для определения точек монтирования |
| `automount` | Автоматическая точка монтирования. Используется для монтирования сменных носителей - флешек, внешних жестких дисков, оптических дисков и т.д. |
| `socket` | Сокет. Представляет сокет, находящийся в файловой системе или в Интернете. Поддерживаются сокеты AF_INET, AF_INET6, AF_UNIX. Реализация довольно интересная. Например, если сервису service1.service соответствует сокет service1.socket, то при попытке установки соединения с service1.socket будет запущен service1.service  |
| `device` | Устройство. Представляет устройство в дереве устройств. Работает вместе с udev: если устройство описано в виде правила udev, то его можно представить в systemd в виде модуля device |
| `path` | Файл или каталог, созданный где-то в файловой системе |
| `scope` | Процесс, который создан извне |
| `slice` | Управляет системными процессами. Представляет собой группу иерархически организованных модулей |
| `swap` | Представляет область подкачки (раздел подкачки) или файл подкачки (свопа) |
| `timer` | Представляет собой таймер системы инициализации systemd  |

##### Цели


#### 2. [[⬆]](#toc) <a name='managing_services'>Управление сервисами при использовании systemd</a>

##### Параметры программы systemctl

| Command | Description |
| ------- | ----------- |
| `git status` | рьирлорплопрл |
| `lern text` | some text, some text, some text |
| `lern text text` | some text, some text, some text, some text, some text, some text |
| `lern text` | some text, some text, some text |
| `lern text` | some text, some text, some text |


#### 4. [[⬆]](#toc) <a name='recommended_sources'>Дополнительные источники</a>

1. [????????????????](https://help.ubuntu.ru/wiki/grub)
2. [????????????????](https://www.alexgur.ru/articles/2275/)
3. [????????????????](https://losst.pro/nastrojka-zagruzchika-grub)
4. Весь Linux Для тех, кто хочет стать профессионалом, стр.324
