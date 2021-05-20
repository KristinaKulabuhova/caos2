### Сокеты с установкой соединения
#### Сокет
*Сокет* - это файловый дескриптор, открытый как для чтения, так и для записи. Предназначен для взаимодействия:

* разных процессов, работающих на одном компьютере (хосте);
* разных процессов, работающих на разных хостах.
Создается сокет с помощью системного вызова socket:
```
#include <sys/types.h>
#include <sys/socket.h>

int socket(
  int domain,    // тип пространства имён
  int type,      // тип взаимодействия через сокет
  int protocol   // номер протокола или 0 для авто-выбора
)
```

Сокет позволяет одному процессу общаться сразу со многими.
Механизм сокетов появился ещё в 80-е годы XX века, когда не было единого стандарта для сетевого взаимодействия, и сокеты являлись абстракцией поверх любого механизма сетевого взаимодействия, поддерживая огромное количество разных протоколов.

В современных системах используемыми можно считать несколько механизмов, определяющих пространство имен сокетов; все остальное - это legacy, которое мы дальше рассматривать не будем.

* `AF_UNIX` (man 7 unix) - пространство имен локальных UNIX-сокетов, которые позволяют взаимодействовать разным процессам в пределах одного компьютера, используя в качестве адреса уникальное имя (длиной не более 107 байт) специального файла.
* `AF_INET` (man 7 ip) - пространство кортежей, состоящих из 32-битных IPv4 адресов и 16-битных номеров портов. IP-адрес определяет хост, на котором запущен процесс для взаимодействия, а номер порта связан с конкретным процессом на хосте.
* `AF_INET6` (man 7 ipv6) - аналогично `AF_INET`, но используется 128-разрядная адресация хостов IPv6; пока этот стандарт поддерживается не всеми хостерами и провайдерами сети Интернет.
* `AF_PACKET` (man 7 packet) - взаимодействие на низком уровне.
Через сокеты обычно происходит взаимодействие одним из двух способов (указывается в качестве второго параметра `type`):

* `SOCK_STREAM` - взаимодействие с помощью системных вызовов `read` и `write` как с обычным файловым дескриптором. В случае взаимодействия по сети, здесь подразумевается использование протокола `TCP`.
* `SOCK_DGRAM` - взаимодейтсвие без предвариательной установки взаимодействия для отправки коротких сообщений. В случае взаимодействия по сети, здесь подразумевается использование протокола `UDP`.

### Пара сокетов
Иногда сокеты удобно использовать в качестве механизма взаимодействия между разными потоками или родственными процессами: в отличии от каналов, они являются двусторонними, и кроме того, поддерживают обработку события "закрытие соединения". Пара сокетов создается с помощью системного вызова `socketpair`:
```
int socketpair(
  int domain,    // В Linux поддерживатся только AF_UNIX
  int type,      // SOCK_STREAM или SOCK_DGRAM
  int protocol,  // Только значение 0 в Linux
  int sv[2]      // По аналогии с pipe, массив из двух int
)
```
В отличии от неименованных каналов, которые создаются системным вызовом `pipe`, для пары сокетов не имеет значения, какой элемент массива `sv` использовать для чтения, а какой - для записи, - они являются равноправными.

### Использование сокетов в роли клиента
Сокеты могут участвовать во взаимодействии в одной из двух ролей. Процесс может быть сервером, то есть объявить некоторый адрес (имя файла, или кортеж из IP-адреса и номера порта) для приема входящих соединений, либо выступать в роли клиента, то есть подключиться к какому-то серверу.

Сразу после создания сокета, он ещё не готов к взамиодействию с помощью системных вызовов `read` и `write`. Установка взаимодействия с сервером осуществляется с помощью системного вызова `connect`. После успешного выполнения этого системного вызова - взаимодействие становится возможным до выполнения системного вызова `shutdown`.
```
int connect(
  int sockfd,                  // файловый дескриптор сокета

  const struct sockaddr *addr, // указатель на *абстрактную*
                               // структуру, описывающую
                               // адрес подключения

  socklen_t addrlen            // размер реальной структуры,
                               // которая передается в
                               // качестве второго параметра
)
```
Поскольку язык Си не является объектно-ориентированным, то нужно в качестве адреса передавать:

1. Структуру, первое поле которой содержит целое число со значением, совпадающим с `domain` соответствующего сокета
2. Размер этой структуры.
Конкретными стурктурами, которые "наследуются" от абстрактной структуры `sockaddr` могут быть:

1. Для адресного пространства UNIX - стрктура `sockaddr_un`
```
#include <sys/socket.h>
#include <sys/un.h>

struct sockaddr_un {
  sa_family_t   sun_family;    // нужно записать AF_UNIX
  char          sun_path[108]; // путь к файлу сокета
};
```
2. Для адресации в IPv4 - структура `sockaddr_in`:
```
#include <sys/socket.h>
#include <netinet/in.h>

struct sockaddr_in {
  sa_family_t    sin_family; // нужно записать AF_INET
  in_port_t      sin_port;   // uint16_t номер порта
  struct in_addr sin_addr;   // структура из одного поля:
                             // - in_addr_t s_addr;
                             //   где in_addr_t - это uint32_t
};
```
3. Для адресации в IPv6 - структура `sockaddr_in6`:
```
#include <sys/socket.h>
#include <netinet/in.h>

struct sockaddr_in6 {
  sa_family_t    sin6_family; // нужно записать AF_INET6
  in_port_t      sin6_port;   // uint16_t номер порта
  uint32_t       sin6_flowinfo; // дополнительное поле IPv6
  struct in6_addr sin6_addr;  // структура из одного поля,
                              // объявленного как union {
                              //     uint8_t  [16];2

                              // т.е. размер in6_addr - 128 бит
  uint32_t       sin6_scope_id; // дополнительное поле IPv6
};
```
### Адреса в сети IPv4
Адрес хоста в сети IPv4 - это 32-разрядное беззнаковое целое число в *сетевом порядке байт*, то есть Big-Endian. Для номеров портов - аналогично.

Одному имени могут соответсвовать несколько адресов (распределение нагрузки)
и одному адресу соответствует несколько адресов.

Конвертация порядка байт из сетевого в системный и наоборот осуществляется с помощью одной из функций, объявленных в `<arpa/inet.h>`:

* `uint32_t htonl(uint32_t hostlong)` - 32-битное из системного в сетевой порядок байт;
* `uint32_t ntohl(uint32_t netlong)` - 32-битное из сетевого в системный порядок байт;
* `uint16_t htons(uint16_t hostshort)` - 16-битное из системного в сетевой порядок байт;
* `uint16_t ntohs(uint16_t netshort)` - 16-битное из сетевого в системный порядок байт.

IPv4 адреса обычно записывают в десятичной записи, отделяя каждый байт точкой, например: `192.168.1.1`. Такая запись может быть конвертирована из текста в 32-битный адрес с помощью функций `inet_aton` или `inet_addr`.

### Закрытие сетевого соединения
Системный вызов `close` предназначен для закрытия файлового дескриптора (всего 1024 файловых дескриптора), и его нужно вызывать для того, чтобы освободить запись в таблице файловых дескрипторов. Это является необходимым, но не достаточным требованием при работе с TCP-сокетами.

Помимо закрытия файлового дескриптора, хорошим тоном считается уведомление противоположной стороны о том, что сетевое соединение закрывается

Это уведомление осуществляется с помощью системного вызова shutdown.

### Использование сокетов в роли сервера
Для использования сокета в роли сервера, необходимо выполнить следующие действия:

1. Связать сокет с некоторым адресом (именем). Для этого используется системный вызов `bind`, параметры которого точно такие же, как для системного вызова `connect`. Если на компьютере более одного IP-адреса, то адрес `0.0.0.0` означает "все адреса". Часто при отладке и возникает проблема, что порт с определенным номером уже был занят на предыдущем запуске программы (и, например, не был корректно закрыт). Это решается принудительным повторным использованием адреса:
```
// В релизной сборке такого обычно быть не должно!
#ifdef DEBUG
int val = 1;
setsockopt(lfd, SOL_SOCKET, SO_REUSEADDR, &val, sizeof(val));
setsockopt(lfd, SOL_SOCKET, SO_REUSEPORT, &val, sizeof(val));
#endif
```
```
int bind(int socket, const struct sockaddr *addr, socklen_t address_len)
// где
// struct sockaddr_in - адрес IPv4
// struct sockaddr_in6 - адрес IPv6
// struct sockaddr_un - адрес локального UNIX-сокета
```
2. Создать очередь, в которой будут находиться входящие, но ещё не принятые подключения. Это делается с помощью системного вызова `listen`, который принимает в качестве параметра максимальное количество ожидающих подключений. Для Linux это значение равно 128, определено в константе `SOMAXCONN`.

3. Принимать по одному соединению с помощью системного вызова `accept`. Второй и третий параметры этого системного вызова могуть быть `NULL`, если нас не интересует адрес того, кто к нам подключился. Системный вызов `accept` блокирует выполнение до тех пор, пока не появится входящее подключение. После чего - возвращает файловый дескриптор нового сокета, который связан с конкретным клиентом, который к нам подключился.

---

### Протокол UDP
#### Схема взаимодействия TCP/IP
После создания сокета типа `SOCK_STREAM`, он должен быть подключен к противоположной стороне с помощью системного вызова `connect`, либо принять входящее подключение с помощью системного выхова `accept`.

После этого становится возможным сетевое взаимодействие с использованием операций ввода-вывода.

Сетевое взаимодействие по TCP/IP (создание сокета с параметрами `AF_INET` и `SOCK_STREAM`) подразумевает, что ядро операционной системы преобразует непрерывный поток данных в последовательность TCP-сегментов, упакованных в IP-пакеты, и наоборот.

### Сокеты UDP
Механизм отправки сообщений по UDP подразумевает передачу данных без предварительной установки соединения. Сокет, ориентированный на отправку UDP-сообщений имеет тип `SOCK_DGRAM` и используется совместно с адресацией IPv4 (`AF_INET`) либо IPv6 (`AF_INET6`).
```
// Создание сокета для работы по UDP/IP
int sockfd = socket(AF_INET, SOCK_DGRAM, 0);
```
Как и в случае с TCP, для адресация UDP подразумевает, что помимо IP-адреса хоста необходимо определиться с номером порта, который обслуживает отдельный процесс.

### Системные вызовы для передачи и приема данных без установки соединения
```
// Отправить пакет данных
ssize_t sendto(int sockfd,                  // сокет
               const void *buf, size_t len, // данные и размер
               int flags,                   // дополнительные опции
               // адрес назначения (и его размер как для bind/connect)
               const struct sockaddr *dest_addr, socklen_t addrlen);

// Получить пакет данных
ssize_t recvfrom(int sockfd,             // сокет
                 void *buf, size_t len,  // данные и размер
                 int flags,              // дополнительные опции
                 // адрес отправителя (и размер как для accept)
                 const struct sockaddr *src_addr, socklen_t *addrlen); 
```
Cистемный вызов `sendto` предназначен для отправки сообщения. Поскольку предварительно соединение не было установлено, то обязательным является указание адреса назначения: IP-адрес хоста и номер порта.

Системный вызов `recvfrom` предназначен для приема сообщения, и является блокирующим до тех пор, пока придет хотя бы одно сообщение UDP.

Размер буфера, в который `recvfrom` должен записать данные, должен быть достаточного размера для хранения сообщения, в противном случае данные, которые не влезли в буфер, будут потеряны.

Для того, чтобы иметь возможность принимать данные по UDP, необходимо анонсировать прослушивание определенного порта с помощью системного вызова `bind`; параметры адреса для `recvfrom` предназначены только для получения информации об отправителе, и являются опциональными (эти значения могут быть NULL).

### Взаимодействие на уровне сетевого интерфейса
#### Пакетные сокеты
Система Linux позволяет взаимодействовать с сетевыми устройствами на низком уровне, используя специальный тип сокетов: пакетные сокеты `AF_PACKET`.

Более подробно работа с сокетами на низком уровне рассмотрена в статье Introduction to RAW Sockets

Для создания таких сокетов требуются либо права `root`, либо настройка `cap_net_raw`, в противном случае системный вызов `socket` вернет значение `-1`.

При работе с обычными TCP или UDP сокетами, ядро операционной системы полностью абстрагирует пользовательский процесс от дополнительной информации, связанной с доставкой сетевых данных.

При работе с пакетными сокетами необходимо самостоятельно реализовывать обработку требуемых заголовков.

Существует два уровня абстракции для пакетных сокетов: передача данных, которые заворачиваются в стандартный фрейм Ethernet (`AF_PACKET, SOCK_DGRAM`), там и полностью произвольный поток данных (`AF_PACKET, SOCK_RAW`), который может быть использован, например, для отправки широковещательных Ethernet-фреймов.

![](https://github.com/victor-yacovlev/mipt-diht-caos/blob/master/practice/sockets-udp/raw-sockets-figure.png)

### Бинарные заголовки сетевых протоколов
Для работы с заголовками сетевых протоколов средствами языков Си/C++ можно использовать обычные структуры.

Порядок, в котором объявлены поля структуры в тексте программы, является при этом существенным, поскольку он соответствует тому, в каком порядке хранятся данные. Кроме того, необходимо учитывать тот факт, что компиляторы оптимизируют код, выравнивания поля структур в соответствии с особенностями архитектур процессоров, и необходимо явным образом указывать использование "упакованных" структур.

**Пример***: заголовок Ethernet-кадра может быть представлен следующим образом.
```
typedef struct {
    /* MAC-адрес получателя, 6 байт */
    uint8_t destination[6];
    /* MAC-адрес отправителя, 6 байт */
    uint8_t source[6];
    /* Тип передаваемого пакета */
    uint16_t type;
} __attribute__((__packed__)) ethernet_header_t;
```
Кроме того, необходимо помнить о том, что большинство сетевых протоколов подразумевают использование сетевого порядка байт, поэтому нужно использовать функции `htons`, `ntohs`, и др., для того, чтобы правильно представлять целочисленные значения.

### Адресация без IP-адреса
У каждого сетевого интерфейса есть имя в системе, например `eth0` или `wlan0`, которое можно посмотреть в выводе команды `ifconfig`, и *порядковый номер (индекс)*. У каждого, даже не настроенного, сетевого интерфейса есть свой аппаратный адрес (MAC-адрес), размер которого обычно 6 байт.

При адресации через семейство протоколов `AF_PACKET` используется структура `sockaddr_ll`:
```
struct sockaddr_ll {
	unsigned short sll_family;   /* Always AF_PACKET */
	unsigned short sll_protocol; /* Physical-layer protocol */
	int            sll_ifindex;  /* Interface number */
	unsigned short sll_hatype;   /* ARP hardware type */
	unsigned char  sll_pkttype;  /* Packet type */
	unsigned char  sll_halen;    /* Length of address */
	unsigned char  sll_addr[8];  /* Physical-layer address */
};
```
Поле `sll_family` должно иметь значение `AF_PACKET` (поскольку необходимо отделять этот тип адресов от других возможных `struct sockaddr`).

Для отправки низкоуровневых пакетов определенному устройству с использованием протокола Ethernet, когда используется комбинация `(AF_PACKET, SOCK_DGRAM)`, необходимо заполнять поля:

* `sll_protocol` - значение константы из `<linux/if_ether.h>`, которая определят тип пакета данных (протокол), который содержится внутри Ethernet-фрейма;
* `sll_halen` - длина адреса в байтах; для современных реализаций Ethernet это значение равно `6` (константа `ETH_ALEN` из `<linux/if_ether.h>)` ;
* `sll_ifindex` - индекс сетевого устройства; нумерация начинается с `1`, специальное значение `0` может быть использовано только для чтения (признак того, что интересуют данные из любого устройства);
* `sll_addr` - значение MAC-адреса.
* Все остальные поля заполняются драйвером устройства и должны быть инициализированы нулями.
Если используется отправка пакетов без заголовка Ethernet, то есть, используется комбинация `(AF_PACKET, SOCK_RAW)`, то достаточно указать только порядковый индекс сетевого интерфейса `sll_ifindex`.

### Управление устройствами ввода-вывода
Для управления файло-подобными устройствами ввода-вывода используется системный вызов `ioctl`, сигнатура которого такая же, как для `fcntl`: первый аргумент - это файловый дескриптор, затем целочисленная команда, а потом возможны аргументы произвольного типа, в зависимости от команды.

Набор команд для работы с сетевыми интерфейсами описан в man 7 netdevice. Многие из них могут быть выполнены только при наличии соответствующих прав (если модифицируют параметры сетевого интерфейса). С помощью GET-команд, отправляемых через системный вызов `ioctl`, можно выяснить индекс устройства по его имени, связанный с ним MAC-адрес, IP-адрес, если устройство настроено, и т. д.

---

### Протокол HTTP
#### Общие сведения
Протокол HTTP используется преимущественно браузерами для загрузки и отправки контента. Кроме того, благодаря своей простоте и универсальности, он часто используется как высокоуровневый протокол клиент-серверного взаимодействия.

Большинтсво серверов работают с версией протокола `HTTP/1.1`, который подразумевает взаимодействие в текстовом виде через TCP-сокет. Клиент отправляет на сервер текстовый запрос, который содержит:

* Команду запроса
* Заголовки запроса
* Пустую строку - признак окончания заголовков запроса
* Передаваемые данные, если они подразумеваются
В ответ сервер должен отправить:

* Статус обработки запроса
* Заголовки ответа
* Пустую строку - признак окончания заголовков ответа
* Передаваемые данные, если они подразумеваются
Стандартным портом для `HTTP` является порт 80, для `HTTPS` - порт с номером 443, но это жёстко не регламентировано, и при необходимости номер порта может быть любым.

### Основные команды и заголовки HTTP
* `GET` - получить содержимое по указанному URL;
* `HEAD` - получить только метаинформацию (заголовки) по указанному URL, но не содержимое;
* `POST` - отправить данные на сервер и получить ответ.
Кроме основных команд, в протоколе HTTP можно определять произвольные дополнительные команды в текстовом виде (естественно, для этого потребуется поддержка как со стороны сервера, так и клиента). Например, расширение WebDAV протокола HTTP, предназначенное для передачи файлов, дополнительно определяет команды `PUT`, `DELETE`, `MKCOL`, `COPY`, `MOVE`.

Заголовки - это строки вида `ключ: значение`, определяющие дополнительную метаинформацию запроса или ответа.

По стандарту `HTTP/1.1`, в любом запросе должен быть как минимум один заголовок - `Host`, определяющий имя сервера. Это связано с тем, что с одним IP-адресом, на котором работает HTTP-сервер, может быть связано много доменных имен.

Полный список заголовков можно посмотреть в Википедии.

Пример взаимодействия:
```
$ telnet ejudge.atp-fivt.org
$ telnet ejudge.atp-fivt.org 80
Trying 87.251.82.74...
Connected to ejudge.atp-fivt.org.
Escape character is '^]'.
GET / HTTP/1.1  (/ - получить корень)
Host: ejudge.atp-fivt.org (тут должен быть хотя бы 1 обязательный заголовок - имя хоста)
(Connection: close - необязательный заголовок, который закрывает соеденение после ответа на это вопрос)

HTTP/1.1 200 OK
Server: nginx/1.14.0 (Ubuntu)
Date: Tue, 23 Apr 2019 21:18:43 GMT
Content-Type: text/html; charset=UTF-8
Content-Length: 4383
Connection: keep-alive
Last-Modified: Mon, 04 Feb 2019 17:01:28 GMT
ETag: "111f-58114719b3ca3"
Accept-Ranges: bytes

<html>
  <head>
    <meta charset="utf-8"/>
    <title>АКОС ФИВТ МФТИ</title>
  </head>
...
```
### Протокол HTTPS
Протокол HTTPS - это реализация протокола HTTP поверх дополнительного уровня SSL, который, в свою очередь работает через TCP-сокет. На уровне SSL осуществляется проверка сертификата сервера и обмен ключами шифрования. После этого - начинается обычное взаимодействие по протоколу HTTP в текстовом виде, но это заимодейтвие передается по сети в зашифрованном виде.

Аналогом `telnet` для работы поверх SSL является инструмент `s_client` из состава OpenSSL:
```
$ openssl s_client -connect yandex.ru:443
```
### Утилита cURL
Универсальным инструментом для взаимодействия по HTTP в Linux считается curl, которая входит в базовый состав всех дистрибутивов. Работает не только по протоколу HTTP, но и HTTPS.

Основные опции `curl`:

* `-v` - отобразить взаимодействие по протоколу HTTP;
* `-X КОМАНДА` - отправить вместо `GET` произвольную текстовую команду в запросе;
* `-H "Ключ: значение"` - отправить дополнительный заголовок в запросе; таких опций может быть несколько;
* `--data-binary "какой-то текст"` - отправить строку в качестве данных (например, для `POST`);
* `--data-binary @имя_файла` - отправить в качестве данных содержимое указанного файла.

### Библиотека `libcurl`
У утилиты `curl` есть программный API, который можно использовать в качестве библиотеки, не запуская отдельный процесс.

API состоит из двух частей: полнофункциональный асинхронный интерфейс (`multi`), и упрощённый с блокирующим вводом-выводом (`easy`).

Пример использования упрощённого интерфейса:
```
#include <curl/curl.h>

CURL *curl = curl_easy_init();
if(curl) {
  CURLcode res;
  curl_easy_setopt(curl, CURLOPT_URL, "http://example.com");
  res = curl_easy_perform(curl);
  curl_easy_cleanup(curl);
}
```
Этот код эквивалентен команде
```
$ curl http://example.com
```
Дополнительные параметры, эквивалентные отдельным опциям команды `curl`, определяются функцией `curl_easy_setopt`.

Выполнение HTTP-запроса приводит к записи результата на стандартный поток вывода, но обычно бывает нужно получить данные для дальнейшей обработки.

Это делается установкой одной из callback-функций, которая ответственна за вывод:
```
#include <curl/curl.h>

typedef struct {
  char   *data;
  size_t length;
} buffer_t;

static size_t  
callback_function(
            char *ptr, // буфер с прочитанными данными
            size_t chunk_size, // размер фрагмента данных
            size_t nmemb, // количество фрагментов данных
            void *user_data // произвольные данные пользователя
            )
{
  buffer_t *buffer = user_data;
  size_t total_size = chunk_size * nmemb;

  // в предположении, что достаточно места
  memcpy(buffer->data, ptr, total_size);
  buffer->length += total_size;
  return total_size;
}            

int main(int argc, char *argv[]) {
  CURL *curl = curl_easy_init();
  if(curl) {
    CURLcode res;

    // регистрация callback-функции записи
    curl_easy_setopt(curl, CURLOPT_WRITEFUNCTION, callback_function);

    // указатель &buffer будет передан в callback-функцию
    // параметром void *user_data
    buffer_t buffer;
    buffer.data = calloc(100*1024*1024, 1);
    buffer.length = 0;
    curl_easy_setopt(curl, CURLOPT_WRITEDATA, &buffer);

    curl_easy_setopt(curl, CURLOPT_URL, "http://example.com");
    res = curl_easy_perform(curl);

    // дальше можно что-то делать с данными,
    // прочитанными в buffer

    free(buffer.data);
    curl_easy_cleanup(curl);
  }
}
```

---

### Уровни 
![](https://github.com/KristinaKulabuhova/caos2/blob/main/dTPXyqzrbFo.jpg)

#### Уровень сеанса
* До начала взаимодействрия выполняются предвариетльные действия по согласованию сеанса
* После согласования сеанса, передаваемые данные могут отличаться от того, что видно на уровне приложений
* Пример: HTTPS соединение
(используется для шифрования)

---

### SSL - Trancport Layer Secutiry

* Договоренность об отдельном номере порта для TLS-сессий, например 443 вместо 80.
или
* Переключение в режим TLS после установки соединения, например команда STARTTLS в почтовых сервесах.

Криптография в Linux, как и во многих других UNIX-подобных системах реализована с помощью пакета `openssl` или совместимого с ним форка `libressl`.

Пакет предоставляет:

* команду `openssl` для выполнения операций в командной строке
* библиотеку `libcrypto` с реализацией алгоритмов шифрования
* библиотеку `libssl` с реализацией взаимодействия по протоколам SSL и TLS.


### Криптографисекие алгоритмы
* Хеш-функциии: CRC32, MD5, SHA
	- по блоку исходных данных строиться некоторое скалярное значение фиксированного размера
	- операция хеширования не обратима
	- пример ипользования: хранение паролей.
	
* С симметричным ключем: DES, AES, Кузнечик
	- исходные данные разбиваются на блоки фиксированного размера
	- к каждому блоку приненяется алгоритм с использованием ключа того же, либо кратного ему размера
	- операция обратима при использовании того же самого ключа (нужно как-то передать)
	- пример использования: шифрование данных на диске

Бло́чный шифр — разновидность симметричного шифра, оперирующего группами бит фиксированной длины — блоками, характерный размер которых меняется в пределах 64‒256 бит. Если исходный текст (или его остаток) меньше размера блока, перед шифрованием его дополняют. Фактически, блочный шифр представляет собой подстановку на алфавите блоков, которая, как следствие, может быть моно- или полиалфавитной. Блочный шифр является важной компонентой многих криптографических протоколов и широко используется для защиты данных, передаваемых по сети.

Режим шифрования — метод применения блочного шифра (алгоритма), позволяющий преобразовать последовательность блоков открытых данных в последовательность блоков зашифрованных данных. При этом для шифрования одного блока могут использоваться данные другого блока.

Обычно режимы шифрования используются для изменения процесса шифрования так, чтобы результат шифрования каждого блока был уникальным вне зависимости от шифруемых данных и не позволял сделать какие-либо выводы об их структуре. Это обусловлено, прежде всего, тем, что блочные шифры шифруют данные блоками фиксированного размера, и поэтому существует потенциальная возможность утечки информации о повторяющихся частях данных, шифруемых на одном и том же ключе.

Шифрование независимыми блоками

Простейшим режимом работы блочного шифра является режим электронной кодовой книги или режим простой замены (англ. Electronic CodeBook, ECB), где все блоки открытого текста зашифровываются независимо друг от друга. Однако, при использовании этого режима статистические свойства открытых данных частично сохраняются, так как каждому одинаковому блоку данных однозначно соответствует зашифрованный блок данных. При большом количестве данных (например, видео или звук) это может привести к утечке информации о их содержании и дать больший простор для криптоанализа.

Удаление статистических зависимостей в открытом тексте возможно с помощью предварительного архивирования, но оно не решает задачу полностью, так как в файле остается служебная информация архиватора, что не всегда допустимо.

Шифрование, зависящее от предыдущих блоков

Шифрование в режиме сцепления блоков шифротекста
Чтобы преодолеть эти проблемы, были разработаны иные режимы работы.
Общая идея заключается в использовании случайного числа, часто называемого вектором инициализации (IV). В популярном режиме сцепления блоков (англ. Cipher Block Chaining, CBC) для безопасности IV должен быть случайным или псевдослучайным. После его определения, он складывается при помощи операции исключающее ИЛИ с первым блоком открытого текста. Следующим шагом шифруется результат и получается первый шифроблок, который используем как IV для второго блока и так далее. В режиме обратной связи по шифротексту (англ. Cipher Feedback, CFB) непосредственному шифрованию подвергается IV, после чего складывается по модулю два (XOR, исключающее ИЛИ) с первым блоком. Полученный шифроблок используется как IV для дальнейшего шифрования. У режима нет особых преимуществ по сравнению с остальными. В отличие от предыдущих режимов, режим обратной связи вывода (англ. Output Feedback, OFB) циклически шифрует IV, формируя поток ключей, складывающихся с блоками сообщения. Преимуществом режима является полное совпадение операций шифрования и расшифрования. 

(CTR) Counter Режим счетчика - одна из схем симметричного шифрования, при котором зашифрованный блок текста представляет собой побитное сложение блока открытого текста с зашифрованным значением счетчика. Причем блоки счетчика могут инкриминироваться с различной скоростью.

Следует помнить, что вектор инициализации должен быть разным в разных сеансах. В противном случае приходим к проблеме режима ECB. Можно использовать случайное число, но для этого требуется достаточно хороший генератор случайных чисел. Поэтому обычно задают некоторое число — метку, известную обеим сторонам (например, номер сеанса) и называемое nonce (англ. Number Used Once — однократно используемое число). Секретность этого числа обычно не требуется. Далее IV — результат шифрования nonce. 



* С асимметричной парой ключей: RSA
	- генерируется пара ключей: открытый для шифрования и закрытый для дешифрования
	- зная открытй ключ невозможно восстановить закрытый
	- примеры использования: TLS и SSH; цифровая подпись с использованием третей стороны (сертификата)

#### Сертификат открытого ключа
* Содержит:
	- открытый ключ
	- срок действия
	- метаданные
	- цифровую подпись сертификата: хеш сертификата, зашифрованный закрыты ключем
* Генерируется третьей стороной - удостоверяющим центром

#### Взаимодействие по TLS
* Сервем первым отправляет свой вертификат, содержащий открытый кллюч
* У клиента есть база открытых ключей CA, одним из которых подписан сертификат сервера; выполняется проверка подлинности
* Клиент генерирует

### Workflow
Преобразования данных криптографическими функциями подразумевает три стадии:

1. Инициализация - функции, заканчивающиеся на `Init`
2. Добавление очередной порции данных с помощью одной из функций, имя которой заканчивается на `Update`. Этот процесс можно повторять итеративно по мере поступления данных
3. Финализация - функции, оканчивающиеся на `Final`; на этом этапе появляется итоговый результат преобразования.

### Функции `libcrypto`
Функции, имена которых начинаются с `SHA` или `MD5` предназначены для вычисления хеш-значений. Они используют простой workflow из трех стадий.

Для кодирования или декодирования с использованием симметричного ключа используется стандартный workflow, но на стадии инициализации настраивается контекст - переменная, которая хранит состояние шифрующего автомата.

Пример:
```
// Создание контекста
EVP_CIPHER_CTX* ctx = EVP_CIPHER_CTX_new();

// Генерация ключа и начального вектора из
// пароля произвольной длины и 8-байтной соли
EVP_BytesToKey(
  EVP_aes_256_cbc(),    // алгоритм шифрования
  EVP_sha256(),         // алгоритм хеширования пароля
  salt,                 // соль
  password, strlen(password), // пароль
  1,                    // количество итераций хеширования
  key,          // результат: ключ нужной длины
  iv            // результат: начальный вектор нужной длины
);

// Начальная стадия: инициализация
EVP_DecryptInit(
  ctx,                  // контекст для хранения состояния  
  EVP_aes_256_cbc(),    // алгоритм шифрования
  key,                  // ключ нужного размера
  iv                    // начальное значение нужного размера
);
```
Соль используется для усложнения определения прообраза хэш-функции.

Режим счётчика (counter mode, CTR) предполагает возврат на вход соответствующего алгоритма блочного шифрования значения некоторого счётчика, накопленного с момента старта. Режим делает из блочного шифра потоковый, то есть генерирует последовательность, к которой применяется операция XOR с текстом сообщения. Исходный текст и блок зашифрованного текста имеют один и тот же размер блока, как и основной шифр (например, DES или AES).

(CTR) Counter Режим счетчика - одна из схем симметричного шифрования, при котором зашифрованный блок текста представляет собой побитное сложение блока открытого текста с зашифрованным значением счетчика. Причем блоки счетчика могут инкриминироваться с различной скоростью.
