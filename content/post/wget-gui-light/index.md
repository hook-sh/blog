---
title: Web-gui для wget (light)
slug: wget-webgui-wide
date: 2014-08-25T16:23:00+00:00
aliases: [/dev/wget-gui-light, /dev/wget-webgui-wide]
image: featured-image.jpg
categories: [dev]
tags:
  - ajax
  - bash
  - css
  - gui
  - habr
  - linux
  - php
  - wget
---

> Данная статья является копией [публикации на хабре](https://habr.com/post/234353/)

Ранее здесь находилось описание возможных ситуаций, когда данное решение могло бы вам понадобиться, но давайте его опустим. Возможность удобного создания удаленных закачек, которые выполняются привычным wget-ом (можно спокойно увидеть их список при помощи **ps**), с отображением прогресса — идея не новая. И даже есть [некоторые][1] [решения][2], но не актуальные, так как более 5 лет никем не поддерживаются.

<!--more-->

Для торрентов всё просто и тривиально — ставим Transmission или любой аналогичный клиент с веб-мордой. Но для ссылок на простые файлы/страницы нужно что-то своё. Вот короткий список задач, которые меня подтолкнули к написанию оного:

- Смотрю фильм онлайн при помощи планшета, но появляются дела и надо бы его сохранить, чтоб досмотреть позже;
- На удаленный сервер надо скачать файл, и приходится запускать терминал каждый раз;
- Надо бы скачать образ свежего linuxmint, но на домашний NAS, а не ноутбук, работая за которым пришла эта идея;
- Во время серфинга часто возникает задача сохранить файл и расшарить его.

Если вам стало интересно — добро пожаловать под кат:

#### Системные требования

Веб-интерфейс построен на Javascript + css3 с клиентской стороны, и php (выбран как наиболее популярный) с серверной. Для полноценной работы потребуется:

- `*nix` (по крайней мере писалось именно под эту платформу, для запуска под другой потребуются рабочие порты `wget`, `ps` и `kill`);
- `php5.x` (скорее всего работать будет и на php4.x, по к моменту публикации это протестировано не было);
- Браузер с поддержкой `Javascript` (и очень желательно — с поддержкой `css3`).

#### Особенности и настройки серверной части

Как уже было сказано выше — в роли серверной части выступает скрипт, написанный на php. Он выполняет следующие задачи:

- Получение информации о запущенных задачах;
- Отмена запущенных задач;
- Добавление новых задач;
- Возвращение результатов в JSON формате.

Для своей работы ему требуются `ps`, `wget` и `kill` соответственно. Для получения значения состояния закачки (на сколько процентов завершена) используется следующий алгоритм:

- Задачи wget запускаются с флагами `--background` и `--progress=bar:force`;
- Вывод лога загрузки производится в файл, установленный в параметре `--output-file=FILE`;
- При запросе состояния задач с помощью `ps -ax` получаем путь к файлу, установленный в `--output-file=FILE`;
- Читаем крайнюю строку этого файла, получая регуляркой из него искомое значение.

Путь для временных лог-файлов устанавливается в строке

```php
define("tmp_path", "/tmp");
```

Путь до директории, в которую будет происходить сохранение всех загружаемых файлов устанавливается в строке

```php
define("download_path", BASEPATH."/downloads");
```

Доступно удобное указание путей до `ps`, `wget` и `kill`. Для этого достаточно убрать комментарий вначале строки и указать свой путь, например:

```php
define("wget", "/usr/bin/wget");
```

Возможна установка ограничения на скорость закачки из секции настроек. За это отвечает строка

```php
define("wget_download_limit", "1024");
```

Если оставить её закомментированной — никакого ограничения не будет.

Для определения в списке задачи запущенной через GUI от любой другой используется определенный флаг, уникальный для GUI. Он установлен в строке:

```php
define("wget_secret_flag", "--max-redirect=4321");
```

И его менять без необходимости не надо. Кстати, если хотите чтоб другие ваши задачи, запущенные из терминала отображались в веб-интерфейсе — достаточно как раз этот параметр к ним и добавить. Но не забывайте, что есть ещё и некоторые другие параметры, не менее обязательные (в зависимости от настроек).

Постарался написать с максимально экономичным отношением к ресурсам и минимальной зависимостью от системы, но большим опытом в php не обладаю, поэтому — буду признателен рекомендациям по оптимизации.

Скрипт отвечает как на POST, так и GET запросы. Разницы между ними нет. Так же отвечает на параметры, переданные в командной строке. Например: `php ./rpc.php get_list` или `php ./rpc.php add_task http://goo.gl/5Qi0Xs`.

##### Пример POST-запроса и Json ответа:

```bash
Request:
192.168.1.2/wget/rpc.php?action=add_task&url=http://mirror.yandex.ru/linuxmint/stable/17/linuxmint-17-cinnamon-dvd-64bit.iso
Answer:
{
    status: 1,
    msg: "Task added",
    id: 10910
}

Request:
192.168.1.2/wget/rpc.php?action=get_list
Answer:
{
    status: 1,
    msg: "Active tasks list",
    tasks: [
        {
            url: "mirror.yandex.ru/linuxmint/stable/17/linuxmint-17-cinnamon-dvd-64bit.iso",
            progress: 95,
            id: 10910
        }
    ]
}
```

#### Особенности и настройки клиентской части

Не используются «новые html5 теги», но используются свойства css3 для оформления прогресс-бара загрузок и адаптива. Дизайн выполнен в минималистичном стиле. При отсутствии задач в центре страницы располагается поле для добавления адреса закачки, если задачи имеются — это поле смещается вверх страницы, и ниже располагаются задачи.

Все запросы — асинхронные (без перезагрузки страницы). Дизайн страницы — адаптивный:

![Screenshot](https://hsto.org/files/479/aec/f73/479aecf737f647e485fb325671fe6df5.png)

Изменение состояния отображается также в заголовке вкладки (окна):

![Screenshot](https://hsto.org/files/e8c/78d/52f/e8c78d52ff494d0b9f6d7aa56bf957b8.png)

В нижней части страницы располагается javascript-закладка («Download this»), переместив которую в панель закладок браузера можно одним кликом добавлять новые задачи (при клике будет добавлена активная вкладка; если открыта вкладка с видео-файлом и будет нажата эта «закладка» на панели закладок — будет добавлена задача на скачивание этого видео-файла):

![Screenshot](https://hsto.org/files/462/4d0/f9c/4624d0f9c3494fee9987e4ba757a423b.png)

Весь javascript код документа расположен в файле core.js. В верхней его части располагаются основные настройки.

Описывать функциональные моменты смысла особого не вижу, но скажу — функции разделены на логические группы, скрипт **не** минифицирован, комментарии имеют место быть.

При нажатии на F5 происходит принудительное обновление задач, страница перезагружается только по нажатию на кнопку обновления в браузере.

#### Установка

- Скачать или склонировать крайнюю версию репозитория;
- Распаковать в директорию, доступную «извне»;
- Изменить путь в «rpc.php»: `define("download_path", BASEPATH."/downloads");`
- Открыть в браузере, проверить работоспособность. В случае возникновения ошибок — [задайте вопрос, сопроводив его всей отладочной информацией][3].

#### Ссылки

- [Скачать](https://github.com/tarampampam/wget-gui-light/archive/master.zip)
- [GitHub](https://github.com/tarampampam/wget-gui-light)

[1]:http://exir.ru/wget4web/screen.htm
[2]:http://sourceforge.net/projects/download-webgui/files/?source=navbar
[3]:https://github.com/tarampampam/wget-gui-light/issues/new