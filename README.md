# Модификация рантайма kphp

## Цель
Я расскажу про добавление новых функций в runtime KPHP. Точнее про тернистую дорогу на пути.

## План
1. Подготовка
2. runtime
    - добавление функций
    - подключение библиотек
    - флаги
    - изменение подключаемых библиотек
3. Тесты
    - cpp тесты
    - php тесты
4. pull_request

## 1. Подготовка

[Устанавливаем kphp из репозитория](https://vkcom.github.io/kphp/kphp-internals/developing-and-extending-kphp/compiling-kphp-from-sources.html)

## 2. runtime
### Добавление функций
В качестве примера возьмем ситуацию, когда нам нужно реализовать функцию `mb_check_encoding` из php. Первым делом идем в доки:
![Снимок экрана 2023-06-02 в 01 39 38](https://github.com/andreylzmw/kphp-runtime-docs/assets/110744283/0177f3fd-c393-4fe3-85c6-45acbf46663b)

Узнаем, что функция проверяет кодировку строки или массива строк. Массив строк обрабатывается рекурсивно, так что сфокусируемся на функции, работающей для строки. Теперь идем в [код php](https://github.com/php/php-src) смотреть как работает функция в php:
![Снимок экрана 2023-06-02 в 01 51 49](https://github.com/andreylzmw/kphp-runtime-docs/assets/110744283/c2f006ce-b6a3-4616-85ce-883f03a20706)

Идем смотреть в сишный файл, чтобы протрейсить реализацию:
![Снимок экрана 2023-06-02 в 01 55 59](https://github.com/andreylzmw/kphp-runtime-docs/assets/110744283/3f18acc6-98ab-4f60-ab55-b49c80349ce1)

Видим, что конвертация идет через входной параметр типа `mbfl_encoding`. Ищем:
![Снимок экрана 2023-06-02 в 02 12 42](https://github.com/andreylzmw/kphp-runtime-docs/assets/110744283/36469744-244b-4cd0-b46f-a013f84cc861)

Сразу видим, что эта структура - часть какой-то библиотеки `libmbfl`, которая видимо и занимается конвертацией. Ищем ее в интернете и находим репозиторий:
![Снимок экрана 2023-06-02 в 02 10 43](https://github.com/andreylzmw/kphp-runtime-docs/assets/110744283/4ffc702e-eea2-4609-b1d0-6ed55b50fed6)

Отлично, мы поняли, что нам нужно лишь использовать эту библиотеку, которая возьмет всю конвертацию на себя, а мы просто допишем интерфейс. Скачиваем эту библиотеку, изучаем ее и пишем реализацию в любом сишном файлике (не внутри kphp, сейчас нам надо, чтобы просто работало). Для проверки кодировки нужна функция конвертации `mb_convert_encoding`, которую мы тоже реализуем:
```c
#include "libmbfl/mbfl/mbfilter.h"

mbfl_string *mb_convert_encoding(const char *str, const char *to, const char *from) {

	int len = strlen(str);
	enum mbfl_no_encoding from_encoding, to_encoding;
	mbfl_buffer_converter *convd = NULL;
	mbfl_string _string, result, *ret;

	/* from internal to mbfl */
	from_encoding = mbfl_name2no_encoding(from);
	to_encoding = mbfl_name2no_encoding(to);

	/* init buffer mbfl strings */
	mbfl_string_init(&_string);
	mbfl_string_init(&result);
	_string.no_encoding = from_encoding;
	_string.len = len;
	_string.val = (unsigned char*)str;

	/* converting */
	convd = mbfl_buffer_converter_new(from_encoding, to_encoding, 0);
	ret = mbfl_buffer_converter_feed_result(convd, &_string, &result);
	mbfl_buffer_converter_delete(convd);

	/* fix converting with multibyte encodings */
	if (len % 2 != 0 && ret->len % 2 == 0 && len < ret->len) {
		ret->len++;
		ret->val[ret->len-1] = 63;
	}
	
	return ret;
}

bool mb_check_encoding(const char *value, const char *encoding) {

	/* init buffer mbfl strins */
	mbfl_string _string;
	mbfl_string_init(&_string);
	_string.val = (unsigned char*)value;
	_string.len = strlen((char*)value);

	/* from internal to mbfl */
	const mbfl_encoding *enc = mbfl_name2encoding(encoding);

	/* get all supported encodings */
	const mbfl_encoding **encs = mbfl_get_supported_encodings();
	int len = sizeof(**encs);

	/* identify encoding of input string */
	/* Warning! String can be represented in different encodings, so check needed */
	const mbfl_encoding *i_enc = mbfl_identify_encoding2(&_string, encs, len, 1);

	/* perform convering */
	const char *i_enc_str = (const char*)mb_convert_encoding(value, i_enc->name, enc->name)->val;
	const char *enc_str = (const char*)mb_convert_encoding(i_enc_str, enc->name, i_enc->name)->val;

	/* check equality */
	/* Warning! strcmp not working, because of different encodings */
	bool res = true;
	for (int i = 0; i < strlen(enc_str); i++)
		if (enc_str[i] != value[i]) {
			res = false;
			break;
		}

	free((void*)i_enc_str);
	free((void*)enc_str);
	return res;
}
```
Функции работают, но только для сишных строк, теперь нам нужно перенести эти локально работающие функции в runtime kphp. Для этого есть 4 шага:
1. Добавить php-интерфейс (txt)
2. Добавить код с интерфейсом (h)
3. Добавить код с реализацией (cpp)
4. Добавить файлы в сборку (cmake)

#### (2.20/4). Добавить php-интерфейс (txt)
Для того, чтобы правильно перенести php интерфейс, нужно знать про типы в kphp [тут](https://vkcom.github.io/kphp/kphp-language/static-type-system/kphp-type-system.html)
![kphp-types](https://github.com/andreylzmw/kphp-runtime-docs/assets/110744283/34deba9b-61c7-4e20-bbc5-6418c69455fd)

Итак, открываем файл `builtin-functions/_functions.txt`. И в самый конец добавляем переделанный интерфейс из php. Например в php mb_check_encoding имеет следующий интерфейс:
```php
function mb_check_encoding(array|string|null $value = null, ?string $encoding = null): bool
```
в kphp это будет:
```php
function mb_check_encoding(array|string|null $value = null, ?string $encoding = null): bool;
```

Аналогично для mb_convert_encoding в php:
```php
function mb_convert_encoding(array|string $string, string $to_encoding, array|string|null $from_encoding = null): array|string|false
```
в kphp:
```php
function mb_convert_encoding(array|string $string, string $to_encoding, array|string|null $from_encoding = null): array|string|false;
```
Вот и все, теперь нужно добавить код.

#### (2.40/4). Добавить код с интерфейсом (h)
Все модули рантайма находятся в папке `runtime`. Наши функции являются частью расширения `mbstring` для php. Оказывается уже есть файлик `mbstring.cpp`. Давайте откроем и посмотрим:
![Снимок экрана 2023-06-02 в 22 25 11](https://github.com/andreylzmw/kphp-runtime-docs/assets/110744283/223d3e0a-dab2-4794-b2f1-b234bbdd2f75)

Хм, функция уже реализована, но только для двух кодировок (UTF-8 и Windows-1251). Ничего страшного, теперь мы покажем им все кодировки! (о последсвиях расскажу в конце). Кстати почему название функции начинается с `f$`? Так парсер понимает какие функции нужно искать в интерфейсах (см шаг (2.20/4).  Добавить php-интерфейс (txt)). Видим, что входными параметрами являются переменные типа `string`. Ловушка! Это не string из I/O! Это string из kphp!



#### (2.60/4). Добавить код с реализацией (cpp)

#### (2.80/4). Добавить файлы в сборку (cmake)

## 


