URFAClient 1.0.2
==========

Универсальный PHP клиент для биллинговой системы NetUp UTM5 на основе api.xml

## Зависимости
- UTM 5.2.1-008+
- PHP 5.3+
- Filter
- Hash
- OpenSSL
- SimpleXML

## Описание файлов
- URFAClient - главный класс библиотеки
- URFAClient_API - объект предоставляет обращение к функциям из api.xml
- URFAClient_Connection - объект соединения с ядром UTM5
- URFAClient_Packet - объект для подготовки получения/отправки бинарных данных ядру
- URFAClient_Collector - сборщик информации для класса API
- URFAClient_Log - журнал с собранными данными
- admin.crt - сертификат для вызова админских функций
- api.xml - файл с описанием api (UTM-5.3-002-update5)

## Описание конфига
- login (required) - логин админа или абонента
- password (required) - пароль админа или абонента соответственно
- address (required) - адрес ядра UTM5
- admin (default: TRUE) - указываем какой пользователь подключается. Если TRUE предоставляет сертификат admin.crt для соединения.
- port (default: 11758) - порт ядра UTM5
- api (default: 'api.xml') - полный путь до файла api.xml
- log (default: FALSE) - сборщик логов. Если TRUE, перехватывает исключения из URFAClient_API.

## Пример
Рассмотрим пример использования библиотеки на примере функции rpcf_add_user, у нас есть XML описание:
```
<function name="rpcf_add_user" id="0x2005">
  <input>
    <integer name="user_id" default="0"/>
    <string name="login"/>
    <string name="password"/>
    <string name="full_name" default=""/>
    <if variable="user_id" value="0" condition="eq">
      <integer name="unused" default="0"/>
    </if>
    <integer name="is_juridical" default="0"/>
    <string name="jur_address" default=""/>
    <string name="act_address" default=""/>
    <string name="flat_number" default=""/>
    <string name="entrance" default=""/>
    <string name="floor" default=""/>
    <string name="district" default=""/>
    <string name="building" default=""/>
    <string name="passport" default=""/>
    <integer name="house_id" default="0"/>
    <string name="work_tel" default=""/>
    <string name="home_tel" default=""/>
    <string name="mob_tel" default=""/>
    <string name="web_page" default=""/>
    <string name="icq_number" default=""/>
    <string name="tax_number" default=""/>
    <string name="kpp_number" default=""/>
    <string name="email" default=""/>
    <integer name="bank_id" default="0"/>
    <string name="bank_account" default=""/>
    <string name="comments" default=""/>
    <string name="personal_manager" default=""/>
    <integer name="connect_date" default="0"/>
    <integer name="is_send_invoice" default="0"/>
    <integer name="advance_payment" default="0"/>
    <integer name="parameters_count" default="size(parameter_value)"/>
    <for name="i" from="0" count="size(parameter_value)">
      <integer name="parameter_id" array_index="i"/>
      <string name="parameter_value" array_index="i"/>
    </for>
  </input>
  <output>
    <integer name="user_id"/>
    <string name="error_msg"/>
    <if variable="user_id" value="0" condition="eq">
      <error code="10" comment="unable to add or edit user"/>
    </if>
    <if variable="user_id" value="-1" condition="eq">
      <error code="10" comment="unable to add user, probably login exists"/>
    </if>
  </output>
</function>
```
И так, нам нужно описать входные параметры (элемент input) в ассоциативный массив.
Если в элементе присутствует атрибут _default_, параметр считается необязательным.
Со скалярными значениями все просто:
```
<integer name="user_id" default="0"/> -> array('user_id' => 1)
<string name="login"/> -> array('login' => 'test')
```
И так далее, порядок параметров неважен. **Внимание!** Тип данных в PHP имеет значение, например для типа integer array('user\_id' => '1') выдаст следующую ошибку _user\_id can only be a integer_.

А вот циклы расскажу более подробно. Как было замечено, разработчики биллинга не пришли к единому формату описания.
В нашем примере используется count="size(parameter_value)", в других можно встретить название поля счетчика,
для нашего примера там было бы написано count="parameters_count". Отсюда возникает вопрос, какое имя давать параметру для массива?
Поэтому было принято решение использовать имя атрибута счетчика в качестве имени для параметра массива. В нашем случае будет так:
```
array(
    array(
        'parameter_id' => 0,
        'parameter_value' => 'м',
    ),
    array(
        'parameter_id' => 1,
        'parameter_value' => '13.06.2014',
    ),
);
```
**Внимание!** Параметр обязательный (если параметр ненужен, передайте пустой массив) и должен быть двумерным массивом.

Если попадется элемент error будет выброшено исключение _XML Described error:_, а далее атрибуты ошибки.

C условиями тоже все просто, если истина, то заходим внутрь. И содержание обрабатывается как описано выше.

В итоге получаем минимальный набор параметров для создания пользователя:
```
include 'URFAClient/init.php';

$api = URFAClient::init(array(
    'login'    => 'admin',
    'password' => 'admin',
    'address'  => 'bill.example.org',
));

$result = $api->rpcf_add_user(array(
    'login'=>'test',
    'password'=>'test',
    'parameters_count' => array(),
));
```
В переменную $result попадут данные которые описаны в элементе output. Более расширенные примеры смотри в example.php.

## Возможные проблемы
- Тестировалось на версиях биллинга UTM-5.2.1-008-update6 и UTM-5.3-002-update5
- Не тестировался тип данных long при использовании php x32
- Тестировались не все функции из api.xml
- При обновлении api.xml обязательно проверяйте используемые функции, осообенно которые содержат циклы (это связано в алгоритме расположения элементов)

По возникшим проблемам присылайте лог(URFAClient::trace_log()), api.xml и версию биллинговой системы. Удачи!

## История изменений

**v1.0.2**
- Поправлена поддержка IPv6
- Обновлен api.xml до версии ядра 5.3-002-update8

**v1.0.1**
- Поправлена обработка элемента output