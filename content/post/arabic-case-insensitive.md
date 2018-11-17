+++
title = "Arabic Case Insensitive In Database Systems: How To Solve Alef With and Without Hamza Problem"
date = 2017-04-15T16:35:17+02:00
lastmod = "2018-11-17"
tags = [ "Database", "Arabic", "MySQL"]
categories = [
    "Database",
    "MySQL",
    "Arabic"
]
description= "This tutorial shows how to make case insenstive search in Arabic language fields in databases. Step by step examples shows 3 different ways to make the search function treat Alef with Hamza 'أ' and Alef without Hamza 'ا' as one character."
+++

When I first started learning database systems, I ran into problems dealing with Arabic text in databases. Since these problems were specific to the Arabic language, I couldn't easily find solutions on the internet. I try in this post to summarize the solutions I know to a common Arabic text problem in database systems, hoping it will help someone struggling to deal with Arabic text in databases.

You have characters which are considered the same in Arabic (like Alef `'ا'` and Alef with Hamza `'أ'`), but these are two different characters in database systems. When you try to make a search function in your website, for example, you want to ignore the differences between these characters. The use of case-insensitive collation like `utf8_unicode_ci` won't solve the problem. The `'ا'` and `'أ'` aren't considered equal in the [character mapping](http://collation-charts.org/mysql60/mysql604.utf8_unicode_ci.middle_eastern.html) of `utf8_unicode_ci`.

There are three solutions for this problem:

+ [Create a custom collation in your database system](#sol1)
+ [Add a normalized field to your table](#sol2)
+ [Use regular expressions in queries](#sol3)

I will show you how to apply these solutions for MySQL. For other database systems, search the documentation for similar solutions. Each solution will be applied to the following example table:

{{< highlight console >}}

+----+--------------+
| id | name         |
+----+--------------+
|  1 | احمد         |
|  2 | أحمد         |
|  3 | أسامه        |
|  4 | أسامة        |
|  5 | اسامه        |
|  6 | اسَامه        |
+----+--------------+
6 rows in set

{{< /highlight >}}

<a name="sol1"></a>
1\. Create a custom collation
----------------------------

This is the recommended solution for most cases. You won't change any data in your database. All you are going to do is simply telling the DBS to treat these characters as one. The steps we will use here are based on MySQL documentation [Adding a UCA Collation to a Unicode Character Set](https://dev.mysql.com/doc/refman/5.7/en/adding-collation-unicode-uca.html).

First, we will need to modify a special MySQL file which contains the character sets and add our new collation there. The file name is `Index.xml`.  The location of the file varies from one system to another. We will get the location of the file from `information_schema` database. This is the database where MySQL stores databases metadata and information. Run the following query on `information_schema` database:

{{< highlight console >}}

SHOW VARIABLES LIKE 'character_sets_dir';
{{< /highlight >}}

You should get a result like the following:

{{< highlight console >}}

+--------------------+----------------------------+
| Variable_name      | Value                      |
+--------------------+----------------------------+
| character_sets_dir | /usr/share/mysql/charsets/ |
+--------------------+----------------------------+

1 row in set (0.00 sec)

{{< /highlight >}}

On my Debian Linux system, the file path is `/usr/share/mysql/charsets/Index.xml`. Backup the file, open it, and go to the element `<charset name="utf8">`. We will add our collation under it.

We need an id and a name for our collation, the range of IDs from [1024 to 2047 is reserved for user-defined collations](https://dev.mysql.com/doc/refman/5.7/en/adding-collation-choosing-id.html), so choose a number in this range. I choose the name to be `utf8_arabic_ci`.

Let's add the collation to the `Index.xml` file and then explain what the collation rules mean. Add the following:

{{< highlight xml >}}

<collation name="utf8_arabic_ci" id="1029">
  <rules>
      <reset>\u0627</reset>   <!-- Alef 'ا' -->
      <i>\u0623</i>           <!-- Alef With Hamza Above 'أ' -->
      <i>\u0625</i>           <!-- Alef With Hamza Below 'إ' -->
      <i>\u0622</i>           <!-- Alef With Madda Above 'آ' -->
  </rules>
  <rules>
      <reset>\u0629</reset>   <!-- Teh Marbuta 'ة' -->
      <i>\u0647</i>           <!-- Heh 'ه' -->
  </rules>
  <rules>
      <reset>\u0000</reset>   <!-- Unicode value of NULL  -->
      <i>\u064E</i>           <!-- Fatha 'َ' -->
      <i>\u064F</i>           <!-- Damma 'ُ' -->
      <i>\u0650</i>           <!-- Kasra 'ِ' -->
      <i>\u0651</i>           <!-- Shadda 'ّ' -->
      <i>\u064F</i>           <!-- Sukun 'ْ' -->
      <i>\u064B</i>           <!-- Fathatan 'ً' -->
      <i>\u064C</i>           <!-- Dammatan 'ٌ' -->
      <i>\u064D</i>           <!-- Kasratan 'ٍ' -->
  </rules>
</collation>

{{< /highlight >}}

This added part tells MySQL that `utf8_arabic_ci` (or whatever you name it) is a child of the utf8 character set, adding the following rules:

+ `'أ','إ','آ'` is the same character as `'ا'` (All the Alef forms are one character)
+ `'ه'` is the same character as `'ة'` (so `"نسمة"` is the same as `"نسمه"`)
+ Tashkil characters are ignored completely as if they aren't there

You can add more rules as you like. After saving the file, restart MySQL server to use our collation, on my Debian I use:

{{< highlight console >}}
sudo service mysql restart
{{< /highlight >}}

Now, go to the table with Arabic text and change the column collation to our new collation. I will do this to my example table using the following query:

{{< highlight console >}}

ALTER TABLE persons
MODIFY name VARCHAR(50)
CHARACTER SET 'utf8'
COLLATE 'utf8_arabic_ci';

{{< /highlight >}}

**Note**: If you are using PHPMyAdmin or some similar interface, don't expect to find the custom collation in the dropdown list of collations. You will have to write a query like the above to change the column to our new collation.

If the query succeeded, then everything should be set. Let's make a search query and see if it is working as it should:

{{< highlight console >}}

SELECT * FROM persons WHERE name = "اسامة";

+----+--------------+
| id | name         |
+----+--------------+
|  3 | أسامه        |
|  4 | أسامة        |
|  5 | اسامه        |
|  6 | اسَامه        |
+----+--------------+
4 rows in set (0.00 sec)

{{< /highlight >}}

Sure enough, the variations of Alef are shown, also the Teh and the Heh, and the Tashkil is ignored (Notice the last name has Tashkil in it).

This was the first solution. We didn't touch the data itself here. All the change were made to the DBS itself. But what about if you can't access the character sets files (like when you are using a shared hosting for example), or when you are using SQLite and you don't have an option to add a new collation, the second solution will do the job in these cases.

<a name="sol2"></a>
2\. Add a normalized field
====

This solution doesn't require editing configuration files, and it is independent of the database system. It should work even if you changed the DBS for any reason. However, it will require adding an additional column to our table and some data processing. The idea is simple, add a new column and fill it with the Arabic text in a "normalized form", then use the normalized column in your queries. Let's see how we can do this.

I will use some PHP code in this example to add the normalized column to our table. You can make similar functions in any language once you get the idea. Consider this PHP function:

{{< highlight php >}}

function normalize_name($name) {
    $patterns     = array( "/إ|أ|آ/" ,"/ة/", "/َ|ً|ُ|ِ|ٍ|ٌ|ّ/" );
    $replacements = array( "ا" ,  "ه"      , ""         );
    return preg_replace($patterns, $replacements, $name);
}

{{< /highlight >}}

What this function does is pretty simple, it replaces all occurrences of `'ة'` with `'ه'`, all forms of Alef with Alef without Hamza`'ا'`, and removes the Tashkil. Let's see it in action:

{{< highlight php >}}

normalize_name("أحمد");  // return: احمد
normalize_name("آمنة");  // return: امنه
normalize_name("أسامه"); // return: اسامه
normalize_name("مٌحَمَّد");  // return: محمد

{{< /highlight >}}

OK, now we got our normalize function. The next step will be adding a new column to our table and filling it with the normalized name. A simple program should fill the column data, I will leave this to you to avoid adding unnecessary details. After that, we should have a table with the following data:

{{< highlight php >}}

1  احمد   احمد
2  أحمد   احمد
3  أسامه  اسامه
4  أسامة  اسامه
5  اسامه  اسامه
6  اسَامه  اسامه

{{< /highlight >}}

The second column in the previous layout is the normalized name field. Now we got our normalized data. How do we use it to solve our problem?

If the user searched for the name `"آسامة"`, we will pass this name to the normalize function first, which will return the normalized name `"اسامه"`, then we will query our `normazlized_name` column and display the original `name` column in our search results:

{{< highlight console >}}

SELECT id, name FROM persons WHERE normalized_name = "اسامه";

+----+--------------+
| id | name         |
+----+--------------+
|  3 | أسامه        |
|  4 | أسامة        |
|  5 | اسامه        |
|  6 | اسَامه        |
+----+--------------+
4 rows in set (0.00 sec)

{{< /highlight >}}

That's it. The search for `"آسامة"` resulted in all variations of the name.

So in short, we added the normalized field to the table, passed the search string to our normalize function, searched for the normalized name, and displayed the original name. This is a more modular solution but it requires more work. This was the second method.

<a name="sol3"></a>
3\. Use Regular Expressions in queries
====
In this solution, we won't change any configuration files, nor add extra data to our database. However, Regexp search is slower than regular search, and you will lose the advantage of using indices which could affect the performance badly. I don't know a way in regular expressions to ignore Tashkil in the database field. I don't recommend using Regular Expressions for search functions except in tiny databases, but you might find it useful for special kind of queries.

Regex isn't in the standard SQL, but most database systems will provide it with a different syntax. To apply this solution, we will replace our search string with a 'regexp' pattern.

Regular expressions in MySQL is done using `REGEXP` or its synonym `RLIKE`. You can browse [MySQL documentation for regexp](https://dev.mysql.com/doc/refman/5.7/en/regexp.html) to find out more about its syntax. To search for all occurrences of "اسامة", we will use the following pattern:

{{< highlight console >}}

"[ا|أ|إ|آ]سام[ه|ة]"

{{< /highlight >}}

This pattern simply means "Look for any form of Alef as the first character, and look for Teh or Heh at the end". Let's try it out on our example table:

{{< highlight console >}}

SELECT id, name FROM persons WHERE name REGEXP "[ا|أ|إ|آ]سام[ه|ة]";

+----+------------+
| id | name       |
+----+------------+
|  3 | أسامه      |
|  4 | أسامة      |
|  5 | اسامه      |
+----+------------+
3 rows in set (0.01 sec)

{{< /highlight >}}

Sure enough, all occurrences of `"اسامة"` are shown (except the one with the Tashkil as mentioned before). We will have to write a function to generate this pattern if we want to use this method in search fields. I will give you an example to do this in PHP, but this is just an example and most likely you will need a different approach:

{{< highlight php >}}

function generate_pattern($search_string) {
    $patterns     = array( "/(ا|إ|أ|آ)/", "/(ه|ة)/" );
    $replacements = array( "[ا|إ|أ|آ]", "[ه|ة]" );
    return preg_replace($patterns, $replacements, $search_string);
}

{{< /highlight >}}

This function will generate patterns like the following:

{{< highlight php >}}

generate_pattern("أسامة"); // return '[ا|إ|أ|آ]س[ا|إ|أ|آ]م[ة|ه]'
generate_pattern("أسامه"); // return '[ا|إ|أ|آ]س[ا|إ|أ|آ]م[ة|ه]'
generate_pattern("احمد");  // return '[ا|إ|أ|آ]حمد'

{{< /highlight >}}

Notice in the first and the second line, the function replaced the Alef in the middle of `'اسامة'` as well. You will need to strict it to only replace Alef at the beginning of the word, and replace Heh and Teh Marbota at the end of the word only. I will leave it to you to adjust the patterns.

Conclusion
====

That will be it for all the methods I know to accomplish Arabic insensitive search. Arabic is a great and a very rich language, this requires special handling in layouts, programming and databases, creating challenges we have to deal with as Arab programmers. Sharing our experiences is the key to create a strong and helpful community. And to add our contribution to the worldwide community of developers. Hope you found this article useful.
