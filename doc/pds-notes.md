# Глоссарий

- TDM — Time-division multiplexing, мультиплексирование с разделением по
  времени.
- span — Полоса. В терминах DAHDI олицетворяет собой канал передачи данных. Для
  E1/T1 соответствует одному физическому транку.
- channel — Канал. В терминах DAHDI соответствует логическому каналу внутри
  одной полосы. Для E1/T1 соответствует конкретному временному интервалу
  (таймслоту).
- агрегированный канал — Упорядоченное множество каналов одной полосы,
  представляющее единый логический канал. Идентифицируется каналом с наименьшим
  порядковым номером (пырвым в наборе, называемым мастером).

# Интерфейс драйвера устройства DAHDI

## Span

- spanconfig — Вызывается подсистемой DAHDI в ответ на запрос настройки
  параметров конкретной полосы. Для E1 это line code (AMI или HDB3), framing
  (G.704, G.704 без CRC4), signaling (CCS или CAS и подтипы).
- startup — Сообщение от системы DAHDI, что она готова к работе с полосой.
  Устройство может, например, поднять несущую.
- shutdown — Сообщение от системы DAHDI, что она закончила работать с полосой.

## Channel

- chanconfig — Сообщение от системы DAHDI об изменении настройки конкретного
  канала.
- open — Сообщение от системы DAHDI о начале передачи данных через конкретный
  канал. В случае агрегированного канала в hard HDLC режиме, сообщение
  передаётся только для мастер-канала.
- close — Сообщение от системы DAHDI о завершении передачи данных через
  конкретный канал.

# Приём и передача данных

Режим приёма и передачи данных определяет механизм, которым производится обмен
данными между подсистемой DAHDI и драйвером устройства. Режим передачи определён
для каждого из каналов.

## TDM режим

Данный режим традиционно применяется для передачи голосовых данных, но может
быть также использован для передачи данных, если устройство или его драйвер не
поддерживают HDLC режим. В последнем случае всю работу по
кодированию/декодированию HDLC кадров, по подсчёту/проверке контрольной суммы
(если требуется) и детектированию ошибок берёт на себя подсистема DAHDI.

В TDM режиме приём и передача между устройством и подсистемой DAHDI должны
осуществляться строго по таймеру (для E1 период равен 1 мс для стандартной
сборки). Каждый цикл для конкретной полосы производится приём и передача блока
данных между DAHDI и устройством.

Каждый блок состоит из серии кусочков (chunk) фиксированного размера (всегда 8
октетов для стандартной сборки DAHDI), представляющих данные конкретных открытых
каналов.

В данном режиме агрегированные и неагрегированные каналы для драйвера устройства
ничем не отличаются.

## HDLC режим

В данном режиме вся работа по кодированию/декодированию HDLC кадров, по
подсчёту/проверке контрольной суммы (если требуется) и детектированию ошибок
берёт на себя устройство. Подсистема DAHDI принимает и передает целые пакеты
(кадры) данных.

# Протокол настройки и передачи данных

Данный протокол большей частью обусловлен особенностями интерфейса драйвера
устройств DAHDI. Протокол может быть изменён по согласованию сторон. В данный
момент описывается реализация текущего прототипа.

В качестве рабочего имени устройства/драйвера было выбрано PDS.

Если не оговорено особо, все значения передаются в сетевом порядке байт
(big-endian notation).

Обмен данными между драйвером и устройством происходит посредством передачи
Ethernet кадров через соответствующий сетевой интерфейс (по умолчанию eth1,
может быть изменён заданием параметра модуля master с именем соответствующего
интерфейса).

Поскольку соединение с устройством типа точка-точка и производится привязка к
определённому интерфейсу (мастеру), то в качестве адреса назначения было решено
использовать широковещательный адрес. В любом случае, текущий прототип драйвера
никак не анализирует Ethernet адреса получателя и отправителя.

Передаваемые кадры данных поделены на три класса:

1. TDM Event — блок данных в TDM режиме.
2. HDLC Event — HDLC кадр в соответствующем режиме.
3. Control Message — все управляющие сообщения.

Каждый класс идентифицируется собственным значением поля EtherType. Далее
рассматриваются пакеты после удаления Ethernet заголовка, если иное не оговорено
особо.

## PDS TDM Event

Представляет собой блок данных, соответствующих конкретной полосе данных. Блок
содержит кусочки данных всех каналов полосы, открытых в данном режиме.

Подтверждения приёма/передачи блока не передаются, потери могут быть
детектированы по порядковому номеру пакета: каждая полоса имеет отдельные
счётчики приёма и передачи блока данных.

В текущем прототипе данный режим реализован в основе своей, но практически никак
не тестировался. В данный момент приём и передача данных тихо и безусловно
отключены.

В связи с тем, что TDM режим решено не использовать, формат пакета далее не
описывается: конкретный формат можно посмотреть в соответствующем заголовочном
файле.

Замечание: Формат пакета бит-в-бит совместим с DAHDI TDMoE, за исключением
значения EtherType.

## PDS HDLC Event

Представляет из себя пакет с EtherType равным 0x7a01. Сообщение имеет следующий
заголовок:

1. span — 16-битный номер полосы (нумерация начинается с единицы).
2. cutoff — 8-битное поле. Если не равно нулю, то определяет реальную часть
   передаваемых данных: всё, что идёт после должно игнорироваться при приёме.
   Необходимо при передаче коротких HDLC кадров из-за выравнивания Ethernet
   кадров.
3. flags — 8-битное поле. Флаги задаются установкой в единицу соответствующих
   битов поля. На данный момент для данного класса не определено никаких флагов
   — поле задано лишь для единообразия основной части заголовка с заголовками
   других классов и, возможно, для целей расширения в будущем.
4. seq — 16-битный порядковый номер пакета. Для каждого последующего номер
   увеличивается на единицу по модулю 2^16. Каждая полоса имеет свой счётчик.
5. channel — 16-битный номер мастер-канала.

Непосредственно за заголовком идут данные инкапсулированного HDLC кадра. Тип
данных внутри кадра никак не контроллируется (транспортный уровень).

Замечание: подтверждения приёма/передачи сообщения не передаются, потери могут
быть детектированы по порядковому номеру пакета. Обоснование: на данном этапе,
потери пакетов (кадров) — это нормальная штатная ситуация, поэтому не вижу
смысла усложнять систему и пытаться гарантировать доставку данного класса
сообщений.

## PDS Control Message

Представляет из себя пакет с EtherType равным 0x7a02. Сообщение имеет следующий
заголовок:

1. span — 16-битный номер полосы (нумерация начинается с единицы). Нулевое
   значение здесь означает, что команда/уведомление относится ко всему
   устройству (контроллеру), а не только к конкретной полосе.
2. code — 8-битный код типа сообщения (операции или уведомления).
3. flags — 8-битное поле. Флаги задаются установкой в единицу соответствующих
   битов поля. На данный момент для данного класса определен один единственный
   флаг:
   - PDS_MESSAGE_REPLY — нулевой бит; означает, что данное сообщение является
     ответом на соответствующий запрос. Для запросов всегда сброшен. У запроса и
     ответа все остальные поля заголовка обязаны совпадать (span, code и seq).
4. seq — 16-битный порядковый номер пакета. Для каждого последующего номер
   увеличивается на единицу по модулю 2^16. Каждая полоса имеет свой счётчик.

Непосредственно за заголовком следует последовательность из нуля или более
аргументов. Если не определено иначе, аргументы представляют собой
последовательность 16-битных величин (в сетевом порядке).

Первый аргумент ответа должен быть кодом статуса выполения запроса.

Неизвестные (лишние) аргументы следует тихо игнорировать. Коды возврата:

- 0 — Ok: Запрос выполнен успешно.
- 1 — NoSys: Команда и/или режим известны, но не реализованы.
- 2 — InVal: Недостаточно аргументов и/или указано неподдерживаемое значение.
- 3 — Busy: Устройство или ресурс занято и не может в данный момент обрабатывать
  запросы. Может, например, означать, что канал уже открыт, либо используется
  для передачи сигналов.

### PDS Reset

Производит сброс контроллера устройства. Не требует аргументов. После сброса
внутренне состояние определено следующим образом:

- счётчик порядкового номера пакета сброшен в ноль;
- sync source: внешний источник собственной полосы;
- line code: HDB3;
- framing: G.704;
- signaling: CCS режим, канал не задан (сигнализация не используется);
- все каналы закрыты.

### PDS Setup

Установка режима работы полосы. Параметры:

- sync-source-span;
- line-code;
- framing;
- signaling.

См. spanconfig выше за описанием параметров.

(TBD: Развернуть описание согласно комментарию в заголовочном файле протокола.)

### PDS Enslave

Агрегация канала. Параметры:

- channel — дочерний канал;
- master-channel — мастер-канал.

Если channel был ранее агрегирован, то он освобождается (удаляется из
агрегированного канала). Если channel ≠ master-channel, то он добавляется в
набор агрегированных каналов соответствующего мастер-канала.

### PDS Open TDM

Включает TDM режим для определённого канала. В данной реализации отключено.

### PDS Open HDLC

Открывает канал для передачи в HDLC режиме. Параметр: номер канала.

Замечание: указанный канал должен быть мастером.

### PDS Close

Закрывает канал, прекращает передачу данных по нему. Параметр: номер канала.

Замечание: Следует возвращать положительный статус даже если канал уже закрыт.

### PDS Notify Alarms

Уведомление о статусе от устройства. Параметр: битовая карта состояния.

Замечание: Периодически посылается устройством, не требует ответа.

### PDS Notify Counts

Уведомление о текущих значениях счетчиков. Параметр: массив 32-битных счетчиков.

Замечание: Периодически посылается устройством, не требует ответа.
