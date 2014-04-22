Title: Настройка Network Manager для работы OpenVPN в Ubuntu 12.04.
Slug: openvpn-network-manager-ubuntu
Category: Администрирование
Date: 2012-10-30 02:37
Source: False

Казалось бы, что может быть проще - поставить пакет и настроить подключение в Network Manager.

Но нет, и тут есть нюансы. При использовании [дефолтного конфига OpenVPN](//libc6.org/page/nastrojka-servera-openvpn-v-debian), Network Manager начинает прописывать какие-то совершенно адские маршруты из-за чего даже VPN гейт пропинговать не получается.

Я нашёл способ как обойти эту проблему. Я не буду рассказывать по пунктам куда и чего, там всё достаточно прозрачно сделано, просто указываем сертификаты, адрес сервера и всё.

В настройках подключения нужно выбрать: _Автоматически (VPN, только адрес)_

В разблокированном поле указываем DNS серверы.

Сохраняем, подключаемся и у нас всё работает.