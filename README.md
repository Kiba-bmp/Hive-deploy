# Hive на кластере Team B

Домашка номер три: развернуть Apache Hive так, чтобы с ним могли работать несколько клиентов одновременно, и сделать загрузку данных в партиционированную таблицу.

Эту домашку мы оформили как описание: ниже все шаги, команды и конфиги по порядку. Hive ставится поверх нашего HDFS из первой домашки, на тех же машинах. Team A, как и раньше, не трогаем.

## Почему не embedded

По умолчанию Hive хранит метаданные (какие есть базы, таблицы, партиции) во встроенной Derby прямо внутри клиента. Derby в таком режиме блокирует файлы базы, и второй клиент к ней уже не подключится. Это как раз то, что задание запрещает.

Поэтому у нас три отдельных процесса. PostgreSQL хранит метаданные. Hive Metastore - отдельный сервис, только он ходит в PostgreSQL, остальные спрашивают метаданные у него. HiveServer2 принимает подключения клиентов по JDBC, у каждого подключения своя сессия, и их может быть сколько угодно.

Цепочка получается такая: клиент (beeline) идёт в HiveServer2, тот в Metastore, тот в PostgreSQL. Сами данные таблиц лежат в нашем HDFS.

## Раскладка по узлам

Всё серверное ставим на team-28-01 (10.28.0.13). Там сейчас работают только DataNode и NodeManager, это самый свободный узел. На team-28-nn и так уже NameNode, DataNode, ResourceManager и NodeManager.

На team-28-01 будут PostgreSQL, Metastore, HiveServer2 и клиент beeline. На team-28-nn ставим только beeline, чтобы подключиться вторым клиентом с другой машины. На team-28-00 ничего не меняется.

Подключение как раньше: с ноутбука на edge team-28-en под team28b, оттуда внутрь по ключу team28b_internal.

## Версии

Hive - 4.0.0-alpha-2, как рекомендовано в задании. Hadoop 3.4.3 и Java 11 (Temurin) уже стоят с первой домашки. PostgreSQL 16. JDBC-драйвер PostgreSQL 42.5.1, эту версию использует сам Hive в своей документации.

sudo у team28b нет, поэтому поставить PostgreSQL через apt не получится. Ставим его через micromamba (это маленький менеджер пакетов conda-forge, один бинарник) в домашний каталог. Ничего системного, как и раньше.

## Где всё лежит

Hive в ~/apps/hive, конфиги в ~/apps/hive/conf. PostgreSQL в ~/apps/pgsql, его данные в ~/pg/data, сокет в ~/pg/sock. Логи сервисов Hive в ~/hive/logs, pid-файлы в ~/hive/run, временные файлы в ~/hive/tmp.

В HDFS таблицы Hive лежат в /user/hive/warehouse, сырые файлы для загрузки - в /user/team28b/staging/events.

## Порты

Team A тоже будет делать эту домашку и, скорее всего, займёт стандартные порты. Поэтому мы сдвигаем всё на +100, как в первой домашке.

PostgreSQL у нас на 5532 вместо 5432, и слушает только localhost. Metastore на 9183 вместо 9083. HiveServer2 на 10100 вместо 10000, его Web UI на 10102 вместо 10002.

Перед стартом проверяем на team-28-01, что наши порты свободны:

```bash
ss -ltn | grep -E ':(5532|9183|10100|10102)\b' || echo СВОБОДНЫ
```

## PostgreSQL

Ставим micromamba, через него PostgreSQL:

```bash
mkdir -p ~/apps && cd ~/apps
curl -Ls https://micro.mamba.pm/api/micromamba/linux-64/latest | tar -xj bin/micromamba
~/apps/bin/micromamba create -y -r ~/apps/mamba -p ~/apps/pgsql -c conda-forge postgresql=16
~/apps/pgsql/bin/postgres --version
```

Создаём кластер базы. Через сокет пускаем только своего пользователя (peer), по TCP - только с паролем. Режим trust не ставим: машины общие, и тогда на наш порт мог бы зайти кто угодно.

```bash
mkdir -p ~/pg/sock && chmod 700 ~/pg/sock
~/apps/pgsql/bin/initdb -D ~/pg/data -U team28b -E UTF8 --auth-local=peer --auth-host=scram-sha-256

cat >> ~/pg/data/postgresql.conf << 'EOF'
port = 5532
listen_addresses = 'localhost'
unix_socket_directories = '/home/team28b/pg/sock'
EOF

~/apps/pgsql/bin/pg_ctl -D ~/pg/data -l ~/pg/postgres.log start
```

Создаём пользователя hive и базу metastore. База должна принадлежать hive: начиная с PostgreSQL 15 создавать таблицы в схеме public может только владелец базы, иначе инициализация схемы Hive упадёт с permission denied.

```bash
PSQL="$HOME/apps/pgsql/bin/psql -h $HOME/pg/sock -p 5532 -d postgres"
$PSQL -c "CREATE ROLE hive LOGIN PASSWORD 'hive_team28b';"
$PSQL -c "CREATE DATABASE metastore OWNER hive;"
```

Проверяем, что по TCP с паролем пускает:

```bash
PGPASSWORD=hive_team28b ~/apps/pgsql/bin/psql -h localhost -p 5532 -U hive -d metastore -c "SELECT current_user;"
```

Пароль лежит открытым текстом, для учебного стенда это нормально: база слушает только localhost.

## Установка Hive

```bash
cd ~/apps
curl -L -o hive.tar.gz https://archive.apache.org/dist/hive/hive-4.0.0-alpha-2/apache-hive-4.0.0-alpha-2-bin.tar.gz
tar -xzf hive.tar.gz && mv apache-hive-4.0.0-alpha-2-bin hive && rm hive.tar.gz
```

alpha-2 уже нет на обычных зеркалах Apache, только в archive.apache.org, а он медленный. Архив большой, качается долго.

Кладём JDBC-драйвер PostgreSQL, без него Metastore не подключится к базе:

```bash
curl -L -o ~/apps/hive/lib/postgresql-42.5.1.jar https://repo1.maven.org/maven2/org/postgresql/postgresql/42.5.1/postgresql-42.5.1.jar
```

Дальше Guava. У Hive своя версия этой библиотеки, у Hadoop 3.4.3 новее. Если в classpath попадают обе, всё падает с NoSuchMethodError на Preconditions.checkArgument. Оставляем ту, что от Hadoop:

```bash
rm ~/apps/hive/lib/guava-*.jar
cp ~/apps/hadoop/share/hadoop/common/lib/guava-*.jar ~/apps/hive/lib/
```

## Переменные окружения

JAVA_HOME и HADOOP_HOME уже прописаны в ~/.bashrc с первой домашки, добавляем Hive:

```bash
echo 'export HIVE_HOME=$HOME/apps/hive' >> ~/.bashrc
echo 'export PATH=$HIVE_HOME/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

И hive-env.sh, чтобы Hive точно знал, где Java и наш Hadoop:

```bash
cat > ~/apps/hive/conf/hive-env.sh << 'EOF'
export JAVA_HOME=$HOME/apps/java
export HADOOP_HOME=$HOME/apps/hadoop
export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
export HIVE_CONF_DIR=$HOME/apps/hive/conf
export HADOOP_HEAPSIZE=1024
EOF
```

Через HADOOP_CONF_DIR Hive берёт наш core-site.xml с fs.defaultFS = hdfs://team-28-nn:9100, так что таблицы окажутся в нашем HDFS, а не у Team A. Память ограничили, потому что на узле всего около 2.5 ГБ и там уже работают DataNode и NodeManager.

## hive-site.xml

Весь Hive настраивается одним файлом ~/apps/hive/conf/hive-site.xml:

```xml
<?xml version="1.0"?>
<configuration>
  <property><name>javax.jdo.option.ConnectionURL</name><value>jdbc:postgresql://localhost:5532/metastore</value></property>
  <property><name>javax.jdo.option.ConnectionDriverName</name><value>org.postgresql.Driver</value></property>
  <property><name>javax.jdo.option.ConnectionUserName</name><value>hive</value></property>
  <property><name>javax.jdo.option.ConnectionPassword</name><value>hive_team28b</value></property>

  <property><name>hive.metastore.uris</name><value>thrift://10.28.0.13:9183</value></property>
  <property><name>hive.metastore.port</name><value>9183</value></property>
  <property><name>hive.metastore.warehouse.dir</name><value>/user/hive/warehouse</value></property>
  <property><name>hive.metastore.event.db.notification.api.auth</name><value>false</value></property>

  <property><name>hive.server2.thrift.bind.host</name><value>0.0.0.0</value></property>
  <property><name>hive.server2.thrift.port</name><value>10100</value></property>
  <property><name>hive.server2.webui.port</name><value>10102</value></property>
  <property><name>hive.server2.enable.doAs</name><value>false</value></property>

  <property><name>hive.execution.engine</name><value>mr</value></property>
  <property><name>mapreduce.framework.name</name><value>local</value></property>
  <property><name>hive.exec.dynamic.partition</name><value>true</value></property>
  <property><name>hive.exec.dynamic.partition.mode</name><value>nonstrict</value></property>

  <property><name>hive.exec.local.scratchdir</name><value>/home/team28b/hive/tmp</value></property>
  <property><name>hive.downloaded.resources.dir</name><value>/home/team28b/hive/tmp/${hive.session.id}_resources</value></property>
  <property><name>hive.querylog.location</name><value>/home/team28b/hive/tmp</value></property>
</configuration>
```

Теперь по блокам, что тут зачем.

Первые четыре свойства javax.jdo.option - это и есть отказ от embedded. Без них Hive молча создаст Derby в текущем каталоге.

hive.metastore.uris говорит всем клиентам ходить в метастор по сети, а не открывать базу самим. Адрес пишем через IP, а не имя хоста. Это мы уже проходили во второй домашке: имена узлов на этих машинах резолвятся ещё и в 127.0.1.1, и процесс может привязаться к loopback. По той же причине HiveServer2 явно слушает 0.0.0.0, иначе beeline с team-28-nn до него не достучится.

hive.server2.enable.doAs выключаем. Иначе HiveServer2 пытается выполнять запросы от имени подключившегося пользователя, а для этого надо прописывать proxyuser в core-site.xml. Без этого каждое подключение падает с ошибкой "team28b is not allowed to impersonate". core-site.xml из первой домашки трогать не хотим. Из-за того же proxyuser отключаем и проверку notification api.auth, иначе HiveServer2 получает отказ от метастора.

Движок ставим mr, а MapReduce запускаем в режиме local, то есть прямо внутри HiveServer2. Результат при этом всё равно пишется в HDFS. Для наших объёмов этого хватает, и загрузка не зависит от YARN: MapReduce на YARN мы во второй домашке не настраивали, а NodeManager думает, что у него 8 ГБ памяти вместо реальных 2.5. Предупреждение о том, что MR deprecated, при запросах нормально.

Динамическое партиционирование включаем сразу, оно понадобится для загрузки.

Последние три свойства уводят временные файлы Hive в наш home, а не в общий /tmp.

## Каталоги в HDFS

Один раз, под team28b, HDFS должен быть запущен:

```bash
hdfs dfs -mkdir -p /tmp /user/hive/warehouse /user/team28b/staging/events
hdfs dfs -chmod 1777 /tmp
hdfs dfs -chmod g+w /user/hive/warehouse
mkdir -p ~/hive/logs ~/hive/run ~/hive/tmp ~/hive/data
```

HDFS у нас свой, поэтому стандартные /tmp и /user/hive здесь никому не мешают.

## Инициализация схемы метастора

Это как format у NameNode: делается один раз. Поэтому сначала смотрим, нет ли схемы уже:

```bash
schematool -dbType postgres -info
```

Если ругается, что схемы нет, создаём:

```bash
schematool -dbType postgres -initSchema
```

После этого -info должен показать версию схемы 4.0.0-alpha-2 и адрес jdbc:postgresql://localhost:5532/metastore. Если там где-то мелькает Derby, значит hive-site.xml не подхватился.

Заодно можно убедиться, что таблицы метастора правда в PostgreSQL:

```bash
PGPASSWORD=hive_team28b ~/apps/pgsql/bin/psql -h localhost -p 5532 -U hive -d metastore -c '\dt'
```

Там должны быть DBS, TBLS, PARTITIONS и остальные.

## Запуск

Порядок важен: сначала PostgreSQL, потом Metastore, потом HiveServer2. HiveServer2 при старте идёт в метастор, метастор в базу.

Готовых start-скриптов у Hive нет, поэтому запускаем через nohup и сохраняем pid, чтобы потом остановить ровно свой процесс:

```bash
nohup hive --service metastore -p 9183 > ~/hive/logs/metastore.out 2>&1 &
echo $! > ~/hive/run/metastore.pid
```

Ждём, пока появится порт 9183 в ss -ltn, обычно это до минуты. Потом HiveServer2:

```bash
nohup hive --service hiveserver2 > ~/hive/logs/hiveserver2.out 2>&1 &
echo $! > ~/hive/run/hiveserver2.pid
```

HiveServer2 поднимается долго, порт 10100 может появиться только через минуту-две. Это нормально.

Остановка в обратном порядке, только по своим pid. killall java и pkill java, как и раньше, не используем:

```bash
kill $(cat ~/hive/run/hiveserver2.pid)
kill $(cat ~/hive/run/metastore.pid)
~/apps/pgsql/bin/pg_ctl -D ~/pg/data stop
```

## Проверка и несколько клиентов

Процессы и порты:

```bash
~/apps/java/bin/jps -m
ss -ltn | grep -E ':(5532|9183|10100|10102)\b'
```

В jps должно быть два RunJar: один с HiveMetaStore, второй с HiveServer2.

Первый клиент, с team-28-01:

```bash
beeline -u "jdbc:hive2://10.28.0.13:10100/default" -n team28b
```

Второй клиент, с team-28-nn. Туда ставим только сам Hive (скачать, распаковать, заменить Guava, прописать переменные), без драйвера, без hive-site.xml и без запуска сервисов. beeline нужен только адрес HiveServer2, команда та же самая.

Пока обе сессии открыты, создаём таблицу в одной и делаем SHOW TABLES во второй. Таблица сразу видна, потому что обе сессии работают через один и тот же метастор. В embedded-режиме второй клиент вообще не смог бы подключиться.

Параллельные подключения можно проверить и одной строкой, три клиента одновременно:

```bash
for i in 1 2 3; do beeline -u "jdbc:hive2://10.28.0.13:10100/default" -n team28b -e "SELECT $i;" & done; wait
```

## Web UI

HiveServer2 Web UI открываем через SSH-туннель с ноутбука, команда одинаковая на Linux и macOS:

```bash
ssh -N -L 10102:team-28-01:10102 team28b@2.59.83.133
```

В браузере http://localhost:10102. На вкладке Active Sessions видно все открытые подключения, в том числе с разных узлов.

На 10002 не заходим, это стандартный порт, его может занять Team A.

## Загрузка в партиционированную таблицу

Загрузка в два шага. Сначала сырой CSV кладём в HDFS в каталог staging, над ним внешняя таблица events_staging: Hive просто читает файлы оттуда. Потом одним INSERT переливаем всё в таблицу events, разбитую на партиции по дате. Динамическое партиционирование само раскладывает строки по каталогам event_date=2026-10-01, event_date=2026-10-02 и так далее, перечислять партиции руками не нужно.

Через LOAD DATA делать не стали: так в одну партицию грузится один файл целиком, и данные пришлось бы заранее резать по датам. А тут в файле даты могут быть вперемешку.

INSERT OVERWRITE перезаписывает только те партиции, которые есть во входном файле. Если загрузить тот же файл второй раз, дублей не будет. А если загрузить новый день, старые партиции не пострадают.

Тестовые данные генерируем на awk, 1000 событий за 5 дней:

```bash
awk -v n=1000 'BEGIN {
  srand(28)
  split("view,click,purchase,refund", t, ",")
  print "event_id,user_id,event_type,amount,event_date"
  for (i = 1; i <= n; i++)
    printf "%d,%d,%s,%.2f,2026-10-%02d\n", i, 1+int(rand()*100), t[1+int(rand()*4)], rand()*1000, 1+int(rand()*5)
}' > ~/hive/data/events.csv
```

Кладём файл в staging, старые файлы оттуда предварительно убираем:

```bash
hdfs dfs -rm -r -f -skipTrash '/user/team28b/staging/events/*'
hdfs dfs -put -f ~/hive/data/events.csv /user/team28b/staging/events/
```

Таблицы, файл create_tables.sql:

```sql
CREATE DATABASE IF NOT EXISTS team28b;
USE team28b;

CREATE EXTERNAL TABLE IF NOT EXISTS events_staging (
  event_id BIGINT,
  user_id INT,
  event_type STRING,
  amount DECIMAL(10,2),
  event_date STRING
)
ROW FORMAT DELIMITED FIELDS TERMINATED BY ','
STORED AS TEXTFILE
LOCATION '/user/team28b/staging/events'
TBLPROPERTIES ('skip.header.line.count'='1');

CREATE TABLE IF NOT EXISTS events (
  event_id BIGINT,
  user_id INT,
  event_type STRING,
  amount DECIMAL(10,2)
)
PARTITIONED BY (event_date STRING)
STORED AS ORC
TBLPROPERTIES ('transactional'='false');
```

event_date в events не указан среди обычных колонок, потому что в Hive колонка партиции хранится не в файлах, а в имени каталога. transactional=false написали явно, чтобы таблица не стала ACID, для этого нужен менеджер транзакций, а мы его не настраиваем.

Сама загрузка, файл load_events.sql:

```sql
SET hive.exec.dynamic.partition=true;
SET hive.exec.dynamic.partition.mode=nonstrict;
USE team28b;

INSERT OVERWRITE TABLE events PARTITION (event_date)
SELECT event_id, user_id, event_type, amount, event_date
FROM events_staging
WHERE event_date IS NOT NULL;
```

Колонка партиции должна идти в SELECT последней: Hive берёт значение партиции по позиции, а не по имени. Настройки динамического партиционирования продублировали прямо в файле, чтобы загрузка работала, даже если в hive-site.xml их нет.

Запуск обоих файлов через beeline:

```bash
beeline -u "jdbc:hive2://10.28.0.13:10100/default" -n team28b -f create_tables.sql
beeline -u "jdbc:hive2://10.28.0.13:10100/default" -n team28b -f load_events.sql
```

Чтобы загрузить новые данные, достаточно положить новый CSV в staging и снова выполнить load_events.sql. create_tables.sql повторно запускать не обязательно, но и не страшно, там везде IF NOT EXISTS.

## Как понять, что всё получилось

В beeline:

```sql
SHOW PARTITIONS team28b.events;
SELECT event_date, COUNT(*) FROM team28b.events GROUP BY event_date ORDER BY event_date;
```

Партиций должно быть пять, с 2026-10-01 по 2026-10-05. Сумма по всем датам должна быть 1000, столько строк в CSV без заголовка.

И главное - смотрим в HDFS:

```bash
hdfs dfs -ls /user/hive/warehouse/team28b.db/events
```

Там пять каталогов event_date=..., в каждом ORC-файл. Это значит, что таблица разбита на партиции физически, а не только в метаданных.

## Логи

stdout и ошибки старта сервисов пишутся в ~/hive/logs/metastore.out и ~/hive/logs/hiveserver2.out. Подробный лог Hive по умолчанию лежит в /tmp/team28b/hive.log, каталог назван по нашему пользователю, с Team A не пересекается. Лог PostgreSQL в ~/pg/postgres.log.

Ошибки считаем так же, как в прошлых домашках:

```bash
grep -icE "error|fatal|exception" ~/hive/logs/*.out /tmp/team28b/hive.log
```

Предупреждения про несколько SLF4J bindings - это нормально, Hive и Hadoop приносят каждый свой логгер.

## Изоляция от Team A

Всё ставится в /home/team28b, sudo не нужен. Наши порты 5532, 9183, 10100 и 10102, стандартные 5432, 9083, 10000 и 10002 не занимаем. PostgreSQL слушает только localhost, сокет в каталоге с правами 700, пускает только с паролем или от team28b. Hive работает с нашим HDFS на 9100, к HDFS Team A на 9000 не обращается. Процессы останавливаем только по своим pid-файлам.

## Куда смотреть,если не работает

Если падает с NoSuchMethodError про Preconditions, значит снова конфликт Guava, нужно ещё раз заменить jar.

Если schematool показывает Derby, значит hive-site.xml лежит не в ~/apps/hive/conf или в нём ошибка.

Если при initSchema permission denied for schema public, значит база metastore создана не с OWNER hive.

Если beeline локально подключается, а с team-28-nn нет, скорее всего HiveServer2 слушает на 127.0.1.1. Проверяем ss -ltn и hive.server2.thrift.bind.host.

Если процессы падают с OutOfMemory, уменьшаем HADOOP_HEAPSIZE в hive-env.sh.

Если при старте ClassCastException про URLClassLoader, это несовместимость alpha-2 с Java 11. Тогда рядом ставим Temurin 8 в ~/apps/java8 и указываем его в hive-env.sh.

## Как повторить с нуля

Всё на team-28-01, HDFS должен быть запущен. Ставим PostgreSQL и создаём базу metastore. Ставим Hive, драйвер, меняем Guava, прописываем переменные. Кладём hive-site.xml. Создаём каталоги в HDFS. Один раз делаем initSchema. Запускаем PostgreSQL, Metastore, HiveServer2. Проверяем beeline с team-28-01 и с team-28-nn. Генерируем CSV, кладём в staging, выполняем create_tables.sql и load_events.sql. Проверяем партиции и каталоги в HDFS.
