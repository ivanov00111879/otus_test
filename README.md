Обновление ядра системы

Задание
1. Запустите ВМ c Ubuntu.
2. Обновите ядро ОС на новейшую стабильную версию из mainline-репозитория.

Подключаемся к ВМ с ОС Ubuntu и проверяем текущую версию ядра командой uname -r 
<img width="682" height="324" alt="image" src="https://github.com/user-attachments/assets/b8d5f3f4-e9bc-4c77-acb7-0febf37e7f49" />


Далее в браузере переходим в репозиторий https://kernel.ubuntu.com/mainline, находим свежую версию ядра для нашей архитектуры. На текущий момент последняя версия 7.1.2. Переходим в каталог и находим ПО для нужной нам архитектуры: 
Скрин 11

Возвращаемся в ВМ, создаем каталог командой mkdir kernel && cd kernel и загружаем ПО по ссылкам из репозитория командами:
wget https://kernel.ubuntu.com/mainline/v7.1.2/amd64/linux-headers-7.1.2-070102-generic_7.1.2-070102.202606271039_amd64.deb
wget https://kernel.ubuntu.com/mainline/v7.1.2/amd64/linux-headers-7.1.2-070102_7.1.2-070102.202606271039_all.deb
wget https://kernel.ubuntu.com/mainline/v7.1.2/amd64/linux-image-unsigned-7.1.2-070102-generic_7.1.2-070102.202606271039_amd64.deb
wget https://kernel.ubuntu.com/mainline/v7.1.2/amd64/linux-modules-7.1.2-070102-generic_7.1.2-070102.202606271039_amd64.deb

Устанавливаем все пакеты сразу:
sudo dpkg -i *.deb 

Проверяем, что ядро появилось в /boot.
ls -al /boot

Перезагружаем ВМ и проверяем текущую версию ядра командой uname -r 
Скрин 2
