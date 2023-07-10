# Документация по работе с API Tegro.money при помощи языка Python

# Getting Started
Для начала работы с REST API сервиса Tegro.money первостепенно потребуется сгенерировать ключ для доступа по ссылке https://tegro.money/my/shop-settings/. 

Все данные запроса к сервису передаются посредством протокола HTTP при помощи метода POST на следующий адрес https://tegro.money/api/<api_method>. Тело POST-запроса должно быть сериализовано в JSON-объект. Каждый запрос должен содержать подпись, так же тело POST-запроса должно быть подписано в таком виде, в котором оно отправляется на сервер Банка (в сериализованном виде).

> Каждый запрос должен содержать в себе поле **nonce**, со значением, отличным от предыдущего. Как пример - можно использовать текущее время и дату.

Используйте дял подписи ваш секретный ключ, сформированный алгоритмом SHA-256

# Пример формирования запроса
```
import requests
import json
import time
import hashlib
import hmac

api_key = 'EEFA1913EA9D9351469B1E5D852A'

data = {
    'shop_id': '1913EA9D9351469B1E5D852A',
    'nonce': int(time.time()),
}

body = json.dumps(data)
sign = hmac.new(api_key.encode('utf-8'), body.encode('utf-8'), hashlib.sha256).hexdigest()

headers = {
    'Authorization': 'Bearer ' + sign,
    'Content-Type': 'application/json'
}

url = 'https://tegro.money/api/orders/'

response = requests.post(url, data=body, headers=headers)

print(response.text)
```

# Создание платежа 
С Tegro.money и продавец, и покупатель получают «электронного кассира», который значительно упрощает проведение операций и ускоряет платежи.

Для создания платежа нужно передать необходимые параметры на специальный url: https://tegro.money/pay/?params

## Обязательные параметры
| Ключ          | Описание               |
| ------------- | ---------------------- |
| shop_id       | Публичный ключ проекта |
| amount        | Сумма платежа          |
| order_id      | Идентификатор заказа (номер платежа или email клиента)|
| currency      | Валюта платежа (RUB, USD, EUR) |
| sign          | Подпись запроса |
## Дополнительные параметры
| Ключ          | Описание               |
| ------------- | ---------------------- |
| lang          | Язык интерфейса (ru, en) |
| test          | Если указан со значением "1" - оплата пройдет в тестовом режиме |          |
| payment_system      | ID платежной системы|
| success_url      | Урл успеха |
| fail_url         | Урл ошибки |
| notify_url         | Урл уведомлений |

Для формирования подписи необходимо отсортировать по ключу все обязательные параметры, объединить пары ключ/значение символом & и добавить в конец Ваш секретный ключ. Затем захешировать получившуюся строку MD5, например
```
import hashlib

secret = 'GB%^&*YJni677'
data = {
    'shop_id': 'D0F98E7D7742609DC508D86BB7500914',
    'amount': 100,
    'currency': 'RUB',
    'order_id': '123',
    'test': 1,
}
sorted_data = dict(sorted(data.items()))
query_string = '&'.join([f"{key}={value}" for key, value in sorted_data.items()])
str_to_sign = f"{query_string}{secret}"
sign = hashlib.md5(str_to_sign.encode()).hexdigest()
```
Внимание! Если в форму оплаты был передан флаг тестовой оплаты test=1, этот параметр так же участвует в формировании подписи:
```
import hashlib
import urllib.parse

secret = 'GB%^&*YJni677'
data = {
    'shop_id': 'D0F98E7D7742609DC508D86BB7500914',
    'amount': 100,
    'currency': 'RUB',
    'order_id': '123',
    'test': 1,
}
sorted_data = dict(sorted(data.items()))
query_string = urllib.parse.urlencode(sorted_data)
str_to_sign = f"{query_string}{secret}"
sign = hashlib.md5(str_to_sign.encode()).hexdigest()
```
Возможен переход сразу в платежную систему, если Вы готовы передать все данные для оплаты во входящем запросе. Для этого отправить данные нужно методом POST на урл https://tegro.money/pay/form/ обязательно указать параметр payment_system и передать все обязательные поля для этого способа оплаты. В большинстве случаев это email, для дополнительной информации обратитесь в службу поддержки.

Пример:
```
<form action="https://tegro.money/pay/form/" method="post">
    <input type="hidden" name="shop_id" value="D0F98E7D7742609DC508D86BB7500914">
    <input type="hidden" name="amount" value="100">
    <input type="hidden" name="order_id" value="123">
    <input type="hidden" name="lang" value="ru">
    <input type="hidden" name="currency" value="RUB">
    <input type="hidden" name="payment_system" value="11">
    <input type="hidden" name="fields[email]" value="user@site.ru">
    <input type="hidden" name="sign" value="e51845e62b106d245cc96c431d8aae42">
    <input type="submit" value="Оплатить">
</form>
```
# Создание простой формы в личном кабинете
Перейдите на страницу настроек магазина https://tegro.money/my/shop-settings/ и щелкните по вкладке "Форма оплаты"
В таблице заполните необходимые поля и нажмите кнопку "получить форму", чтобы получить HTML код для вставки формы на свою страницу

# Передача информации о заказе
В некоторых случаях на форму оплаты необходимо отправить данные о составе заказа, как то - название товара/услуги, количество и стоимость. Для этого необходимо включить в запрос параметр receipt, например:
```
<form action="https://tegro.money/pay/form/" method="post">
<input type="hidden" name="shop_id" value="D0F98E7D7742609DC508D86BB7500914">


<!-- Товар 1 -->
<input type="hidden" name="receipt[items][0][name]" value="Пример услуги">
<input type="hidden" name="receipt[items][0][count]" value="1">
<input type="hidden" name="receipt[items][0][price]" value="20">

<!-- Товар 2 -->
<input type="hidden" name="receipt[items][1][name]" value="Пример услуги 2">
<input type="hidden" name="receipt[items][1][count]" value="2">
<input type="hidden" name="receipt[items][1][price]" value="40">


<!-- общая сумма оплаты должна равняться сумме всех товаров! -->
<input type="hidden" name="amount" value="100">

<input type="hidden" name="order_id" value="123">
<input type="hidden" name="lang" value="ru">
<input type="hidden" name="currency" value="RUB">
<input type="hidden" name="payment_system" value="11">
<input type="hidden" name="fields[email]" value="user@site.ru">
<input type="hidden" name="sign" value="e51845e62b106d245cc96c431d8aae42">
<input type="submit" value="Оплатить">
</form>
```
Внимание! Данную форму необходимо отправлять на наш платежный урл методом POST

# Уведомление об оплате
После оплаты, на указанный в настройках Вашего магазина URL уведомлений будет отправлен запрос, содержащий информацию об оплате
| Ключ          | Описание               |
| ------------- | ---------------------- |
| shop_id         | Публичный ключ проекта |
| amount        | Сумма платежа |
| order_id     | Идентификатор заказа|
| payment_system      | Платежная система |
| currency        |Валюта платежа (RUB, USD, EUR) |
| test         | Если был задан при оплате |
| sign        |Подпись запроса |
Подпись формируется так же как и при создании формы, но в формировании хеш участвуют все поля, например:
```
import hashlib
import urllib.parse

secret = 'GB%^&*YJni677'
data = request.POST.copy() 
data.pop('sign', None)
sorted_data = dict(sorted(data.items()))
query_string = urllib.parse.urlencode(sorted_data)
str_to_sign = f"{query_string}{secret}"
sign = hashlib.md5(str_to_sign.encode()).hexdigest()
```



# API
## Создание заказа
> POST https://tegro.money/api/createOrder/

Вы можете использовать этот метод для получения прямой ссылки на оплату заказа

### Формат запроса

#### Header
||||
| ------------- | ---------------------- | ------------- |
| Authorization | string |Подпись запроса | 

#### Body
||||
| ------------- | ---------------------- | ------------- |
| shop_id        | string  | Shop ID |
| nonce          | integer | Уникальный номер запроса |
| currency       | string  | Валюта оплаты, RUB/USD/EUR etc |
| amount         | number  | Сумма оплаты |
| order_id       | string  | Номер заказа в Вашем магазине |
| payment_system | integer | ID платежной системы |
| fields         | array   | Данные о покупателе |
| receipt        | array   | Данные о корзине |

### Формат ответа
```
{
  "type": "success",
  "desc": "",
  "data": {
    "id": 755555,
    "url": "https://tegro.money/pay/complete/755555/7f259f856e7682a6e98179036a623696/"
  }
}
```
### Пример запроса 
```
{
  "shop_id": "1913EA935149B1E5D852A",
  "nonce": 1613435880,
  "currency": "RUB",
  "amount": 1200,
  "order_id": "test order",
  "payment_system": 5,
  "fields": {
    "email": "user@email.ru",
    "phone": "79111231212"
  },
  "receipt": {
    "items": [
      {
        "name": "test item 1",
        "count": 1,
        "price": 600
      },
      {
        "name": "test item 2",
        "count": 1,
        "price": 600
      }
    ]
  }
}
```
## Список магазинов
> POST https://tegro.money/api/shops/
Получение списка ваших магазинов
### Формат запроса 

#### Header
||||
| ------------- | ---------------------- | ------------- |
| Authorization | string |Подпись запроса |

#### Body
||||
| ------------- | ---------------------- | ------------- |
| shop_id | string  | Shop ID |
| nonce   | integer | Уникальный номер запроса |

### Формат ответа
```
{
  "type": "success",
  "desc": "",
  "data": {
    "user_id": 1,
    "shops": [
      {
        "id": 1,
        "date_added": "2020-11-03 18:04:07",
        "name": "DEMO1",
        "url": "https://demo1",
        "status": 1,
        "public_key": "D0F98E7DD86BB7500914",
        "desc": "DEMO1 SHOP"
      },
      {
        "id": 2,
        "date_added": "2020-11-03 22:38:58",
        "name": "DEMO2",
        "url": "https://demo2",
        "status": 0,
        "public_key": "1913EA935149B1E5D852A",
        "desc": "DEMO2 SHOP"
      }
    ]
  }
}
```
### Пример запроса 
```
{
  "shop_id": "1913EA935149B1E5D852A",
  "nonce": 1613435880
}
```

## Баланс
> POST https://tegro.money/api/balance/
Получение баланса всех кошельков

### Формат запроса 

#### Header
||||
| ------------- | ---------------------- | ------------- |
| Authorization | string |Подпись запроса |

#### Body
||||
| ------------- | ---------------------- | ------------- |
| shop_id | string  | Shop ID |
| nonce   | integer | Уникальный номер запроса | 

### Формат ответа 
```
{
  "type": "success",
  "desc": "",
  "data": {
    "user_id": 1,
    "balance": {
      "RUB": "1396.68",
      "USD": "0.00",
      "EUR": "1.23",
      "UAH": "0.00"
    }
  }
}
```
### Пример запроса 
```
{
  "shop_id": "1913EA935149B1E5D852A",
  "nonce": 1613435880,
  "currency": "RUB",
  "amount": 1200,
  "order_id": "test order",
  "payment_system": 5,
  "fields": {
    "email": "user@email.ru",
    "phone": "79111231212"
  },
  "receipt": {
    "items": [
      {
        "name": "test item 1",
        "count": 1,
        "price": 600
      },
      {
        "name": "test item 2",
        "count": 1,
        "price": 600
      }
    ]
  }
}
```

## Пример запроса 
```
{
  "shop_id": "1913EA935149B1E5D852A",
  "nonce": 1613435880
}
```

## Проверка заказа
> POST https://tegro.money/api/order/
Получение информации о заказе

### Формат запроса 

#### Header
||||
| ------------- | ---------------------- | ------------- |
| Authorization | string |Подпись запроса|

#### Body
||||
| ------------- | ---------------------- | ------------- |
| shop_id   | string  | Shop ID |
| nonce     | integer | Уникальный номер запроса | 
| order_id  | integer | Номер платежа tegro.money|
| payment_id| string  | или номер платежа магазина|

### Формат ответа 
```
{
  "type": "success",
  "desc": "",
  "data": {
    "id": 1232,
    "date_created": "2020-11-14 23:32:37",
    "date_payed": "2020-11-14 23:33:39",
    "status": 1,
    "payment_system_id": 10,
    "currency_id": 1,
    "amount": "64.18000000",
    "fee": "4.00000000",
    "email": "user@site.ru",
    "test_order": 0,
    "payment_id": "Order #17854"
  }
}
```
### Пример запроса 
```
{
  "shop_id": "1913EA935149B1E5D852A",
  "nonce": 1613435880,
  "payment_id": "test order"
}
```

## Проверка заказа
> POST https://tegro.money/api/orders/
Получение информации о заказах

### Формат запроса 

#### Header
||||
| ------------- | ---------------------- | ------------- |
| Authorization | string |Подпись запроса|

#### Body
||||
| ------------- | ---------------------- | ------------- |
| shop_id   | string  | Shop ID|
| nonce     | integer | Уникальный номер запроса| 
| page 	    | integer | Страница|
### Формат ответа 
```
{
  "type": "success",
  "desc": "",
  "data": [
    {
      "id": 123,
      "date_created": "2020-11-14 23:32:37",
      "date_payed": "2020-11-14 23:33:39",
      "status": 1,
      "payment_system_id": 10,
      "currency_id": 1,
      "amount": "64.18000000",
      "fee": "4.00000000",
      "email": "user@somesite",
      "test_order": 0,
      "payment_id": "Order #4175"
    },
    {
      "id": 124,
      "date_created": "2020-11-14 23:30:05",
      "date_payed": null,
      "status": 0,
      "payment_system_id": 10,
      "currency_id": 1,
      "amount": "64.18000000",
      "fee": "4.00000000",
      "email": "user2@somesite",
      "test_order": 0,
      "payment_id": "Order #4174"
    }
  ]
}
```
### Пример запроса 
```
{
  "shop_id": "1913EA935149B1E5D852A",
  "nonce": 1613435880,
  "page": 1
}
```


## Создание выплаты
> POST https://tegro.money/api/createWithdrawal/
Создание выплаты 

### Формат запроса 

#### Header
||||
| ------------- | ---------------------- | ------------- |
| Authorization | string |Подпись запроса|

#### Body
||||
| ------------- | ---------------------- | ------------- |
| shop_id        | string  | Shop ID |
| nonce          | integer | Уникальный номер запроса| 
| currency 	     | string  | Валюта RUB / USD / EUR etc|
| account 	     | string  | Номер счета для выплаты|
| amount 	     | number  | Сумма|
| payment_id     | string  | Идентификатор вывода| 
| payment_system | integer | Платежная система| 

## Проверка выплаты 
> POST https://tegro.money/api/withdrawal/
Проверка выплаты

### Формат запроса 

#### Header
||||
| ------------- | ---------------------- | ------------- |
| Authorization | string |Подпись запроса

#### Body
||||
| ------------- | ---------------------- | ------------- |
| shop_id        | string  | Shop ID |
| nonce          | integer | Уникальный номер запроса| 
| order_id       | integer | Номер платежа tegro.money|
| payment_id 	 |string   | или номер платежа магазина|
