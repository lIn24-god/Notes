# SQL

## select...from...where...

- SELECT population FROM world  
  WHERE name = 'Germany'；

- SELECT name, population FROM world  
  WHERE name IN ('Sweden', 'Norway', 'Denmark');

- SELECT name, area FROM world  
  WHERE area BETWEEN 200000 AND 250000；

- SELECT name, population FROM world  
  WHERE name LIKE "Al%"；

- SELECT name,length(name) FROM world  
  WHERE length(name)=5 and region='Europe'；
- SELECT name, gdp/population from world  
  WHERE population >= 200000000;
- SELECT name, population/1000000 FROM world  
  WHERE continent = 'South America';
- SELECT name, population, area FROM world  
  WHERE area >= 3000000  
  OR population >= 250000000;
- SELECT name, population, area FROM world  
  WHERE (population >= 250000000 OR area >= 3000000)  
  AND NOT (population >= 250000000 AND area >= 3000000);

---

- SELECT name, ROUND(population / 1000000, 2), ROUND(gdp / 1000000000, 2)  
  FROM world  
  WHERE continent = 'South America';
- SELECT name, ROUND(gdp / population / 1000) * 1000  
  FROM world  
  WHERE gdp >= 1000000000000;

(ROUND()这个函数是用来进行“四舍五入”的，比如ROUND(population,2)就是保留population的两位小数)

(这第二个示例是为了“四舍五入到千位”，有点抽象，反正原理就是先除以1000，再用ROUND保留到个位，最后再把1000乘回来)

---

- SELECT name, capital  
  FROM world  
  WHERE LENGTH(name) = LENGTH(capital);

（LENGTH()这个函数是用来计算字符串长度的）

---

- SELECT name, capital  
  FROM world  
  WHERE LEFT(name, 1) = LEFT(capital, 1)  
  AND name <> capital;

（LEFT()这个函数是用来取特定字母的，比如LEFT(name,1)取的就是name里面的首字母）

（这个符号<>是表示“不等于”的，就是“!=”）

---

- SELECT name  
  FROM world  
  WHERE name LIKE '%a%'  
  AND name LIKE '%e%'  
  AND name LIKE '%i%'  
  AND name LIKE '%o%'  
  AND name LIKE '%u%'  
  AND name NOT LIKE '% %'；
