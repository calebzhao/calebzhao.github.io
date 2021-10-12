# mysql的update包含子查询

以例子区域表(sys_region)为例

```
CREATE TABLE `yancao-mp`.`Untitled`  (
  `id` int(0) NOT NULL AUTO_INCREMENT COMMENT '自增ID',
  `pid` int(0) NULL DEFAULT NULL COMMENT '父级id',
  `code` varchar(40) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL COMMENT '代码',
  `name` varchar(50) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL COMMENT '名称',
  `short_name` varchar(50) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL COMMENT '简称',
  `parent_code` varchar(40) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL COMMENT '上级代码',
  `lng` varchar(20) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL COMMENT '经度',
  `lat` varchar(20) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL COMMENT '纬度',
  `sort` int(0) NULL DEFAULT NULL COMMENT '排序',
  `remark` varchar(250) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL COMMENT '备注',
  `status` tinyint(0) NULL DEFAULT NULL COMMENT '状态',
  `tenant_code` varchar(32) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL COMMENT '租户ID',
  `level` varchar(11) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL COMMENT '级别',
  `create_time` datetime(0) NULL DEFAULT NULL COMMENT '创建时间',
  `create_user` bigint(0) NULL DEFAULT NULL COMMENT '创建人',
  `update_time` datetime(0) NULL DEFAULT NULL COMMENT '更新时间',
  `update_user` bigint(0) NULL DEFAULT NULL COMMENT '更新人',
  PRIMARY KEY (`id`) USING BTREE,
  INDEX `Index_1`(`code`, `tenant_code`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8 COLLATE = utf8_general_ci COMMENT = '中国地区信息' ROW_FORMAT = COMPACT;
```

问题是现在pid列是空的，要将pid列的值设为其parent_code列值所对应行的id

- 错误写法

```
update r1 set r1.pid = (
	select id from sys_region r2 where r2.code = r1.parent_code and r2.level = r1.level - 1
) from sys_region r1
```

- 正确写法，多表关联

```
update sys_region r1,  sys_region r2 set  r1.pid = r2.id where r2.code = r1.parent_code and r2.level = r1.level - 1
```



# 1、mysql空间数据类型



## 1.1、数据类型

```
Geometry:可以存储所有的几何类型
Point:简单点
LINESTRING:简单线
POLYGON:简单面
MULITIPOINT:多点
MULITILINESTRING:多线
MUILITIPOLYGON:多面
GEOMETRYCOLLECTION:任何几何集合
```

在[`MyISAM`](https://dev.mysql.com/doc/refman/5.7/en/myisam-storage-engine.html), [`InnoDB`](https://dev.mysql.com/doc/refman/5.7/en/innodb-storage-engine.html), [`NDB`](https://dev.mysql.com/doc/refman/5.7/en/mysql-cluster.html),  [`ARCHIVE`](https://dev.mysql.com/doc/refman/5.7/en/archive-storage-engine.html) 引擎上都支持空间列



单词释义：

### Spatial 空间

Geometry： 几何



**有2种标准的空间数据格式**

1. WKT文本数据格式 Well-Known Text (WKT) format
2. WKB二进制数据格式 Well-Known Binary (WKB) format

![](https://gitee.com/passion/images/raw/master/20201118122224.gif) 

- 1. 一个点：POINT（15 20）

     请注意，**点坐标无指定分隔逗号**。

- 2. 一根线条，例如有四点组成的线条： LINESTRING（0 0，10 10，20 25，50 60）

     请注意，**点坐标对用逗号分隔**。

- 3. 一个多边形

     一个外环和一个内环组成的多边形：POLYGON（（0 0,10 0,10 10 10 0 0），（5 5, 7 5, 7 7, 5 7, 5 5））

     **首点和尾点的坐标相同才能构成一个面**

     三角形：SELECT ST_GeomFromText('POLYGON((0 0, 10 0, 0 10, 0 0))')

- 4. 多点集合，例如三个点的值：MultIPOINT（0 0，20 20，60 60）

- 5. 多线集合，例如两根线条的集合：MULTILINESTRING（（10 10，20 20），（15 15，30 15））

- 6. 多边形集合，例如两个多边形值的集合：MULTIPOLYGON（（（0 0,10 0,10 10 10,0 0）），（（5 5 7 5 7 7 5 7 5 5）））

- 7. 集合，例如两个点和一条线段的集合：GeometryCollection（POINT（10 10），POINT（30 30），LINESTRING（15 15，20 20））

## 1.2、空间索引（ R-tree索引）

- 非空列才能创建空间索引

  ```
  -- 创建表时建所以
  CREATE TABLE geom (g GEOMETRY NOT NULL, SPATIAL INDEX(g));

  -- 修改表索引
  CREATE TABLE geom (g GEOMETRY NOT NULL);
  ALTER TABLE geom ADD SPATIAL INDEX(g);
  
  -- 直接创建索引
  CREATE SPATIAL INDEX g ON geom (g);
  ```
  
  

##  1.3、相关函数

### 1.3.1、Point(x, y)

``` 
select Point(15, 20)
```

将坐标转换成空间数据

### 1.3.2、ST_GeomFromText(‘wtk格式文本’)

```
-- 可以转换任何几何类型
select ST_GeomFromText('POINT(15 20)')

-- 点
ST_PointFromText('POINT (1 1)')

-- 线
ST_LineStringFromText('LINESTRING(0 0,1 1,2 2)')

-- 面
ST_PolygonFromText('POLYGON((0 0,10 0,10 10,0 10,0 0),(5 5,7 5,7 7,5 7, 5 5))')

-- 集合
ST_GeomCollFromText('GEOMETRYCOLLECTION(POINT(1 1),LINESTRING(0 0,1 1,2 2,3 3,4 4))')

-- 16进制wkb格式
select ST_GeomFromWKB(X'01010000000000000000002E400000000000003440')

-- 将mysql内部存储的空间数据转换成blob类型的wkb格式数据
SELECT ST_AsBinary(Point(15, 20))
```

将wkt格式的文本转换为空间数据

### 1.3.3、ST_X(wkb)

```
SELECT ST_X(Point(15, 20))

SELECT ST_X(ST_GeomFromText('POINT(15 20)'))
```

从空间数据中提取x坐标

### 1.3.4、ST_MPointFromText('MULTIPOINT (1 1, 2 2, 3 3)')

```sql
select ST_MPointFromText('MULTIPOINT (1 1, 2 2, 3 3)')

-- mysql 5.7.9之前不支持这种写法，5.7.9之后支持每个点用圆括号包裹起来
select ST_MPointFromText('MULTIPOINT ((1 1), (2 2), (3 3))')
```

将wkt格式的**多个点**的文本转换为空间数据集合

### 1.3.5、ST_AsText(wkb) 将空间数据以文本形式输出

- mysql5.7.9及**之后**版本输出格式**每个点使用括号包裹起来的**

``` 
SELECT ST_AsText(ST_GeomFromText('MULTIPOINT (1 1, 2 2, 3 3)'));

+---------------------------------+
| ST_AsText(ST_GeomFromText(@mp)) |
+---------------------------------+
| MULTIPOINT((1 1),(2 2),(3 3))   |
+---------------------------------+
```

- mysql5.7.9**之前**版本输出格式**每个点使用括号包裹起来的**

``` 
SELECT ST_AsText(ST_GeomFromText('MULTIPOINT (1 1, 2 2, 3 3)'));

+---------------------------------+
| ST_AsText(ST_GeomFromText(@mp)) |
+---------------------------------+
| MULTIPOINT(1 1,2 2,3 3)         |
+---------------------------------+
```

### 1.3.6、LENGTH(wkb)函数获取wkb格式数据所占用的字节数

```
mysql> SET @g = ST_GeomFromText('POINT(1 -1)');
mysql> SELECT LENGTH(@g);
+------------+
| LENGTH(@g) |
+------------+
|         25 |
+------------+
```

### 1.3.7、HEX(wkb) 显示空间数据的16进制表示

```
mysql> SELECT HEX(@g);
+----------------------------------------------------+
| HEX(@g)                                            |
+----------------------------------------------------+
| 000000000101000000000000000000F03F000000000000F0BF |
+----------------------------------------------------+
```

- 4 bytes for integer SRID (0)
- 1 byte for integer byte order (1 = little-endian)
- 4 bytes for integer type information (1 = `Point`)
- 8 bytes for double-precision X coordinate (1)
- 8 bytes for double-precision Y coordinate (−1)

### 1.3.8、ST _ isvalid (wkb)检查几何数据是否有效（语法有效、几何上有效）

