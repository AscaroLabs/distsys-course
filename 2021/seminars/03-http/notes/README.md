# Задача 3-http

## Структура рабочей директории

- server
  - \__init__.py
  - Dockerfile
  - server.py
  - start 
- tests
  - Dockerfile
  - test_classes.py
  - test_delete.py
  - test_get.py
  - test_post.py
  - test_put.py
- conftest.py
- docker-compose.yml
- Dockerfile
- file_state.py
- grader.py
- http_builder.py
- README.md

## server

\__init__.py пустой, код только в `server.py`. Описываются классы `HTTPRequest`, `HTTPResponce`, `HTTPServer`, `HTTPHandler`.

Подключаем логгирование
```python
import logging
```
Модуль для работы с путями
```python
import pathlib
```
Data-классы
```python
import dataclasses
```
Socket-сервер
```python
from socketserver import BaseRequestHandler
from socketserver import StreamRequestHandler
from socketserver import ThreadingTCPServer
```
Пакет для command line интерфейсов
```python
import click
```

