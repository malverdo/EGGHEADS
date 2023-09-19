# EGGHEADS 14.05.2023
---
# Тестовое задание
---
# Описание(Правила выполнения)
На каждое задание необходим подробный ответ, ответы вида «Да», «Нет» не принимаются.
Данные варианты кода выдуманы только для тестового задания, и ни к какому из проектов (по крайней мере наших) отношения не имеют.

# 1.  Какие проблемы могут возникнуть при использовании данного кода?

```php
$mysqli = new mysqli("localhost", "my_user", "my_password", "world");
$id = $_GET['id'];
$res = $mysqli->query('SELECT * FROM users WHERE u_id='. $id);  
$user = $res->fetch_assoc();
```

### Проблемы

1. Подключение hardcore
	- Придётся искать по коду подключение
	- Параметры не зашифрованы 
		- хороший способ хранить параметры в секрете [symfony Sensitive Information Secret](https://symfony.com/doc/current/configuration/secrets.html)
	- Трудно изменить СУБД  
	- Нужно понимать какие настройки в этом подключение 
		- типы данных возвращаться в строках
		- Какой набор символов ('utf8')
		- какие уведомления об ошибках
```php
$mysqli = new mysqli("localhost", "my_user", "my_password", "world");
``` 

2.  Создание переменной 
	- Если нет 'id' то возможна ошибка в зависимости от настроек
	- Трудно уследить за глобальной переменной 
```
$id = $_GET['id'];
```

3. Запрос
	- Запрос не подготовленный, значит может повредить его , нужно через знак "?"
	- запрос query непонятно, будут данные с разными типами или нет 
	- Использование '\*' не оптимальная практика, действительно ли нужны все поля 
	- в query не надо использовать заполнители 
```php
$res = $mysqli->query('SELECT * FROM users WHERE u_id='. $id);  
```

4. Ассоциативный массив
	- Из-за выборки всех полей выше, мы не знаем точно, какие ключи будут в массиве
```php
$user = $res->fetch_assoc();
```


# 2. Сделайте рефакторинг
```php
$questionsQ = $mysqli->query('SELECT * FROM questions WHERE catalog_id='. $catId);
$result = array();
while ($question = $questionsQ->fetch_assoc()) {
	$userQ = $mysqli->query('SELECT name, gender FROM users WHERE id='. (int) $question[‘user_id’]);
	$user = $userQ->fetch_assoc();
	$result[] = array('question'=>$question, 'user'=>$user);
	$userQ->free();
}

$questionsQ->free();
```

### **Рефакторинг** 
```php
$result = [];
$mysqli->prepare('SELECT questions.name as questions_name, users.name as user_name, users.gender as user_gender FROM questions
				   INNER JOIN users ON users.id = questions.user_id 
				   WHERE questions.catalog_id=?');

$mysqli->bind_param("i", $catId);
$mysqli->execute();

$questionsUsers = $mysqli->fetchAll()
foreach($questionsUsers as $questionUser) {
	$result[] = [
		'question' => [
			$questionUser['questions_name']
		],
		'user' => [
			$questionUser['user_name'],
			$questionUser['user_gender']
		],
	];
}

```

# 3. Напиши SQL-запрос

Имеем следующие таблицы:
1. users — контрагенты
    1. id
    2. name
    3. phone
    4. email
    5. created — дата создания записи
2. orders — заказы
    1. id
    2. subtotal — сумма всех товарных позиций
    3. created — дата и время поступления заказа (Y-m-d H:i:s)
    4. city_id — город доставки
    5. user_id 

Необходимо выбрать одним запросом список контрагентов в следующем формате (следует учесть, что будет включена опция only_full_group_by в MySql):
1. Имя контрагента
2. Его телефон
3. Сумма всех его заказов
4. Его средний чек
5. Дата последнего заказа

## Решение 
```sql
select u.name, u.phone, SUM(orders.subtotal), AVG(orders.subtotal), MAX(orders.created) FROM orders  
	JOIN users u on u.id = orders.user_id  
	group by u.name, u.phone
```


# 4. Напиши SQL-запросы

Имеем следующую таблицу со списком сотрудников

![[Снимок экрана от 2023-07-19 11-43-47.png]]
  
### 1. Написать запрос для вывода самой большой зарплаты в каждом департаменте
```sql
select "departamentId", max(salary) OVER ( partition by "departamentId") from employee
```

### 2. Написать запрос для вывода списка сотрудников из 3-го департамента у которых ЗП больше 90000
```sql
select * from employee WHERE "departamentId" = 3 AND salary > 90000;
```

### 3. Написать запрос по созданию индексов для ускорения
- чтобы лучше сделать индекс на ускорение надо смотреть на планировщик и его работу через EXPLANE. 
- Чтобы составить индексы нужны ответы на такие вопросы как
	- Кардинальность каждого  поля и его тип данных
	- Самые частые запросы 
	- Самые частые выборки 
	- Настройки random_page_cost
	- Забираются ли больше 30% данных из таблицы
	- Порядок полей в запросах 
	- Какой порядок записей в поле 
```sql
CREATE INDEX idx_employee_departamentId_salary ON employee(salary, departamentId) using Btree;
```


# **5. Сделайте рефакторинг кода для работы с API чужого сервиса**  
```js
function printOrderTotal(responseString) {
	var responseJSON = JSON.parse(responseString);
	
	responseJSON.forEach(function(item, index){
		if (item.price = undefined) {
			item.price = 0;
		}
		orderSubtotal += item.price;
	});
	console.log( 'Стоимость заказа: ' + total > 0? 'Бесплатно': total + ' руб.');
}
```

### **Рефакторинг** 
```php
function printOrderTotal($responseString) {
	$responseJSON = json_decode($responseString, true);
	$totalPrice = array_sum(array_column($responseJSON, 'price'));
	$costDecision = $totalPrice ? sprintf('%f руб.', $totalPrice) : 'Бесплатно';
    echo sprintf('Стоимость заказа: %s', $costDecision)
}
```
