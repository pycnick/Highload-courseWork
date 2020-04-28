### 1. Выбор темы
* Яндекс.Музыка

### 2. Определение возможного диапазона нагрузок

* Месячная аудитория [15 млн](https://vc.ru/services/98608-ot-kataloga-i-poiska-do-personalnogo-muzykalnogo-pomoshchnika-kak-izmenilas-yandeks-muzyka-i-chto-ee-zhdet-v-budushchem) 

* Дневная аудитория [1.9 млн](https://radar.yandex.ru/top_list?thematic=culture%2Cmusic)

* Пользователи в среднем слушают музыку [90](https://vc.ru/media/96460-chislo-podpischikov-yandeks-muzyki-vyroslo-v-tri-raza-za-poltora-goda-i-dostiglo-3-mln) минут в день

### 3. Выбор планируемой нагрузки как 60% доля рынка в России

Самыми большими по количество платных подписчиков являются ["Яндекс.Музыка", Boom/VK Music, Apple Music](https://www.dp.ru/a/2019/05/17/Muzikalnaja_strana__Rinku)                        
(3 млн, 1.2 млн, 0.7 млн соответственно) поэтому планируемая нагрузка 60% всего рынка

За час прослушивания музыки расходуется [85Mb](https://yandex.ru/support/music-app-winmobile/search-and-listen/cost.html) 
трафика. Дневная аудитория 1.9 млн * 90 (минут) / 60 (час) * 85 (Mb) --> 242 Tb -- _дневная нагрузка_.

242 Tb * 8 / 864000 = 22.4 Гб/c

242 * 1014 Гб * 8 / 86400 = 22.8 гбит/c.

### 4. Логическая схема базы данных (без выбора СУБД)
![](./pic/Untitled.png)

### 5. Физическая системы хранения (конкретные СУБД, шардинг, расчет нагрузки, обоснование реализуемости на основе результатов нагрузочного тестирования)

Основные дейсвия это скачивание и прослушивание музыки. Поэтому расчитаем rps на отдачу.

``` 
1 900 0000 активных пользователей
x
90 минут
/
24 часа x 60 минут x 60 сек
------------------------------------------
~ 2 000 RPS
``` 
С такой нагрузкой справится 1 сервер PostgreSQL, но возьмем несколько(2). Кроме этого, Cassandra или memcached (различные 
статические индексы) нужны для различных контент- сервисов, например для поиска или метаданных. Аудиофайлы 
хранятся в Amazon S3 и кэшируются в нашем бэкенде или с помощью CDNs для низкой задержки. Вкачестве CDN можно использовать 
Amazon CloudFront. Его нет в России, но он кэширует весь статический и динамический контент сайта и асинхронно отдаёт посетителям. 
Также попутно используется  технология минимизации скриптов и CSS. Также, если это возможно, используется локальное хранилище 
браузера. 

Для серверов с музыкой будем также использовать PostreSQL. Из схемы бд следует, что для хранения музыки пользователя потребуется 8 байт. 
На сервер вместиться с учетом запаса в 50% :

128 * 1024^3 (байты) * 0.5 / (8 * 200 (среднее кол-во песен у пользователя)) = 43 000 000 (хранение записей пользователей)

Берем 2 сервера 

В шардировании нужды нет, ко всем данным доступ одинаковый. Партицируем по userid. 

### 6. Выбор прочих технологий: языки программирования, фреймфорки, протоколы взаимодействия, веб-сервера и т.д. (с обоcнованием выбора)

_web client_: JavaScrip.

_backend_: Чтобы JavaScript мог взаимодействовать с бэкэндом, можно воспользоваться reverse-proxy HAproxy.У Haproxy есть 
возможность указать dns resolvers(resolver обращается к локальному серверу имен) и настроить dns cache. Тем самым Haproxy будет сам обновлять dns cache, если записи в нем 
истекли, и заменять адреса для апстримов в том случае, если они изменились. Сами бэки пишутся на Go или Python.

_app for smartphone_: Java для android, ObjC для ios.

### 7. Расчет нагрузки и потребного оборудования

У нас есть следующие узлы : фронтенд; бэкенд; балансировщик(Software Load Balancers); 
сборщик статистики.

_Фронтенд_: 

объем фронта пример [3 мб](https://habr.com/ru/company/tinkoff/blog/474632/) ->
3 * 80 000(количество пользователей в час) = 240 гб/ч = 533 Мб/c = 0.5 гигабита/c возьмем 64ГБ памяти 32 ядра SSD без RAID 
10G Ethernet(передавать Ethernet пакеты со скоростью 10 гигабит в секунду) Кол-во серверов:

1 900 000 * 2/ 86400  = 43 rps (необходимая нагрузка на отдачу фронтенда)

С такой нагрузкой справится и 1 сервер фронтенда на nginx. Но возьмем 3 сервера.

_Бэкенд_:

1 900 000 * 5/ 86400  = 110 rps (необходимая нагрузка на отдачу фронтенда)
128Гб памяти 32 ядра SATA диск без RAID 10G Ethernet должно хватить 6 серверов (с запасом)

_БД_ :

128Гб памяти 32 ядра SSD RAID6 10G Ethernet
берем 2 для хранения информации о пользователях 

_Сервер для музыки_ :

[130 000 000 всего песен](https://thequestion.ru/questions/82483/skolko_v_mire_pesen_64b53597) * 5.601 МБ (средний 
размер за 3 минуты) = 728Tb 

Предположим, что у нас половина от всех песен в мире.

90 (минут на сервисе) * 1,867(МБ) = 170 МБ аудиоконтента

Положим, что вес всех песен 364Тб;

(80 000 (кол-во пользователей в час) * 170 (МБ среднее скачивание аудиоконтента)) 
/ (756 449 (МБита/час) * 60 * 60 ) ~  15 серверов. Возьмем в два раза больше, чтобы сделать реплики и обеспечить 
отказоустойчивость. 30 серверов со следующей конфигурацией: 

6ТБ [6](https://ddriver.ru/kms_catalog+stat+cat_id-12+nums-78.html) разъемов памяти 32 ядра SATA диск RAID6 10G Ethernet
Общая вместительность:

15 *  6 * 6 = 540 ТБ

_Nginx(Software Load Balancers)_ :

Возьмем 64Гб памяти 32 ядра SATA без RAID 10G Ethernet. 6(пропускная способность + пик нагрузки) для бэка.

_Сборщик статиски_ :

8Гб памяти 32 ядра SATA без RAID 10G Ethernet - 2шт


### 8. Выбор хостинга / облачного провайдера и расположения серверов
Можно двумя способами организовать место хранения данных: в собственных центрах обработки данных получится меньшая задержка
и более стабильная среда. В общедоступном облаке мы получаем гораздо более быструю подготовку аппаратного обеспечения 
и гораздо более динамичные возможности масштабирования. Как было сказано раньше, воспользуемся Amazon Web Services. 
Подсчитаем стоимость хранения  400 Тб, извлечения и передачи данных в месяц в [S3](https://aws.amazon.com/ru/s3/pricing/):

Хранилище:

S3 Standard – универсальное хранилище для всех типов данных, обычно применяется для часто используемых данных.
450 ТБ в месяц	0,022 USD за гигабайт ->  88 000 USD

Запросы и извлечение данных:

S3 Standard	Запросы PUT, COPY, POST, LIST (за 1000 запросов)-- 0,005 USD	
                    GET, SELECT и все остальные запросы (за 1000 запросов) -- 0,0004 USD
2000 * 86400 * 30 = 26 000 USD * 2 = 52 000 USD

Передача исходящих данных из Amazon S3 в Интернет: 

Свыше 150 ТБ в месяц	0,05 USD за 1 ГБ  

240Tb * 30(дней) * 1000(Гб) * 0.05 = 360 000 USD (?)

Итого 500 000 USD


### 9. Схема балансировки нагрузки (входящего трафика и внутрипроектного, терминация SSL)
Балансируем нагрузку с помощью Nginx L7, терминация SSL -> Nginx. Так как будет больше одного Nginx, то воспользуемся
DNS-балансировкой. На одно доменное имя выделяется несколько IP-адресов. Сервер, на который будет направлен клиентский 
запрос, обычно определяется с помощью алгоритма Round Robin.

### 10. Обеспечение отказоустойчивости
Для обеспечения отказоустойчивости для каждого сервера БД храним две репликации (Master-Slave(2)). Таким образом Мастер 
сервер отвечает за изменения данных, а Слейв за чтение. Асинхронность репликации означает, что данные на Слейве могут 
появится с небольшой задержкой. Поэтому, в последовательных операциях необходимо использовать чтение с Мастера, чтобы 
получить актуальные данные. При выходе из строя Слейва, достаточно просто переключить все приложение на работу с Мастером. 
После этого восстановить репликацию на Слейве и снова его запустить.
Если выходит из строя Мастер, нужно переключить все операции (и чтения и записи) на Слейв. Таким образом он станет новым 
Мастером. После восстановления старого Мастера, настроить на нем реплику, и он станет новым Слейвом.

Кроме этого, собираем статистику о сервера и мониторим его. Grafana + Prometeus
