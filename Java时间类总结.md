# Java时间类总结

问题描述：

最近在写一个日志搜索的功能，主要是用户给定一个时间字符串（仅包含日期信息），然后将这个时间之后的日志搜集出来，处理方式的话是根据日志文件的最后修改时间，如果晚于给定时间，说明这个日志文件里面包含了给定时间之后的日志

具体实现：先通过SimpleDateFormat类设置好解析格式，然后去解析一个时间字符串，得到Date对象，通过getTime方法获取时间戳，与文件对象File的lastModified进行比较，最后就能得出结果了

但是事与愿违，测试的时候发现，会拿到给定时间前一天的日志

所以我去看Date.getTime的注释解释

> Returns the number of milliseconds since January 1, 1970, 00:00:00 GMT represented by this Date object.

可见它返回的是当前时间与GMT时区的1970-1-1 0:0:0 的时间差毫秒数

File.lastModified的注解解释是

> A long value representing the time the file was last modified, measured in milliseconds since the epoch (00:00:00 GMT, January 1, 1970), or 0L if the file does not exist or if an I/O error occurs

可见也返回的是文件最后修改日期与GMT时区的1970-1-1 0:0:0 的时间差毫秒数，或者文件不存在/IO error时返回0

时间的解析与比较都是在一台机器上，时区肯定不会有变化的，这两个值之间应该不会有时区导致的差距，最后经过再次测试，是我当时记错了，并不是使用的8-4而是8-3作为参数。这个教训也告诉我们，工作中还是要细致一点，不然很可能会浪费很多无用的时间

不过这也是个机会，学习下Java Date的相关知识与用法。

我想从以下几个方面去学习：

- Date、LocalDate(LocalDateTime、LocalTime)、Instant、Calendar、SimpleDateFormat、TimeZone这几个类的用法

- 日期处理中有什么需要注意的并发问题

- hutool工具类中日期相关的工具类用法

  

### 时间类的用法

#### Date

非常常用的时间表示类，我们经常会用到

```
new Date()
new Date(time)
date.getTime()
```

此外，比较两个时间的先后：

```
date.before(data1)
data.after(date2)
date.equals(data2)
```

### Instant

代表了时间线上的瞬时点

使用了一个long变量来记录秒数，并且还有一个int变量来记录多出的纳秒（10-9s），因此精度可以到纳秒,表示的时间不受时区影响

```
Instant.now() 返回使用系统时钟表示的当前时间的Instant对象
Instant.ofEpoch(long seconds/long milliseconds) 
instant.plus(long amountToAdd, TemporalUnit unit) 当前intsant加上一个秒数
instant.toEpochMilli() 返回时间戳
```

#### Calendar

Java 中用于操作日期和时间的一个类。它提供了许多方法来处理日期和时间，包括获取、设置各种日期和时间字段，以及执行日期和时间的计算。

是一个抽象类，可以通过getInstance方法获得当前时间的实例对象

```
Calendar calendar  = Calendar.getInstance();
calendar.set(2022,Calendar.DECEMBER,24,11,24,22); //设置时间
System.out.println(calendar.get(Calendar.HOUR_OF_DAY));//获取指定字段的值
calendar.setTimeZone(TimeZone.getTimeZone("GMT")); //设置时区
System.out.println(calendar.getTime()); //通过成员变量time记录的时间戳来生产Date time变量会在调用set_field方法设置field值时失效，此时需要重新计算time的值
System.out.println(calendar.get(Calendar.WEEK_OF_MONTH)); // 本月第几周
System.out.println(calendar.get(Calendar.WEEK_OF_YEAR)); // 今年第几周

calendar.add(Calendar.DAY_OF_MONTH, 5); // 增加5天
calendar.add(Calendar.MONTH, -1); // 减少1个月

boolean isBefore = calendar.before(anotherCalendar); // 检查是否在另一个日期之前
boolean isAfter = calendar.after(anotherCalendar); // 检查是否在另一个日期之后
```

#### LocalDate

jdk8新增的时间api，对时间的处理更加的方便,相关类有LocalDate、LocalDateTime、LocalTime

```
        //所有对象均为不可变对象，修改会创建新的对象

        //获取当前年月日
        LocalDate localDate = LocalDate.now();
//        System.out.println(localDate.get(ChronoField.HOUR_OF_DAY)); 抛出不支持的field
        System.out.println(localDate.get(ChronoField.DAY_OF_YEAR));
        System.out.println(localDate.get(ChronoField.YEAR));
        //构造指定的年月日
        LocalDate localDate1 = LocalDate.of(2019, 9, 10);
        LocalDate localDate2 = localDate1.plusYears(10); //注意这个方法返回新对象，原有对象不修改
        //判断时间的先后
        System.out.println(localDate.isAfter(localDate2));

        LocalTime localTime = LocalTime.now();
        System.out.println(localTime.getHour());
        int hour1 = localTime.get(ChronoField.HOUR_OF_DAY);
        System.out.println(localTime.getNano());
        LocalTime localTime1 = LocalTime.of(2,23,21);
        LocalTime plus = localTime1.plus(5, ChronoUnit.HOURS);
        System.out.println(plus);

        //创建localDateTime的方式
        LocalDateTime localDateTime = LocalDateTime.now();
        LocalDateTime localDateTime1 = LocalDateTime.of(2019, Month.SEPTEMBER, 10, 14, 46, 56);
        LocalDateTime localDateTime2 = LocalDateTime.of(localDate, localTime);
        LocalDateTime localDateTime3 = localDate.atTime(localTime);
        LocalDateTime localDateTime4 = localTime.atDate(localDate);
        LocalDateTime with = localDateTime.with(ChronoField.YEAR, 2025);
        LocalDateTime localDateTime5 = localDateTime1.withHour(23);

        //对日期的便捷操作
        //1.获取本月的第一天
        localDate = localDate.with(TemporalAdjusters.firstDayOfMonth());
        //2.获取本月的最后一天
        localDate = localDate.with(TemporalAdjusters.lastDayOfMonth());
        //3.本年第一天，输出:2019-01-01
        localDate = localDate.with(TemporalAdjusters.firstDayOfYear());
        //4.本年最后一天，输出：2019-12-31
        localDate = localDate.with(TemporalAdjusters.lastDayOfYear());
        //5.下一个周几的操作
        //注意，2019-10-22是周二，所以下一个周四就是2019-10-24,如果是下一个周一，那就是:2019-10-28了
        //当然也有对应的previes方法,就是上一个周几
        localDate = localDate.with(TemporalAdjusters.next(DayOfWeek.THURSDAY));
        localDate = localDate.with(TemporalAdjusters.previous(DayOfWeek.THURSDAY));
        //6.本月第2周的周五
        localDate = localDate.with(TemporalAdjusters.dayOfWeekInMonth(2,DayOfWeek.FRIDAY));
        //7.还有下一个月的第一天，当然还有下一年
        localDate = localDate.with(TemporalAdjusters.firstDayOfNextMonth());

        //时间格式化
        localDate = LocalDate.of(2019, 9, 10);
        String s1 = localDate.format(DateTimeFormatter.BASIC_ISO_DATE);
        String s2 = localDate.format(DateTimeFormatter.ISO_LOCAL_DATE);
        //自定义格式化
        DateTimeFormatter dateTimeFormatter = DateTimeFormatter.ofPattern("dd/MM/yyyy");
        String s3 = localDate.format(dateTimeFormatter);
        //解析
        localDate1 = LocalDate.parse("20190910", DateTimeFormatter.BASIC_ISO_DATE);
        localDate2 = LocalDate.parse("2019-09-10", DateTimeFormatter.ISO_LOCAL_DATE);
```

#### 相互转换

```
    //java.util.Date类型转LocalDateTime
    public static LocalDateTime dateToLocalDateTime(Date date) {
        Instant instant = date.toInstant();
        ZoneId zoneId = ZoneId.systemDefault();
        return instant.atZone(zoneId).toLocalDateTime();
    }
    
    //java.time.LocalDateTime转java.util.Date
    public static Date localDateTimeToDate(LocalDateTime localDateTime) {
        ZoneId zoneId = ZoneId.systemDefault();
        ZonedDateTime zdt = localDateTime.atZone(zoneId);
        return Date.from(zdt.toInstant());
    }
```

### 时间处理的并发问题

#### SimpleDateFormat

多线程下会产生并发问题，因为其使用的calendar是静态变量

使用ThreadLocal包装或者随用随创建

#### Date

Date类是可变的，在多线程环境下可能会导致一些并发错误，使用LocalDate类型，在修改属性时会创建一个新的对象，避免出现一些并发问题。

#### DateTimeFormatter

相比SimpleDateFormat，DateTimeFormatter是不可变变量，并且线程安全，全局只需要一个实例即可。

### Hutool日期相关工具类

#### DateUtil

https://doc.hutool.cn/pages/DateUtil/

#### LocalDateTimeUtil

https://doc.hutool.cn/pages/LocalDateTimeUtil/