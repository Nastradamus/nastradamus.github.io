---
layout: post
title: "Как собрать nginx ingress controller старой версии и пропатчить его"
---

В данном примере мы исправим багу в древней версии v0.20.0 и научимся работать с зависимостями Go старых версий через dep.  

## Проблема

ingress-nginx версии v0.20.0 добавляет лишние слэши при rewrite.

Что мешает бесшовной миграции на последнюю версию (v0.32.0):  
Разработчикам в ряде случаев пришлось делать такую конструкцию:

```
path: /service/api/v1/tokens
rewrite-target: /api/v1/tokens
```

Если добавить в конец / в yaml объекта типа `Ingress`, то ingress-nginx 0.20.0 начинает делать rewrite в `/api/v1/tokens//`.
Отказаться от такой конструкции не получается, так как кто-то ходит в сервис и со слэшом на конце и без.

Было решено пропатчить старый nginx ingress, чтобы избавиться от досадного бага. А далее, уже двигаться по плану:

- Менять сначала пути и рерайты в ингрессах на человеческие (со слэшом на конце)
- Мигрировать на новый ingress controller, запущенный на другом порту

Но сначала, надо исправить проблему в исходниках контроллера старой версии.

## Цель

Переменную `location.Rewrite.Target` измнить на `strings.TrimSuffix(location.Rewrite.Target, "/")`  

Здесь в этих строках:  
https://github.com/kubernetes/ingress-nginx/blob/nginx-0.20.0/internal/ingress/controller/template/template.go#L539

## Поехали!

## Скачиваем исходники

```bash
go get k8s.io/ingress-nginx
cd ~/go/src/k8.io/ingress-inginx
```

## Переключаем на бранч, который нужно пофиксить

```
git checkout v0.20.0
```

## Правим архитектуру в Makefile чтобы не собирать лишнее 40 минут

```
vim Makefile
```

(или `sed -i -e  's/^ALL_ARCH.*/ALL_ARCH = amd64/g'`)

## Вносим _ свои _ правки:

```
vim internal/ingress/controller/template/template.go +539
```

(здесь меняем переменную - см. "Цель")

## Фиксим образ для сборки go:

```
vim build/go-in-docker.sh
```

А именно:  
- меняем `E2E_IMAGE` на `docker..ru/ops/golang:1.10.7-alpine3.8-v7`
- удаляем в этом же файле ентрипоинт: `--entrypoint ${FLAGS}`

## Сохраняем правки

```
git add -A
git commit -m "Fix trailing slash bug"
```

## Устанавливаем зависимости для сборки

```
dep ensure =v  # ключевой момент, без этого будет ругаться на internal инклуды
```

## Запускаем сборку и ждём (минут 5)

```
make
```

```bash
tagged quay.io/kubernetes-ingress-controller/nginx-ingress-controller-amd64:0.20.0
```

## Вешаем тэг и пушим:

```
docker tag quay.io/kubernetes-ingress-controller/nginx-ingress-controller-amd64:0.20.0 docker.myregistry.com/ops/nginx-ingress-controller:0.20.0-patched-2

docker push docker.myregistry.com/ops/nginx-ingress-controller:0.20.0-patched-2
```

Готово!


