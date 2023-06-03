# Модификация рантайма kphp

## Цель
Я расскажу про добавление новых функций в runtime KPHP. Точнее про тернистую дорогу на пути.

## План
1. Подготовка
2. runtime
    - добавление функций
    - типы
    - флаги
    - подключение библиотеки
    - изменение подключаемых библиотек
3. Тесты
    - cpp тесты
    - php тесты
4. pull_request

## Подготовка

[Устанавливаем kphp из репозитория](https://vkcom.github.io/kphp/kphp-internals/developing-and-extending-kphp/compiling-kphp-from-sources.html)

## runtime
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

Отлично, мы поняли, что нам нужно лишь использовать эту библиотеку, которая возьмет всю конвертацию на себя, а мы просто допишем интерфейс. Скачиваем эту библиотеку, устанавливаем, изучаем ее и пишем реализацию в любом сишном файлике (не внутри kphp, сейчас нам надо, чтобы просто работало). Для проверки кодировки нужна функция конвертации `mb_convert_encoding`, которую мы тоже реализуем:
```c
#include <libmbfl/mbfl/mbfilter.h>

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

#### Добавить php-интерфейс (txt)
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

#### Добавить код с интерфейсом (h)
Все модули рантайма находятся в папке `runtime`. Наши функции являются частью расширения `mbstring` для php. Оказывается уже есть файлик `mbstring.h`. Давайте откроем и посмотрим:
![Снимок экрана 2023-06-02 в 23 27 45](https://github.com/andreylzmw/kphp-runtime-docs/assets/110744283/05f5d197-ed8b-46ab-a3cf-064152d5c3b9)
Оказывается mb_check_encoding уже реализована, странно.

Оказывается mb_check_encoding уже реализована, странно. Посмотрим cpp файл:
![Снимок экрана 2023-06-02 в 22 25 11](https://github.com/andreylzmw/kphp-runtime-docs/assets/110744283/223d3e0a-dab2-4794-b2f1-b234bbdd2f75)

Хм, функция уже реализована, но все работает только для двух кодировок (UTF-8 и Windows-1251). Ничего страшного, теперь мы покажем им все кодировки! (о последсвиях расскажу в конце). Dидим, что входными параметрами являются переменные типа `string`. Ловушка! Это не string из I/O! Это string из kphp! Где про нее почитать? Все типы включаются из `runtime/kphp_core.h`. Смотрим внутри: ![Снимок экрана 2023-06-02 в 22 53 46](https://github.com/andreylzmw/kphp-runtime-docs/assets/110744283/ef3c256a-8b39-4dd6-99ef-4e0fc279dbdb). Поменяем интерфейс `mb_check_encoding`:
```cpp
bool f$mb_check_encoding(const string &value, const string &encoding);
```

аналогично для `mb_convert_encoding`:
```cpp
string f$mb_convert_encoding(const string &str, const string &to_encoding, const string &from_encoding);
```
Отлично, идем дальше.

#### Добавить код с реализацией (cpp)

В предыдущем шаге мы нашли все типы, так что можно посмотреть в `string.inl` и узнать, как доставть из нее `const char *`, который нам уже знаком или как создать string из `const char *`. Теперь перепишем функцию `mb_check_encoding`:
```сpp
bool f$mb_check_encoding(const string &value, const string &encoding) {
	const char *c_encoding = encoding.c_str();
	const char *c_value = value.c_str();
	// ...
}
```
Теперь осталось сделать аналогичное для функции `mb_convert_encoding`:
```сpp
string f$mb_convert_encoding(const string &str, const string &to_encoding, const string &from_encoding) {
	const char *c_string = s.c_str();
	const char *c_to_encoding = to_encoding.c_str();
	const char *c_from_encoding = from_encoding.c_str();
	// ...
	return string((const char*)ret->val, ret->len);
}
```

#### Добавить файлы в сборку (cmake)
Идем в `runtime/runtime.cmake`. Все cpp должны оказаться в `KPHP_RUNTIME_SOURCES`. Но для удобства можно группировать их. В нашем случае `mbstring.cpp` уже включен в сборку. Но, можно сделать по другому:
```cmake
prepend(KPHP_RUNTIME_MBSTRING_SOURCES
        mbstring.cpp)
```
тогда можно в `KPHP_RUNTIME_SOURCES` включать не отдельный файл `mbstring.cpp`, а `KPHP_RUNTIME_MBSTRING_SOURCES`:
```cmake
prepend(KPHP_RUNTIME_SOURCES ${BASE_DIR}/runtime/
        ${KPHP_RUNTIME_MBSTRING_SOURCES}
	...
```

### Подключение библиотеки
Чтобы kphp мог линковать нужную нам библиотеку `libmbfl` нужно добавить ее (без `lib` в начале) в `external_static_libs` в `compiler/compiler-settings.cpp`:
```cpp
std::vector<vk::string_view> external_static_libs{..., "mbfl"};
```

Фух, теперь точно все. Билдим!
```
mkdir build && cd build && cmake .. -DDOWNNLOAD_MISSING_LIBRARIES=On && make -j$(nproc)
```

Создаем test.php:
```php
echo mb_check_encoding("Hello World", "UTF-8");
```

Запускаем!
```
./objs/bin/kphp2cpp -M test.php && ./kphp_out/cli
```
Ура! Мы успешно добавили новые функции в рантайм kphp!

### !!! Важно !!!
Не все так просто. Мы очень быстро их добавили (лишь бы работало). Теперь когда мы убедились, что все работает нужно переделать некоторые моменты.

1. И было бы хорошим тоном разграничивать логичное поведение функции и ее поведение в php. Например в php `mb_convert_encoding` если при конвертации есть символы, которых нет в выходной кодировке, то их байты заменяются на 0x63 ('?' в ASCII). Это поведение уникально, поэтому лучше ее реализовать в `f$` функции, а всю логику вынести в обычную функцию.

2. Нужны другие типы - `mb_check_enсoding` в качестве параметра данных может принимать массив строк или `null`, нужно это придусмотреть. Причем поведение функции должно быть идентично php (дажа если поведение в php нелогично). Аналогично для `mb_convert_encoding`, которая может и принимать и возвращать строку или массив строк.

3. `mbstring` это расширение kphp, которое в `php-src`, собирается и подключается только если указан определенный флаг при компиляции. Нужно сделать и это.

4. Обработка разных версий php.

### Хороший тон
1. Используйте `const &` для входных параметров (если они не меняются)
2. Разграничевание логики и поведения как в php (используйте `static`):
```сpp
static bool check_enсoding(const char *value, const char *encoding) {
	// ...
}

bool f$mb_check_encoding(const string &value, const string &encoding) {
	const char *c_encoding = encoding.c_str();
	const char *c_value = value.c_str();
	bool res = check_enсoding(c_value, c_encoding);
	// process differences in php
	return res
}
```
3. Используйте `noexcept` если функция не вызывает исключений:
```сpp
static bool check_enсoding(const char *value, const char *encoding) {
	// ...
}

bool f$mb_check_encoding(const string &value, const string &encoding) noexcept {
	const char *c_encoding = encoding.c_str();
	const char *c_value = value.c_str();
	bool res = check_enсoding(c_value, c_encoding);
	// process differences in php
	return res
}
```

### Типы
`mb_check_encoding` в качестве переменной данных может принимать `string|array|null` (судя по php-интерфейсу). В C++ мы не можем записывать типы так, поэтому заменяем тип на `mixed` (смешанный). Про `mixed` можно почитать аналагично типу `string`. Параметр кодировки `mb_check_encoding` (судя по php-интерфейсу) имеет тип `?string`, что нужно заменить на `Optional<string>`. Про `Optional` можно почитать аналагично типу `string`. Главная информация в том, что `mixed` мы можем проверять на любой тип через `.is_<тип>` и попытаться привести к любому типу через `.to_<тип>`. Из `Optional` можно достать значение через `.val()`.
```сpp
static bool check_enсoding(const char *value, const char *encoding) {
	// ...
}

bool f$mb_check_encoding(const mixed &value, const Optional<string> &encoding) noexcept {
	if (encoding.is_null() || value.is_null()) return 1;
	const char *c_encoding = encoding.val().c_str();
	if (value.is_string()) { // ... }
	if (value.is_array()) { // ... }
	return 1;
}
```

### Расширение
`mbstring` это расширение php, его нужно включить через флаг при сборке с `cmake`. Пусть флагом будет - `MBFL` Чтобы это сделать нужно всего 3 шага:
1. `cmake/external-libraries.cmake` - добавить флаг в cmake и пробросить его в компилятор:
```cmake
option(DOWNLOAD_MISSING_LIBRARIES "download and build missing libraries if needed" OFF)
option(MBFL "build mbstring" OFF)
# ...
```
3. `runtime/runtime.cmake` - включить файлы в сборку по флагу
```cmake
# ...
if (MBFL)
	prepend(KPHP_RUNTIME_MBSTRING_SOURCES
		mbstring.cpp)
endif()
# ...
prepend(KPHP_RUNTIME_SOURCES ${BASE_DIR}/runtime/
        ${KPHP_RUNTIME_MBSTRING_SOURCES}
# ...
```

В моем случае некоторый функции из `mbstring.cpp` использовались в самом kphp (не в рантайме). Это значит, что мне нужно всегда собирать `mbstring.cpp`, но не собирать большинство функции по флагу. Чтобы это сделать нужно 3 шага:
1. `cmake/external-libraries.cmake` - добавить флаг в cmake и пробросить его в компилятор:
```cmake
option(DOWNLOAD_MISSING_LIBRARIES "download and build missing libraries if needed" OFF)
option(MBFL "build mbstring" OFF)
# ...
```
2. `compiler/compiler-settings.cpp` - пробросить флаг дальше из компилятора в рантайм:
```cpp
void CompilerSettings::init() {
	// ...
	std::stringstream ss;
	#ifdef MBFL
	ss << " -DMBFL ";
	#endif
	// ...
```
3. `runtime/mbstring.*` - через `#ifdef` отслеживать флаг из cmake:
```cpp
#ifdef MBFL
bool f$mb_check_encoding(const mixed &value, const Optional<string> &encoding) noexcept;
#endif
```

### Обработка разных версий php
В разработке...

### Изменение подключаемых библиотек
Все работает. Библиотека линкуется. Но было бы хорошо, если не нужно было бы ее устанавливать самому, чтобы kphp по флагу `MBFL` скачивал библиотеку, собирал и линковал. Тогда не придется писать новые доки для установки библиотеки + можно заморозить версию библиотеки на форке. Разберем [мой форк](https://github.com/andreylzmw/libmbfl/commit/ed66e4e03afb7cc85385974c00542505be4556e8) `libmbfl`. Мне не повезло с библиотекой, она с 1992 года не обновляется, нет документации, нет статей с ее использование и билдится она по-своему. Чтобы kphph сам скачивал, билдил и линковал библиотеку нужно выполнить шагов:
1. 




# Шпаргалка

