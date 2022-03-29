---
title: Сброс кэша для файла на jsDelivr
slug: purge-jsdelivr-cache
date: 2022-03-29T07:29:19Z
expirydate: 2024-09-30T07:29:19Z
#draft: true
toc: false
image: cover.jpg
categories: [dev]
tags: [cdn, cache]
---

[jsDelivr][jsdelivr] это потрясающий проект для тех, кто желает доставлять контент до пользователей максимально быстро, но при этом иметь возможность прозрачно управлять версионированием.

Что я имею в виду под этим? Допустим, у вас есть некоторый проект, основной ценностью которого является сам контент этого проекта. Это могут быть некоторые файлы шрифтов, картинки, иконки, js/json файлы - вообще всё что угодно. Главное, что этот контент храниться в едином месте - у вас в репозитории (скорее всего на GitHub), а не располагается у самих пользователей. Т.е. за ним пользователям нужно "ходить", и данная модель имеет очевидные плюсы и минусы. Плюсы - централизованное управление, простота обновления (не надо обновлять самих клиентов), прозрачная история изменений. Минусы же - единая точка отказа, и за контентом нужно каждый раз "ходить" по сети.

Если вы выбрали эту модель для доставки контента клиентам и придерживаетесь семантического версионирования, то сразу же после первого релиза проекта встанет вопрос - а как же сделать так, чтоб по определенной ссылке можно было получать контент соответствующий определенной мажорной/минорной версии? Т.е. чтоб после выпуска минорного обновления, не было необходимости ходить за ним по новой ссылке. И тут как раз на помощь и приходит jsDelivr с его возможностью доставлять контент прямо из [GitHub репозитория в соответствии с версионированием](https://www.jsdelivr.com/features#gh):

```text
https://cdn.jsdelivr.net/gh/jquery/jquery@3/dist/jquery.min.js
                         ^^ ^^^^^^^^^^^^^ ^ ^^^^^^^^^^^^^^^^^^
                         |  |             | └ путь до файла в репозитории, т.е. "dist/jquery.min.js"
                         |  |             └ "смотреть" в мажорную версию "3", т.е. "3.*"
                         |  └ имя репозитория на GitHub, т.е. "https://github.com/jquery/jquery"
                         └ указатель на то, что это GitHub репозиторий
```

И всё вроде бы классно, но после минорного релиза, контент по ссылке не обновится в ту же секунду, и инвалидация кэша на стороне CDN может занять длительное время.

Мои усилия по поиску решения этой проблемы не нашли ответа в официальной документации и StackOverflow, но случайно попался рецепт, что неожиданно сработал. Оказалось, что для того, чтоб сбросить кэш для определенного файла достаточно отправить http `GET` запрос по тому же URI файла, но с заменой домена `cdn.jsdelivr.net` на `purge.jsdelivr.net`. Просто и гениально 😉

Т.е. если ссылка, по которой контент "читается" `https://cdn.jsdelivr.net/gh/jquery/jquery@3/dist/jquery.min.js`, то для сброса кэша по нему достаточно стукнуться по URL `https://purge.jsdelivr.net/gh/jquery/jquery@3/dist/jquery.min.js`:

```bash
$ curl -s https://purge.jsdelivr.net/gh/jquery/jquery@3/dist/jquery.min.js
{
  "id": "0vABmLhmJ0EhYxDN",
  "status": "finished",
  "timestamp": "2022-03-29T08:20:01.256Z",
  "paths": {
    "/gh/jquery/jquery@3/dist/jquery.min.js": {
      "throttled": false,
      "providers": {
        "fastly": true,
        "bunny": true,
        "cloudflare": true,
        "gcore": true,
        "quantil": true
      }
    }
  }
}
```

В контексте GitHub actions это может выглядеть так:

```yaml
name: release

on:
  release: # Docs: <https://git.io/JeBz1#release-event-release>
    types: [published]

jobs:
  purge-cdn-cache:
    name: Purge jsDelivr CDN cache
    runs-on: ubuntu-20.04
    steps:
      - uses: fjogeleit/http-request-action@v1 # Action page: <https://github.com/fjogeleit/http-request-action>
        with: {method: 'GET', url: 'https://purge.jsdelivr.net/gh/jquery/jquery@3/dist/jquery.min.js'}
```

[jsdelivr]:https://www.jsdelivr.com/
