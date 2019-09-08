# SpringBoot入门遇到的问题：

1.  MYSQL连接出现：The server time zone value '�й���׼ʱ��' is unrecognized or represents more than one time zone. You must configure either the server or JDBC driver (via the serverTimezone configuration property) to use a more specifc time zone value if you want to utilize time zone support. 

   解决方法：

   方法一：在连接的URL后加上：?serverTimezone=UTC

   ```java
   jdbc:mysql://localhost:3306/todo?serverTimezone=UTC
   ```

   方法二：在mysql中设置时区，默认为SYSTEM set global time_zone='+8:00'

2. 