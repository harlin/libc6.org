Title: Теория и практика IPsec
Slug: teorija-i-praktika-ipsec
Category: Администрирование
Date: 2012-08-25 17:10
Source: False

## Терминология.

Поскольку вопросы связанные с шифрованием достаточно сложны (по крайней мере для меня), то я считаю нужным ввести несколько терминов, без твёрдого знания которых будет невозможно понимание предмета вообще.

 * Аутентификация - проверка подлинности, например по паролю, контрольной сумме файла и пр.
 * Авторизация - предоставление прав доступа субъекту или группе субъектов на определённые действия.
 * Идентификация - установление субъекта по имеющемуся идентификатору.

## Введение.

Везде и повсюду упоминается IPsec только вместе с L2TP, причём в таком свете, что эти два понятия становятся неотделимы, что абсолютно неверно.

Это как спрашивать "что лучше - солнце или сладкое?". IPsec (или IP security) вещь в себе и может работать (и, собственно, работает) отдельно от L2TP. Если совсем просто, то мы берём пакетик, который хотим отправить, шифруем его, отправляем, а принимающая сторона, которая знает ключ - расшифровывает его. И есть два типа соединений: туннель и транспорт. Их отличие заключается в том, что в случае туннеля шифруется весь пакет, а в случае транспорта - пакет за исключением заголовков (из-за этого нет проблем с NAT, поскольку NAT модифицирует заголовки).

Немного об алгоритме шифрования. Аутентификация происходит следующим образом:

  1. Клиент посылает серверу кодовую фразу, по которому сервер опознаёт клиента за своего, а клиент - сервер.
  2. Клиент и сервер генерируют сертификаты и пересылают их друг другу по протоколу IKE и периодически их обновляют (что-то около суток).
  3. Между серверами создаётся туннель по которому начинает передаваться защищённый трафик.

Причём, что характерно, в Linux есть две реализации IPsec: встроенная в ядро (NETKEY) и поставляемая парнями из [OpenSwan](https://www.openswan.org) (KLIPS). Чтобы понять разницу между реализациями можно почитать эту [статью](http://www.installationwiki.org/Openswan). Там же указаны рекомендации по поводу того, что именно нужно использовать в каждом конкретном случае. Я для себя остановился на встроенном варианте, поскольку это требует меньше времени на поддержку.

Если решите использовать KLIPS, вы будете также привязаны непосредственно к OpenSwan, поскольку только эта программа умеет работать с KLIPS. Есть несколько аналогов прикладного ПО, которое позволяет конфигурировать IPsec: [racoon(ipsec-tools)](http://ipsec-tools.sourceforge.net), [StrongSwan](http://www.strongswan.org). Они тоже хороши, а racoon, на мой взгляд, намного проще в кофигурации (хотя авторы статьи считают по другому), но я остановился на OpenSwan.

То что написано далее - не претендует на абсолютную истину и может содержать неточности. Я стараюсь написать попроще, а аккуратно и попроще - разные вещи. Многое я беру из [Openswan: Building and Integrating Virtual Private Networks](http://www.amazon.com/Openswan-Building-Integrating-Networks-ebook/dp/B008GT1AI8). Так что, если вы уже читали её - то нового вы ничего не узнаете.

## Базовые понятия криптографии в IPsec.

### Алгоритм Диффи-Хеллмана.

Про него я уже упоминал в рамках статьи по [настройке OpenVPN](//libc6.org/page/nastrojka-servera-openvpn-v-debian), но не рассказывал как это работает.

Я думаю вам будет лень идти в википедию и прочитать. Расскажу в двух словах:

1. Две стороны знают некие два числа _p_ и _g_.
2. Каждая сторона генерирует большое случайное число: _a_ и _b_
3. Обе стороны вычисляют числа _A=g^a <abbr title="">mod</a> p_ и _B=g^b mod p_ и пересылают друг другу.
4. Имея на рука: _a,A,B,p,g_ и _b,A,B,p,g_ вычисляются _B^a mod p = s_ и _A^b mod p = s_. _s = g^ab mod p_ и оно одинаково у обоих сторон.

В практических реализациях, для _a_ и _b_ используются числа порядка 10^100 и _p_ порядка 10^300. Число g не обязано быть большим и обычно имеет значение в пределах первого десятка.

Этот алгоритм применяется только, если передающиеся данные нельзя изменить.

### Обмен публичными ключами при помощи IKE

#### Фаза 1. Создание ISAKMP SA.

Суть её заключается в том, что компьютеры устанавливают между собой связь и пытаются удостоверится, что их сообщения никто не прослушивает. Далее они выполняют проверку другой стороны, основываясь на типе и содержимом идентификатора который они получили, для того чтобы предотвратить атаку MITM. То, что содержится в идентификаторе может существенным образом отличаться, но обычно это хостнейм, IP, почтовый адрес и ASN.1 DN. Внешняя верификация может происходить несколькими путями: PSK или RSA.

Существуют два режима идентификации: основной и аггресивный. Причём аггресивный менее предпочтителен.

##### Основной режим.

В основном режиме происходят 3 обмена: в первом узлы договариваются о правилах, во втором обмениваются открытыми значениями Диффи-Хеллмана и вспомогательными данными, в третьем происходит подтверждение обмена Диффи-Хеллмана.

##### Аггресивный режим.

В агрессивном режиме в первом обмене устанавливаются правила, передаются открытые значения Диффи-Хеллмана и вспомогательная информация. Причем во втором сообщении первого обмена происходит идентификация отвечающей стороны (происходит открытая передача хэша PSK, что позволит злоумышленику сбрутфорсить его). Третье сообщение идентифицирует инициатора и подтверждает участие в обмене. Последнее (четвертое) сообщение может быть не послано.

#### Фаза 2. Быстрый режим.

В быстром режиме осуществляется заполнение SA (см. ниже) используя, уже организованое в фазе 1, шифрование.

## Базовые понятия IPsec.

Введение введением, но для понимания того как именно работает IPsec необходимо ввести несколько базовых понятий.

### Протоколы.

Важно понимать, что номер протокола != номер порта и то, что Вы разрешили на фаерволле прохождения пакетов на порту - не гарантирует того, что пакет пройдёт. Яркий пример другого протокола ICMP, который тоже не имеет порта.

#### Authentication Header (AH).

Протокол номер 51 (посмотреть протоколы можно в /etc/protocols). Он не аутентфицирует заголовок IP пакета, как можно бы было подумать из названия, он аутентифицирует данные передаваемые в пакете, но не шифрует их.

##### Security Parameter Index (SPI).

Когда два компьютера используют приватные ключи для соединения при помощи протокола AH они договариваются о каком-то уникальном номере, с помощью которого они будут идентифицировать подключение.

##### Sequence Number (SN).

Компьютеры устанавливают SN в 0. И для каждого пакета, который они посылают SN увеличивается на один для того, чтобы предотвратить атаку при помощи повторения. В атаке при помощи повторения, злоумышленик получает зашифрованый пакет и без возможности расшифровать данные пересылает его снова и снова. Без SN два компьютера не смогут отличить исходный пакет от копии предыдущего пакета до тех пор пока криптографическая проверка не покажет, что этот пакет копирует предыдущий. Тем не менее это был бы валидный пакет. Таким образом при помощи SN пакет может быть однозначно идентифицирован и при передаче старого пакета он будет отклонён.

##### Integrity check value (ICV).

Контрольная сумма, которая считается для пакета, использует секретный ключ, который знает и отправитель и получатель.

#### Encapsulated Security Payload (ESP).

Протокол номер 50. Этот отличается от AH немногим, он также аутентифицирует пакет, но и добавляет различные политики безопасности, и может даже зашифровать их.


### Режимы работы.

Есть два режима передачи данных: транспорт и туннель. Транспорт это способ сообщения двух хостов между собой, а туннель позволяет обернуть пакет IPsec в IP пакет и отправить. В общем, транспортный режим полезен, только если у вас уже есть какой-то способ инкапсуляции трафика (пресловутый L2TP, например), но в общем случае гораздо более полезным будет использование туннеля.

Почти всегда использование IPsec означает, что Вы будете использовать ESP в туннельном режиме. То есть вы будете, во-первых шифровать пакеты, а во-вторых сможете транслировать одну подсеть в другую.

### Security Policy Database (SPD).

База данных политик безопасности, определяет какие именно пакеты должны или не должны быть обработаны. Например можно сказать, что не нужно принимать никакого незащищённого трафика с этого хоста; при отправке пакета на хост, при отсутствующем IPsec соединении нужно сначала это соединение установить.

### Security Association (SA).

Для того, чтобы обрабатывать пакет ядро он должно знать такие параметры как: SPI, режим работы, ключи и SP, которые к ним применяются. И, поскольку защищённые соединения являются полудуплексными, то нужно иметь два канала SA. И все эти параметры хранятся в базе данных Security Associations Database (SAD). В итоге образуется что-то вроде фаерволла.





*[mod]: Два целых числа a и b сравнимы по модулю натурального числа n (или равноостаточны при делении на n), если при делении на n они дают одинаковые остатки
*[ISAKMP]: Internet Security Association and Key Management Protocol. RFC 2408.
*[MITM]: Man in the middle.
*[ASN.1]: ASN.1 (англ. Abstract Syntax Notation One) — в области телекоммуникаций и компьютерных сетей язык для описания абстрактного синтаксиса данных (ASN.1), используемый OSI. Например: C=CA,L=Toronto,O=Xelerance ,CN=lists.openswan.org, E=paul@xelerance.com
*[PSK]: Preshared Key
*[RSA]: Криптографический алгоритм с открытым ключом