# Тестовое задание

## 1. Вопросы для разогрева
1. Расскажите, с какими задачами в направлении безопасной разработки вы сталкивались?

- Разработкой занимался я не так много, в основном это были FullStack приложения, изучая разработку которых я из интереса пробовал писать небольшие функции для санитизации приходящей с клиента информации.

2. Если вам приходилось проводить security code review или моделирование угроз, расскажите, как это было?

- Как я описал, максимально похожим было написание функций для санитизации информации, соответственно я моделировал, что может прислать пользователь, что может ему позволить сломать что-то и заиметь контроль над моим сервером. Все сводилось к тому, что из строк удалялись все символы, которые могли привести к вызову опасного кода (например для SSTI). Также немного погружался в Reverse Engineering, где искал уязвимости кода в дизассемблированных приложениях.

3. Если у вас был опыт поиска уязвимостей, расскажите, как это было?

- Да, был. Изучая тестирование на проникновение я в основном использую HackTheBox, иногда что-то нахожу сам (например нашел уязвимость на сайте Тинькофф). Обычно уязвимости, с которыми я сталкивался, которые сводились к мисконфигурациям, устаревшим версиям, неправильному обращению с данными и отсутствием правильной обработки ввода. Соответственно отталкиваясь от них я уже использовал различные инструменты (Burpsuite, hashcat, wfuzz, etc), POCs или писал скрипты для получения контроля над системами.

4. Почему вы хотите участвовать в стажировке?

- Считаю этот опыт крайне интересным и полезным, позволяющим понять как происходит работа отдела безопасности в большой компании и способным указать на то, как мне лучше повышать свои компетенции.


## 2. Security code review

### Часть 1. Security code review: GO

```golang
package main

import (
    "database/sql"
    "fmt"
    "log"
    "net/http"
    "github.com/go-sql-driver/mysql"
)

var db *sql.DB
var err error

func initDB() {
    db, err = sql.Open("mysql", "user:password@/dbname") // CWE-798
    if err != nil {
        log.Fatal(err)
    }

	err = db.Ping()
	if err != nil {
		log.Fatal(err)
	}
}

func searchHandler(w http.ResponseWriter, r *http.Request) {
	if r.Method != "GET" {
		http.Error(w, "Method is not supported.", http.StatusNotFound)
		return
	}

	searchQuery := r.URL.Query().Get("query")
	if searchQuery == "" {
		http.Error(w, "Query parameter is missing", http.StatusBadRequest)
		return
	}

	query := fmt.Sprintf("SELECT * FROM products WHERE name LIKE '%%%s%%'", searchQuery) // CWE-89
	rows, err := db.Query(query)
	if err != nil {
		http.Error(w, "Query failed", http.StatusInternalServerError)
		log.Println(err)
		return
	}
	defer rows.Close()

	var products []string
	for rows.Next() {
		var name string
		err := rows.Scan(&name)
		if err != nil {
			log.Fatal(err)
		}
		products = append(products, name)
	}

	fmt.Fprintf(w, "Found products: %v\n", products)
	}

func main() {
    initDB()
    defer db.Close()

http.HandleFunc("/search", searchHandler)
fmt.Println("Server is running")
log.Fatal(http.ListenAndServe(":8080", nil))
}
```

- В данном коде у нас пристутсвуют две уязвимости: в строке 15 (CWE-798 или Use of Hard-coded credentials) и в строке 38 (CWE-89 или Improper Neutralization of Special Elements). 

Первая уязвимость может привести к тому, что код может случайно куда-то утечь и вместе с ним утекут и данные для авторизации в БД, что приведет уже к утечке данных из БД и/или удалению данных/подмены данных и т.д.
Решить эту проблему можно использованием шифрования и хеширования и вынесения данных в отдельные файлы (.env например)

Вторая уязвимость заключается в отстутствии правильной санитизации входящих данных, что может привести к Blind SQL иньекции и более серьезным последствиям. Например если мы используем следующий пэйлоад: "smth' AND SUBSTRING((SELECT Password FROM Users WHERE Username = 'Administrator'), 1, 1) > 't';--, то при условии, что данные из таблицы правильные, меняя букву мы можем подобрать пароль от аккаунта администратора.
Решить эту проблему можно, сделав из обычного запроса подготовленный SQL запрос или же добавив промежутучную функцию для санитизации данных. Первый вариант будет предпочтительнее, так как избегается любая возможность SQL иньекций, поскольку база данных будет обращаться с инпутом исключительно как с данными. 

Первый вариант будет выглядеть примерно вот так: "rows, err := db.Query("SELECT * FROM products WHERE name LIKE '%?%'", searchQuery)"

### Часть 2: Security code review: Python

Пример 1.

```python
from flask import Flask, request
from jinja2 import Template

app = Flask(name)

@app.route("/page")
def page():
    name = request.values.get('name')
    age = request.values.get('age', 'unknown')
    output = Template('Hello ' + name + '! Your age is ' + age + '.').render() // CWE-1336
return output

if name == "main":
    app.run(debug=True)
```

В данном примере у нас классический случай SSTI,  данные из реквеста сразу попадают на рендер и вложенный вредоносный код будет исполняться. Для эксплуатации мы можем использовать класс 'object' и 
"__ subclasses __" , с помощью которых мы получим доступ к чтению и записи файлов внутри системы. Пример {{request.application.__globals__.__builtins__.__import__('os').popen('id').read()}}. Это может вести к получению шелла и дальше к полной компроментации системы

Чтобы не допустить такого сценария событий необходимо настроить sandbox окружение, санитизировать все данные или же использовать библиотеки, в шаблонах которых невозможно исполнение кода. 
Если же использовать другие библиотеки невозможно, то остановится на первых двух способах. Sandbox окружение способно обезопасить не только от этой уязвимости, но и от других, а правильная санитизация не допустит выполнение вредоносного кода. Очищать данные можно с помощью регулярок или списка разрешенных выражений

Пример 2.

```python
from flask import Flask, request
import subprocess

app = Flask(name)

@app.route("/dns")
def dns_lookup():
    hostname = request.values.get('hostname')
    cmd = 'nslookup ' + hostname
    output = subprocess.check_output(cmd, shell=True, text=True) // CWE-20
return output
if name == "main":
    app.run(debug=True)
```

В данном коде у нас есть прямая возможность получить шелл, просто в качестве hostname использовав реверс шелл скрипт и скомпрометировать всю систему

Использовать shell команды в коде, тем более с инпутами пользователей плохая идея, поэтому лучше пользоваться библиотеками (в данном случае подойдет PyNslookup). Альтернативным вариантом может быть настройка санитизации, но первый вариант однозначно лучше

## 3. Моделирование угроз

1. Я вижу следующие потенциальные проблемы:
- Пользователь проходит авторизацию, а значит, что данные можно сбрутить, если система не настроена правильно. 
  Мы зашли в личный кабинет и можем редактировать конфигурацию файлов, потенциально присутствует IDOR. 
  Присутствует возможность загружать файлы, возможно неправильно настроена обработка этих файлов и можно подсунуть вместо картинок файлы с вредоносным кодом.
  В БД у нас хранится конфигурация отправки уведомлений, если в уведомлениях присутствует сообщение, написанное пользователем, то потенциально можно сделать SQL иньекцию

2. Получение злоумышленником персональных данных, исполнение кода на сервере с последующей компроментацией последнего, удаление данных злоумышленником из БД
3. Правильная санитизация, работа с куками, и обработка файлов
4. Как реализована авторизация, работа с токенами?
   Как обрабатывается вся поступающая от пользователя информация
   Какой стэк
  










