---
layout: post
title: PostgreSQL的基本使用
date: 2022-10-24 23:02:23 +0800
category: PostgreSQL
---
# PostgreSQL的基本使用


- 命令行操作
 - 创建一个新的数据库  
 
 ```createdb  <database name>

 - 移除数据库
 ```dropdb <database name>

 - 访问数据库
 ``` psql <database name>
 
 - 启动或者停止PG服务
 ```pg_ctl start|stop -D <DataDIR>

操作数据
 - 创建数据库表
``` CREATE TABLE weather (
    city    varchar(80),
    temp_lo int,    -- low temperature
    temp_hi int,    -- high temperature
    prcp    real,   -- precipitation
    date    date
);

CREATE TABLE cities (
    name    varchar(80),
    location    point
);

 - 删除表

 DROP TABLE tablename;

 - 向表中填充行数据
 ```INSERT INTO weather VALUES ('San Francisco', 46, 50, 0.25, '1994-11-27');

``` INSERT INTO cities VALUES ('San Francisco', '(-194.0, 53.0)');

``` INSERT INTO weather (city, temp_lo, temp_hi, prcp, date)
    VALUES ('San Francisco', 43, 57, 0.0, '1994-11-29');

``` INSERT INTO weather (date, city, temp_hi, temp_lo)
    VALUES ('1994-11-29', 'Hayward', 54, 37);

``` COPY weather FROM '/home/user/weather.txt';
