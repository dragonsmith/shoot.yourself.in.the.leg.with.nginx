# Как сломать себе Nginx

## Краткое описание
1. Берем upstream

```nginx
upstream bad {
  server backend1;
  server backend2;
}
```

по факту - это краткая запись вот такого апстрима, на самом деле:

```nginx
upstream nya {
  server backend1 max_fails=1 fail_timeout=10s;
  server backend2 max_fails=1 fail_timeout=10s;
}
```

Это переводится на русский язык так: если апстрим возвращает ошибку (например 500), то он помечается, как битый, на 10 секунд и на него 10 секунд не идут запросы. Вообще.

2. А теперь внимание за руками. У нас стояла настройка `proxy_next_upstream http_500;` Переводим на русский опять: “Если один апстрим вернул ошибку - спроси у другого”. Что, в целом, правильно.

3. Но!!! Вдруг появились такие запросы, которые вызывали у приложения 500 и это была "норма". Их было всего пара, но они были. Результат? Каждый такой запрос тормозил нам nginx на 10 секунд. Fatality.

## Пример

1. Запускаем:
```shell
tmux new-session -d 'docker-compose up'
tmux split-window -v
tmux -2 attach-session -d
```

2. Дожидаемся, когда в верхней консоли запустить `docker-compose`  и в нижнюю консоль вводим

Пример сломанной конфигурации:
```shell
curl -o /dev/null -s -w "Response Code: %{http_code}\n" localhost:8080/error;
curl -o /dev/null -s -w "Response Code: %{http_code}\n" localhost:8080/error;
curl -s -w "Response Code: %{http_code}\n" -o /dev/null localhost:8080;
```

Пример работающей конфигурации:
```shell
curl -o /dev/null -s -w "Response Code: %{http_code}\n" localhost:8081/error;
curl -o /dev/null -s -w "Response Code: %{http_code}\n" localhost:8081/error;
curl -s -w "Response Code: %{http_code}\n" localhost:8081
```