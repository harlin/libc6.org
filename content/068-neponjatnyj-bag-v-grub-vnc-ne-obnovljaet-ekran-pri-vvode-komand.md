Title: Непонятный баг в GRUB. VNC не обновляет экран при вводе команд.
Slug: neponjatnyj-bag-v-grub-vnc-ne-obnovljaet-ekran-pri-vvode-komand
Category: Администрирование
Date: 2012-06-24 17:53
Source: False

Итак, имеем виртуальную машину, с Ubuntu 10.10, обновляем её до 11.04, перезагружаем и не можем ввести через VNC консоль ни одну команду. Экран как будто бы заморожен. Открываем новую сессию - видим результат ввода.

Бага в трекерах Ubuntu/KVM сходу не нашёл, потом заведу если не найду.

Лечится так:

Необходимо раскомментировать в файле **/etc/default/grub** строку:

    GRUB_TERMINAL=console

Выполнить команду:

    # update-grub

Перезагрузить виртуальную машину:

    # reboot

После этого GRUB не пытается создать графическую консоль и всё работает хорошо.

UPDATE:

Чтобы сразу получить доступ к консоли можно выполнить следующее:

 1. В самом начале загрузки grub в течение 30 секунд ожидает ввода оператора. Необходимо в этот момент нажать <Esc>.
 2. В приглашении нужно набрать:
nofb<Enter>
 3. Система начнет загружаться в текстовом терминале, обновление экрана будет сразу отображаться.

Спасибо @cronfy.