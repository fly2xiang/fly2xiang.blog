title: "地理空间距离计算"
date: 2015-12-08 11:32:12
tags:
- GEO
- Distance
- 地理空间
- 优化
categories: 
- 地理空间距离计算

---


### 直接SQL方式

#### 表结构

```sql
CREATE TABLE `go_goods_coordinate` (
  `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
  `lat` float NOT NULL COMMENT '纬度',
  `lng` float NOT NULL COMMENT '经度',
  PRIMARY KEY (`id`),
  KEY `lat,lng` (`lat`,`lng`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

```
#### 准备测试数据

使用下面的PHP脚本建立基础数据 100W条
```php
$pdo = new PDO('mysql:dbname=test;host=192.168.184.1', 'root', '', array(
    PDO::MYSQL_ATTR_INIT_COMMAND => 'SET NAMES UTF8',
));

$length = 1000000;

$startDate = microtime(true);
echo $startDate.PHP_EOL;

$pdo->beginTransaction();
for($i = 0; $i < $length; $i ++){
    $lat = mt_rand(3000000, 60000000) / 1000000.0;
    $lng = mt_rand(72000000, 136000000) / 1000000.0;
    $statement = $pdo->prepare("INSERT INTO go_goods_coordinate(lat, lng) VALUES(?, ?)");
    $success = $statement->execute(array($lat, $lng));
    if(! $success){
        break;
    }
    if($i % 10000 == 0){
        echo 'inserted 10000 records.'.PHP_EOL;
        sleep(1);
    }
}
$pdo->commit();
$endDate = microtime(true);

echo $endDate.PHP_EOL;
echo ($endDate - $startDate).PHP_EOL;

```

#### SQL查询方式(查询10km内的离我（lat: 50, lng : 100）最近的点前25个)

此种方式由球面距离公式计算点距离，为减少计算与排序的数据量，使用距离估算方式由添加了索引的lat,lng列进行距离估算；另外一种方式是根据GEOHASH估算。

```sql
SET @lat = 50;
SET @lng = 100;
SET @row = 1;

SELECT
  @row := @row + 1 AS row_id, ggc.*, @lat,
  12756274 * ASIN(
    SQRT(
      POWER(SIN((@lat - ggc.lat) * 0.008726646), 2) + COS(@lat * 0.0174533) * COS(ggc.lat * 0.0174533) * POWER(SIN((@lng - ggc.lng) * 0.008726646), 2)
    )
  ) AS len 
FROM
  go_goods_coordinate AS ggc 
WHERE
  ggc.lat < @lat + 0.1
  AND ggc.lat > @lat - 0.1
  AND ggc.lng < @lng + 0.1
  AND ggc.lng > @lng - 0.1
ORDER BY len ASC
LIMIT 25;
```

```
返回了 16 行

执行耗时   : 0.007 sec
传送时间   : 0 sec
总耗时      : 0.007 sec
```

SQL的EXPLAIN结果显示

|id|select_type|table|type|possible_keys|key|key_len|ref|rows|Extra|
|---|---|---|---|---|---|---|---|---|---|
|1|SIMPLE|ggc|range|lat,lng|lat,lng|8|(NULL)|3654|Using where; Using index; Using filesort|

### 使用Sphinx

#### Sphinx配置
```
source go_goods_coordinate
{
        type                    = mysql

        sql_host                = 192.168.184.1
        sql_user                = root
        sql_pass                =
        sql_db                  = test
        sql_port                = 3306  # optional, default is 3306

        sql_query               = \
                SELECT id, lat, lng, RADIANS(lat) AS r_lat, RADIANS(lng) AS r_lng \
                FROM go_goods_coordinate \
                WHERE id > $start AND id < $end
        sql_query_range         = SELECT MIN(id), MAX(id) FROM go_goods_coordinate
        sql_range_step          = 10000

        sql_attr_float          = r_lat
        sql_attr_float          = r_lng
}


index go_goods_coordinate
{
        source                  = go_goods_coordinate
        path                    = /var/lib/sphinx/go_goods_coordinate
}


indexer
{
        mem_limit               = 128M
}


searchd
{
        listen                  = 9312
        listen                  = 9306:mysql41
        log                     = /var/log/sphinx/searchd.log
        query_log               = /var/log/sphinx/query.log
        read_timeout            = 5
        max_children            = 30
        pid_file                = /var/run/sphinx/searchd.pid
        seamless_rotate         = 1
        preopen_indexes         = 1
        unlink_old              = 1
        workers                 = threads # for RT to work
        binlog_path             = /var/lib/sphinx/
}
```
建立索引

```shell
# indexer --all 
Sphinx 2.2.10-id64-release (2c212e0)
Copyright (c) 2001-2015, Andrew Aksyonoff
Copyright (c) 2008-2015, Sphinx Technologies Inc (http://sphinxsearch.com)

using config file '/etc/sphinx/sphinx.conf'...
indexing index 'go_goods_coordinate'...
collected 999800 docs, 13.8 MB
sorted 4.0 Mhits, 100.0% done
total 999800 docs, 13774146 bytes
total 6.741 sec, 2043205 bytes/sec, 148306.59 docs/sec
total 5 reads, 0.029 sec, 7965.8 kb/call avg, 5.9 msec/call avg
total 58 writes, 0.051 sec, 1282.8 kb/call avg, 0.8 msec/call avg
```
启动searchd

```shell
# searchd
```
测试

```php
$lat = 50;
$lng = 100;
$r_lat = deg2rad($lat);
$r_lng = deg2rad($lng);
$circle = 100000000.0;

include '../sphinxapi.php';

$sphinx = new SphinxClient();
$sphinx->SetServer('127.0.0.1', 9312);
$sphinx->SetConnectTimeout(3);
$sphinx->SetArrayResult(true);
$sphinx->SetLimits(0, 2500);
$sphinx->SetGeoAnchor('r_lat', 'r_lng', $r_lat, $r_lng);
$sphinx->SetFilterFloatRange('@geodist', 0.0, $circle);
$sphinx->SetSortMode(SPH_SORT_ATTR_ASC, '@geodist');
$sphinx->SetSelect('id, r_lat, r_lng');
$res = $sphinx->Query("", 'go_goods_coordinate');

echo 'matches: '.count($res['matches']).PHP_EOL;

print_r($res['matches'][0]['attrs']);

echo $res['matches'][0]['attrs']['b_lat'] * 180 / pi().PHP_EOL;
echo $res['matches'][0]['attrs']['b_lng'] * 180 / pi().PHP_EOL;

print_r($res);
```
由查询数的 `$res` 看到查询时间

```shell
[time] => 0.166
```

反复调整查询范围 `$circle` 查询时间都在100~200ms之间，可以得出结论sphinx内部没有对范围查询做优化。

由[sphinx官方的文档](http://sphinxsearch.com/blog/2013/07/02/geo-distances-with-sphinx/)可以看出，sphinx使用的也是球面距离算法（haversine algorithm）。

由[sphinx源码](https://github.com/sphinxsearch/sphinx/blob/master/src/sphinxexpr.cpp#L3472)中可以看出，对距离较近的点sphinx采用了另外的算法。
```cpp
inline float GeodistAdaptiveDeg ( float lat1, float lon1, float lat2, float lon2 )
{
	float dlat = GeodistDegDiff ( lat1-lat2 );
	float dlon = GeodistDegDiff ( lon1-lon2 );

	if ( dlon<13 )
	{
		// points are close enough; use flat ellipsoid model
		// interpolate sqr(k1), sqr(k2) coefficients using latitudes midpoint
		float m = ( lat1+lat2+180 )*GEODIST_TABLE_K/360; // [-90, 90] degrees -> [0, KTABLE] indexes
		int i = int(m);
		i &= ( GEODIST_TABLE_K-1 );
		float kk1 = g_GeoFlatK[i][0] + ( g_GeoFlatK[i+1][0] - g_GeoFlatK[i][0] )*( m-i );
		float kk2 = g_GeoFlatK[i][1] + ( g_GeoFlatK[i+1][1] - g_GeoFlatK[i][1] )*( m-i );
		return (float)sqrt ( kk1*dlat*dlat + kk2*dlon*dlon );
	} else
	{
		// points too far away; use haversine
		static const float D = 2*6371000;
		float a = fsqr ( GeodistFastSin ( dlat*TO_RADF2 ) ) + GeodistFastCos ( lat1*TO_RADF ) * GeodistFastCos ( lat2*TO_RADF ) * fsqr ( GeodistFastSin ( dlon*TO_RADF2 ) );
		return (float)( D*GeodistFastAsinSqrt(a) );
	}
}
```

### 使用Redis的GEO存储方式

截至此时，[Redis的GEO功能](http://redis.io/commands/georadius)还是在unstable版本中，下载安装unstable版本后对Redis进行了测试。

#### 建立测试数据

```php

require '../vendor/autoload.php';
$redis = new \Predis\Client();
$redis->connect();
$redis->select(10);

$length = 1000000;

$startDate = microtime(true);
echo $startDate.PHP_EOL;

for($i = 0; $i < $length; $i ++){
    $lat = mt_rand(3000000, 60000000) / 1000000.0;
    $lng = mt_rand(72000000, 136000000) / 1000000.0;
    $id = $length + $i;
    $redis->executeRaw(array('GEOADD', 'go_goods_coordinate', $lng, $lat, $id));
    if($i % 10000 == 0){
        echo $i.PHP_EOL;
    }
}
$endDate = microtime(true);

echo $endDate.PHP_EOL;
echo ($endDate - $startDate).PHP_EOL;
```

#### 测试

使用以下代码测试性能

```php
require '../vendor/autoload.php';
$redis = new \Predis\Client();
$redis->connect();
$redis->select(10);
$params = array(
    'georadius',
    'go_goods_coordinate',
    '100',
    '50',
    '200',
    'km',
    'withdist',
    'withhash',
    'withcoord',
    'count',
    '10');
$start = microtime(true);
$redis->executeRaw($params);
$end = microtime(true);
echo $end - $start;
```

耗时： 0.016094207763672 也就是大概 16ms

将范围改为 20000000km 再次进行测试

耗时： 0.85910105705261 也就是 859ms

可见Redis内部对查询范围进行了优化，Redis使用的是GEOHASH方式进行的优化。

距离计算算法Redis也是使用球面距离算法进行，可查看[Redis源码](https://github.com/antirez/redis/blob/unstable/deps/geohash-int/geohash_helper.c#L171)

```c
/* Calculate distance using haversin great circle distance formula. */
double geohashGetDistance(double lon1d, double lat1d, double lon2d, double lat2d) {
    double lat1r, lon1r, lat2r, lon2r, u, v;
    lat1r = deg_rad(lat1d);
    lon1r = deg_rad(lon1d);
    lat2r = deg_rad(lat2d);
    lon2r = deg_rad(lon2d);
    u = sin((lat2r - lat1r) / 2);
    v = sin((lon2r - lon1r) / 2);
    return 2.0 * EARTH_RADIUS_IN_METERS *
           asin(sqrt(u * u + cos(lat1r) * cos(lat2r) * v * v));
}
```

### 优化

对于以上三种方式，对于距离的计算都有优化的空间，可参考[美团技术博客的这篇文章](http://tech.meituan.com/lucene-distance.html)。

使用简化的算法，并对cos进行[多项式拟合](https://zh.wikipedia.org/zh-cn/%E6%9B%B2%E7%B7%9A%E6%93%AC%E5%90%88)消除三角函数。

对于Sphinx、Redis可以通过修改源码的方式进行进一步的优化

### 参考链接

[距离估算](http://www.cnblogs.com/LBSer/p/3298057.html)

[球面距离公式](https://en.wikipedia.org/wiki/Haversine_formula)

[sphinx document](http://sphinxsearch.com/blog/2013/07/02/geo-distances-with-sphinx/)

[sphinx github](https://github.com/sphinxsearch/sphinx/blob/master/src/sphinxexpr.cpp#L3472)

[redis github](https://github.com/antirez/redis/blob/unstable/deps/geohash-int/geohash_helper.c#L171)

[美团技术博客](http://tech.meituan.com/lucene-distance.html)

[多项式拟合](https://zh.wikipedia.org/zh-cn/%E6%9B%B2%E7%B7%9A%E6%93%AC%E5%90%88)