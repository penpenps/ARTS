## Backgroud
Before Java 8, we are always troubled to handle various of date and time related problems, such as convert between string and datetime with given format, adopt timeline in different time zone, calculate difference between multiple datatime objects and so on. In Java 8, it provides a unified datetime package in `java.time`. Next, let's go through this package and show cases about how to use it.

## Classes
The classes in `java.time` are all immutable and safe-thread. That means we can decalre a datetime object in an abstract class and use it in multi-thread program.
### `LocalDate`
This class presents a date without zone information.
```java
LocalDate localDate = LocalDate.now();          //Current date without zone information
Month month = localDate.getMonth();             //The month of now, e.g. NOVEMBER
int dayOfMonth = localDate.getDayOfMonth();     //The day of month 
boolean isLeapYear = localDate.isLeapYear();    //Is leap year or not
```
It can also be constructed by given year, month and day.
```java
LocalDate localDate = LocalDate.of(2018, 11, 18);
```
### `LocalTime`
Similar with `LocalDate`, difference is `LocalTime` only contains hour, minute and seconds these clock information and `LocalDate` only includes year, month and day values.
```java
LocalTime localTime = LocalTime.now();      //Current time
int hour = localTime.getHour();             //Hour
int minute = localTime.getMinute();         //Minute
int second = localTime.getSecond();        //Second
int nano = localTime.getNano();             //Nano second
```
### `LocalDateTime`
Obviously, it is a combination of `LocalDate` and `LocalTime`. A `LocalDateTime` object can be created by `now` and `of` function or translated from a `LocalDate` object or a `LocalTime` object by `atTime` or `atDate` methods.
```java
LocalDateTime localDateTime1 = LocalDateTime.of(2018, 11, 18, 20, 0, 0, 0);
LocalDateTime localDateTime2 = localDate.atTime(localTime);
LocalDateTime localDateTime3 = localTime.atDate(localDate);
```
With `toLocalDate()` and `toLocalTime()`, we can get the responsible `LocalDate` and `LocalTime` fields.
```java
LocalDate shadowLocalDate = localDateTime1.toLocalDate();
LocalTime shadowLocalTime = localDateTime1.toLocalTime();
```
### `ZonedDateTime`
`ZonedDateTime` object can repsent not only datetime information like `LocalDateTime` but also contains the offset and time zone value for a datetime. Here is description in Java Doc:
> A ZonedDateTime holds state equivalent to three separate objects, a LocalDateTime, a ZoneId and the resolved ZoneOffset. The offset and local date-time are used to define an instant when necessary. The zone ID is used to obtain the rules for how and when the offset changes. The offset cannot be freely set, as the zone controls which offsets are valid.
What worth to note is that, the offset value cannot be set becuase this value should exactly follow time zone rules.
```java
ZonedDateTime zonedDateTime = ZonedDateTime.now();  //Current datetime with zone info, e.g. 2018-11-18T12:36:10.664Z[Etc/UTC]
int year = zonedDateTime.getYear();                 //Year
Month month = zonedDateTime.getMonth();             //Month
int day = zonedDateTime.getDayOfYear();             //Day
int hour = zonedDateTime.getHour();                 //Hour
int minute = zonedDateTime.getMinute();             //Minute
int second = zonedDateTime.getSecond();             //Second
ZoneOffset offset = zonedDateTime.getOffset();      //The zone offset, such as +08:00
ZoneId zone = zonedDateTime.getZone();              //Time zone, e.g. Asia/Singapore
```
### `Instant`
This class contains a `long` field to store seconds and a `int` field to store nano seconds and it is often used to get a timestamp in events:
> The range of an instant requires the storage of a number larger than a long. To achieve this, the class stores a long representing epoch-seconds and an int representing nanosecond-of-second, which will always be between 0 and 999,999,999. The epoch-seconds are measured from the standard Java epoch of 1970-01-01T00:00:00Z where instants after the epoch have positive values, and earlier instants have negative values. For both the epoch-second and nanosecond parts, a larger value is always later on the time-line than a smaller value.
```java
Instant instant = zonedDateTime.toInstant();        //Get the instant object
long epochSecond = instant.getEpochSecond();        //the epoch seconds comparing with 1970-01-01T00:00:00Z
```

### `Duration`
This class is similar with `Instant` contains seconds and nano seconds values to repsent the diffrence between two time points. It can be created from two datetime objects or give it a absolute time gap.
```java
LocalDateTime from = LocalDateTime.of(2018, Month.JANUARY, 18, 10, 7, 0);    // 2017-01-18 10:07:00
LocalDateTime to = LocalDateTime.of(2018, Month.FEBRUARY, 18, 10, 7, 0);     // 2017-02-18 10:07:00
Duration duration = Duration.between(from, to);     // Duration between 2018-01-18 10:07:00 and 2018-02-18 10:07:00

long days = duration.toDays();              // Days of this duration
long hours = duration.toHours();            // Hours of this duration
long minutes = duration.toMinutes();        // Minutes of this duration
long seconds = duration.getSeconds();       // Seconds of this duration
long milliSeconds = duration.toMillis();    // The total milliseconds of this duration
long nanoSeconds = duration.toNanos();      // The total nano seconds of this duration
```
Use `of` to set duration value:
```java
Duration duration1 = Duration.of(5, ChronoUnit.DAYS);       // 5 days
Duration duration2 = Duration.of(1000, ChronoUnit.MILLIS);  // 1 second
```
## Operations on datetime objects
These classes all contains methods to plus or minus on date and time fields, for example, the below code snip shows how to get the day after 5 days of `zonedDatetime`:
```java
ZonedDateTime next5Days = zonedDateTime.plusDays(5);
```
Actually, in reality program, we often face more complicated calculation about datetime, like get the closest Sunday, get the last Thursday of last month. We can use `TemporalAdjusters` class to enable advanced functions to do calculation like that.
```
LocalDate date = LocalDate.now();
LocalDate closestSunday = date.with(nextOrSame(DayOfWeek.SUNDAY));      // get the next closest Sunday
LocalDate lastSaturday = date.with(lastInMonth(DayOfWeek.SATURDAY));   // get the last Saturday
```
Note: To use `TemporalAdjusters`, you must import it like that: `import static java.time.temporal.TemporalAdjusters.*;`

If these methods provided in `TemporalAdjusters` cannot satisfy your requirement, you can also implement a customized adjuster for your logic.

## Datetime Formatter
In java 8 time package, it provides serveral predefined formatters, you can use it directly. Moreover, you can also use string pattern to format datetime string.
```java
ZonedDateTime dateTime = ZonedDateTime.now();
String basicISODateStr = dateTime.format(DateTimeFormatter.BASIC_ISO_DATE); //20181118Z
String ISOLocalDateTimeStr = dateTime.format(DateTimeFormatter.ISO_LOCAL_DATE_TIME);  //2018-11-18T13:43:39.268
String ISOOffsetDateTimeStr = dateTime.format(DateTimeFormatter.ISO_OFFSET_DATE_TIME); //2018-11-18T13:43:39.268Z
String ISOZonedDateTimeStr = dateTime.format(DateTimeFormatter.ISO_ZONED_DATE_TIME);  //2018-11-18T13:43:39.268Z[Etc/UTC]
String RfcDateTimeStr = dateTime.format(DateTimeFormatter.RFC_1123_DATE_TIME);  //Sun, 18 Nov 2018 13:43:39 GMT
String formatStr1 = dateTime.format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"));  //2018-11-18 13:45:21
String formatStr2 = dateTime.format(DateTimeFormatter.ofPattern("yyyy/MM/dd HH:mm:ss z"));  //2018/11/18 13:47:11 UTC
```

In turn, we can also use `DateTimeFormatter` to parse datetime from string:
```java
String dateTimeStr = "2018-11-18 13:45:21";
LocalDateTime dateTime1 = LocalDateTime.parse(dateTimeStr, DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"));
```

## Time Zone
In above snips, you maybe notice `ZoneId` class. It is used to determine a datetime object time zone information and represents to standard time zone like "Asia/Shanghai" or "Europe/Paris".

We can set a given zone id for a `ZonedDateTime` object:
```java
ZonedDateTime zonedDateTime = ZonedDateTime.now();  //Current datetime with zone info, e.g. 2018-11-18T12:36:10.664Z[Etc/UTC]
ZoneId shanghaiZoneId = ZoneId.of("Asia/Shanghai");
ZonedDateTime shDateTime = ZonedDateTime.of(zonedDateTime.toLocalDateTime(), shanghaiZoneId);    //2018-11-18T13:55:31.338+08:00[Asia/Shanghai]
```
We can parse a string with timezone info to a `ZonedDatetime`:
```java
String dateTimeStr2 = "2018-11-18T13:55:31.338+08:00[Asia/Shanghai]";
ZonedDateTime shadowDateTime = ZonedDateTime.parse(dateTimeStr2, DateTimeFormatter.ISO_ZONED_DATE_TIME);
```
Note: If the datetime string doesn't contain zone Id, like that "2018-11-18T13:55:31.338+08:00", then `shadowDateTime` will use its offset `+08:00` as it cannot determine which zone Id it belongs to, because there are serveral zone Id having the same offset.

## Refrences
- https://lw900925.github.io/java/java8-newtime-api.html
- https://docs.oracle.com/javase/8/docs/api/java/time/ZonedDateTime.html
- https://docs.oracle.com/javase/8/docs/api/java/time/temporal/TemporalAdjusters.html
