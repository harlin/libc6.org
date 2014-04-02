Title: Выбираем правильный формат архива для бэкапа.
Slug: tar-is-bullshit-use-dar-instead
Category: Администрирование
Date: 2014-03-12 21:00
Source: False

[TOC]

## Введение

Есть известная поговорка, что системные администраторы делятся на три типа: тех, кто не делает бэкапы; тех, кто _уже_ делает бэкапы и тех, кто делает и проверяет, что бэкапы рабочие.

Однако этого недостаточно, и сейчас для пользователя системы бэкапов важен такой параметр как скорость, причём не только скорость самого бэкапа, то есть архивирования файлов, но и восстановления.

Согласитесь, ведь глупо считывать целиком весь архив размером в 50-100-1000 гигабайт, чтобы извлечь один файл. 

А если у вас эти архивы инкрементальные, то чтобы восстановить один файл за нужную дату, нужно будет последовательно читать все архивы по порядку. И всё становится намного хуже, если файл архива расположен на удалённом сервере.

И именно этим вы будете заниматься, если будете использовать формат архива TAR. Ведь это промышленный стандарт для архивов, и он используется во многих утилитах для бэкапа.

И причина такого поведения очень проста — отсутствие индексов, по которым можно вытащить один файл из архива.

У TAR вообще много недостатков, многие из них — фатальные. Я приведу небольшой список основных недостатков, на которые натолкнулся за время исследования:

  * Отсутствие индекса, и как следствие — невозможность вытащить один файл, не считывая весь архив.
  * Зоопарк форматов: GNU tar разных версий, BSD tar тоже разных версий, которые подразумевают иногда несовместимость архивов между собой.
  * Отсутствие встроенного шифрования.
  * Невозможность селективного сжатия (зачем нам, например, сжимать jpeg?).
  * Совершенно невнятные ошибки при архивации и при восстановлении (типичным сообщением об ошибке является «Unexpected EOF of archive», которое может обозначать что угодно).
  * Он может просто не сделать архив, например, потому что часть файлов была удалена во время архивации, и tar никак эту ситуацию не обрабатывает.

И это только то, что я вспомнил с ходу.

Я провёл достаточно обширное исследование архиваторов (zip, rar, 7zip) и даже всяких монструозных систем для бэкапов: опенсорсных (ну или условно опенсорсные) типа bacula, и проприетарных.

И нашёл формат архива, который меня и компанию более или менее устроил по всем параметрам и подходил к моей задаче.

Я предлагаю вам обратить внимание на архиватор [dar](http://dar.linux.free.fr/) и коротко расскажу о его преимуществах и недостатках (они есть, но их немного, и с ними можно жить), а потом перейду к практическим примерам.

### Достоинства

  * У файла архива есть индекс, и даже больше — сам индекс можно разместить отдельно и забэкапить, что позволит восстановить архив, если у него был повреждён индекс.
  * Не только привычные дифференциальный и инкрементальный бэкапы, но и декрементальный.
  * Шифрование (blowfish, aes, twofish, serpent, camelia).
  * Можно сжимать файлы с определёнными расширениями.
  * Можно не сжимать файлы с определёнными расширениями.
  * Можно гибко управлять процессом архивации и разархивации (как реагировать на удаление файлов, на изменение, перемещение и пр.).
  * Есть поставляемый с dar менеджер архивов, он позволяет не восстанавливать все архивы подряд при поисках файла, автоматически выбирая только нужные.

Это только основные его преимущества, вообще dar очень богат на фичи и об этом лучше всего говорит цитата из man: «...But due to the lack of available unused letter for command line options...».

Проект достаточно активно развивается и хорошо поддерживается разработчиком. На свои вопросы я получал ответ в течении пары дней, причём ответы всегда очень содержательны. Я знаю только один проект с таким же уровнем поддержки — [libguestfs](http://libguestfs.org/), к слову, я про него [уже писал](http://habrahabr.ru/post/121218/).

### Недостатки

  * Нереальное количество опций, а если всерьёз закапываться в то, как реагировать на различные изменения файлов при архивации/разархивации — свихнуться можно.
  * Совершенно неочевидный процесс архивации/восстановления через пайп (например, по ssh).
  * В определённых ситуациях dar может потребовать реакции пользователя (это решается через добавление аргументов к команде, но, как правило, эта интерактивность проявляется очень неожиданно, особенно пока вы пишете свои первые скрипты с dar).

Не то, чтобы это недостаток, но dar очень многословен. Если tar после выполнения операции напишет одну строчку, то dar пишет очень много и очень подробно. И конечно, его можно заткнуть (ещё никто не избегал `>/dev/null 2>&1`).

## Практикум

Я готов поспорить, что часть аудитории уже побежала устанавливать dar в своих любимых дистрибутивах и читать man'ы самостоятельно. Для тех же, кто остался, я расскажу как им пользоваться. А когда энтузиасты вернутся, я покажу, как пользоваться этой замечательной утилитой, и расскажу о некоторых базовых понятиях, которые вы встретите на страницах `man dar`.

### Архивирование

Первый пример, самый простой:

    dar -R $HOME -c /mnt/backup/archive

Архивирует директорию /home.

Давайте исключим пару директорий (~/movies, ~/downloads):

    dar -R $HOME -c /mnt/backup/archive -P movies -P downloads

Я думаю, все уже заметили, что название архива никак не упоминает расширение файла .dar. А ещё в имени файла откуда-то взялась цифра 1. Это всё потому, что dar изначально предназначается для бэкапа на сменные носители (CD, DVD или, например, ленточные накопители), поэтому он архивирует в _слайсы_, а циферка 1 возникает потому, что этот слайс _первый_. А поскольку мы не указывали ключик `-s 100M` — и единственный. У dar есть также ключи для запуска скриптов, при выполнении определённых операций (такие ключи есть и у tar). Например, когда слайс записан, можно выполнить скрипт и поменять носитель, а потом ещё раз и так далее.

В общем, разбитием архива на несколько частей никого не удивишь.

По умолчанию dar архивирует без сжатия, и чтобы включить сжатие нужно передать ему ключик `-z algo:level`. Поддерживаются gzip, bzip2, lzo. И на выходе мы получим такой же файл .N.dar, без добавления всяких .gz и прочих. Архиватор сам знает что у него внутри.

Перейдём к следующей вкусности — исключениям для сжатия при архивации:

    dar -R $HOME -c /mnt/backup/archive -Y "*.txt" "*.fb2" -Z "*.mp4"

Ключ `-Y` указывает для каких файлов нужно включать компресию, а `-Z` — для каких не нужно. Причём по умолчанию исключение имеет более высокий приоритет (но это поведение можно поменять при необходимости).

А теперь приступим к дифференциальному, инкрементальному и самому вкусному — декрементальному бэкапу.

Если кто-нибудь не знает, что это означает — нестрашно, я расскажу:

 * Дифференциальный: сначала создаётся полная копия, а каждый последующий день сохраняется только разница между этой копией и текущим состоянием файлов.
 * Инкрементальный: создаётся полная копия, на следующий день сохраняется разница между полной и текущим состоянием, на третий — разница между вторым днём и третьим.
 * Декрементальный: каждый день сохраняется полная копия и сохраняется разница между текущим состоянием и вчерашним.

При этом вам никто не мешает реализовывать одновременно и инкрементальный, и декрементальный бэкап. Так что за две недели бэкап может выглядеть так (сверху дни недели, снизу тип бэкапов d- — декрементальный, +i — инкрементальный):

    M  T  W  T  F  S  S  M  T  W  T  F  S
    d- d- d- d- d- d- f +i +i +i +i +i +i

Что позволит обойтись одной полной копией, сэкономив существенное количество места.

Следует также знать, что единственным, что вам понадобится для того чтобы сделать инкрементальный архив — индекс. В терминах dar индекс называется каталогом, а сохранение индекса в файл — изоляцией каталога. Также я далее буду использовать только термины инкрементальный/декрементальный, поскольку дифференциальный архив — частный случай инкрементального

Итак, давайте создадим инкрементальный архив:

    dar -R $HOME -c /mnt/backup/archive_monday -A /mnt/backup/archive 

А теперь сделаем ещё один:

    dar -R $HOME -c /mnt/backup/archive_tuesday -A /mnt/backup/archive_monday

Идею поняли? Отлично, едем дальше. А сейчас давайте сохраним индекс отдельно (обратите внимание, мы его не вырежем из архива, а просто скопируем, это как с бэкапом mbr. Вы ведь делаете бэкап своего загрузчика?), чтобы потом не ворочать многогигабайтным бэкапом только ради создания инкрементного архива. Мы делаем сейчас «изоляцию на лету», но каталог можно сохранить в любое удобное время, взяв его из готового архива.

    dar -R $HOME -c /mnt/backup/archive_wednesday -A /mnt/backup/archive_tuesday -@ /mnt/backup/CAT_archive_wednesday 

А теперь давайте сделаем бэкап ещё разок, используя только индекс CAT_archive_wednesday:

    dar -R $HOME -c /mnt/backup/archive_thursday -A /mnt/backup/CAT_archive_wednesday -@ /mnt/backup/CAT_archive_thursday

Отлично, мы разобрались со знакомым многим инкрементальным бэкапом, но что такой за зверь декрементальный бэкап?

Для начала нам нужен один вчерашний полный архив из которого мы будем делать декрементальный, и сегодняшний полный.

    dar -R $HOME -c /mnt/backup/archive_sunday
    dar -R $HOME -+ /mnt/backup/archive_saturday_decremental -A /mnt/backup/archive_saturday -@ /mnt/backup/archive_sunday -ad

Вообще, тут немного всё запутано (привыкайте), поскольку `-+` по документации создан для _объединения_ двух архивов, а `-@`, как мы уже говорили, служит для изоляции каталога «на лету», и ключ `-ad` меняет поведение этих ключей, чтобы реализовать декремент. В некотором смысле это логично. Наверное.

Ну вот мы и подобрались к моменту истины — восстановлению данных. Ведь все понимают, что сделанный бэкап, который нельзя восстановить, равносилен несделанному бэкапу?

Перед восстановлением было бы неплохо проверить архив:

    dar -t /mnt/backup/archive_sunday

Если dar не вернул код ошибки (в конце man'а перечислены все возможные коды выхода, которые dar может вернуть), то можно восстанавливать:

    mkdir sunday
    dar -x /mnt/backup/archive_sunday -R sunday

### Операции на удалённых машинах

#### Восстановление файлов

Я вскользь уже упоминал, что восстановление файлов с удалённых машин, через пайп (например, по ssh) является нетривиальной задачей.

Попробую подробно рассказать как это работает.

Все сложности связаны с тем, что для восстановления одного файла dar нужно читать индекс. Если же использовать его аналогично tar, в режиме потокового чтения (ключ --sequential-read), то таких проблем не возникает.

Для решения проблемы с чтением индекса создано две версии dar:

 * Основная: dar, которая говорит, что нужно восстановить.
 * Вспомогательная: dar_slave, которая принимает команду и передаёт dar восстановленные данные, которые dar потом записывает на диск.

Поэтому схема работы (для восстановления) выглядит так:

    (2) --> dar --> (1) --> dar_slave archive --> (2)

 1. dar через пайп говорит dar_slave: "Хочу восстановить файл А".
 2. dar_slave считывает индекс файла archive, узнаёт, по какому смещению находится искомый файл, и передаёт его на stdout, который читает dar и пишет полученный файл на диск.

Сложность заключается в том, чтобы осуществить передачу файла от dar_slave в dar. Для такой «кольцевой» передачи данных нам придётся соорудить небольшой костыль при помощи mkfifo:

    mkfifo /tmp/fifo
    dar -x -i /tmp/fifo -R sunday | ssh user@host dar_slave sunday > /tmp/fifo
    rm /tmp/fifo

Этих проблем можно избежать, если монтировать удалённую директорию, например, по NFS.

#### Упаковка файлов

При упаковке файлов тоже есть небольшая хитрость: нужно обязательно сохранять файл индекса архива на локальной машине, чтобы на его основе можно было строить инкрементальные архивы:

    dar -R $HOME -c - -A /mnt/backup/CAT_archive_wednesday -@ /mnt/backup/CAT_archive_thursday | ssh user@host 'cat > archive_thursday'


## Разные приятности

Также в комплекте с dar идёт утилита под названием dar_manager, которая является обёрткой на стероидах над dar. По сути, это приложение, которое, оперируя полученными из архивов индексами, позволяет упростить жизнь при восстановлении данных (например, не придётся копаться в сотне архивов, чтобы найти, откуда можно будет восстановить нужный файл).

Я ей не особо пользовался, только запускал пару раз, чтобы понять для чего и как она используется.

Единственное, что я, возможно, не советовал бы: использовать её в качестве production-решения, поскольку я частенько в списках рассылки встречал сообщения о том, что файл в котором она хранит индексы бьётся, что, впрочем, никак не влияет на сохранность самих данных.

Также в комплекте с dar идёт dar_static: статически скомпилированый бинарник, который никогда не будет лишним положить рядом с архивами.

## Важное замечание

Поскольку утилита достаточно активно разрабатывается, у неё есть периодически возникающие проблемы (которые оперативно решаются в списке рассылки), и в связи с этим в дистрибутивах почти всегда присутствует неактуальная версия dar. Например в Ubuntu 12.04 используется, если не ошибаюсь dar версии 2.4.2, которая не может создать/восстановить архив в некоторых специфичных условиях. С dar версии 2.4.12 лично у меня никаких проблем нет.

Также стоит отметить, что архивы, сделанные версией 2.4, скорее всего не будут распаковываться dar версии 2.3 ввиду изменения формата архива.