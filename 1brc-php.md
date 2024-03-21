# Yet another implementation of 1BRC in PHP
## It's All Gone ThePrimeagen.
He was watching about 1BRC in GO.
A few weeks later, in the internal company's chat, I saw a link to an article on 1BRC in PHP, and people were discussing it.
Then, I thought, am I capable of doing it by myself? Am I good enough in PHP to do it?
The main issue was that PHP does not have built-in multi-thread functionality(`die("It's born to")`).
It can create a new process with `proc_open` but is a blocking operation.
So you can't actually compete with GO, for example.
Until PHP 8.0 where a Fiber class was introduced.

## The story...
Let's talk about the constraints of the task.
In general, they are the same as in the original 1brc [rules and limits](https://github.com/gunnarmorling/1brc?tab=readme-ov-file#rules-and-limits)
except for the fact that it should be done in PHP.
1. No 3rd party libraries
2. The code should be in a single file
3. the output is alphabetically sorted by the station name like this:
```
StationName1=MinTemp/MeanTemp/MaxTemp
StationName2=MinTemp/MeanTemp/MaxTemp
```
### The Main Idea
was to repeat GO article path starting from naive implementation and then iteratively improve it.
Thee first try was to read the file line by line and then process it.
```php
declare(strict_types=1);
$filename = $_SERVER['argv'][1] ?? 'measurements-1000M.txt';
$fd = fopen($filename, "rb");
$result = [];
while ($line = fgetcsv($fd, null, ";")) {
    $name = $line[0];
    $temp = (float)$line[1];
    if (isset($result[$name])) {
        [$min, $max, $sum, $count] = $result[$name];
        $result[$name] = [min($min, $temp), max($max, $temp), $sum + $temp, $count + 1];
    } else {
        $result[$name] = [$temp, $temp, $temp, 1];
    }
}
fclose($fd);
ksort($result);
foreach ($result as $name => $station) {
    [$min, $max, $sum, $count] = $station;
    $mean = $sum / $count;
    printf("%s=%01.2f/%01.2f/%01.2f\n", $name, $min, $mean, $max);
}
```

Looks good. Let's test it.
```fish
time php 1.php
```
```
________________________________________________________
Executed in  400.25 secs    fish           external
   usr time  393.70 secs  114.00 micros  393.70 secs
   sys time    2.97 secs  614.00 micros    2.97 secs

```
Ok, ~400 seconds. Who expected less? This is PHP, you would say.
But this is the baseline. Let's try to improve it.
Following the thoughts of the author of GO article, we need to reduce some checks that are not required.
For example, we know the file format is a string, then a semicolon, then a number, and a line ends.
That means that we actually don't need to use `fgetcsv` and can use `fgets` instead, which does not check for correct CSV format where
```csv
Name with spaces;12.3
```
and
```csv
Name 
with spaces;12.3
```
has the same output as a row with 2 columns in both cases.
```php
while ($line = fgets($fd)) {
    $lines = explode(";", $line);
```
The result is
```
________________________________________________________
Executed in  275.55 secs    fish           external
   usr time  269.89 secs  121.00 micros  269.89 secs
   sys time    2.77 secs  693.00 micros    2.77 secs

```
It is almost 1.6 times faster. Good start, but we can do better. As you can see, we use `min/max` functions on every line. That might cost us some time.
Let's try to remove it and use `if/else` or ternary operator
```php
$result[$name] = [$min < $temp ? $min : $temp, $max > $temp ? $max : $temp, $sum + $temp, $count + 1];
```
```
________________________________________________________
Executed in  264.19 secs    fish           external
   usr time  258.70 secs  114.00 micros  258.70 secs
   sys time    2.76 secs  720.00 micros    2.76 secs

```
That's a surprise. It gives us an improvement of 10 seconds. But what if we store all temperatures in an array and then use `min/max` functions on the final output?
```php
    if (isset($result[$name])) {
        $result[$name][0][] = $temp;
        $result[$name][1] += $temp;
        $result[$name][2] += 1;
    } else {
        $result[$name] = [[$temp], $temp, 1];
    }
```
It gives me a `PHP Fatal error:  Allowed memory size of...` error. Ok, for the moment, I was using a default memory limit of 128M.
Let's remove this limitation. Who cares about memory?
```fish 
time php -dmemory_limit=-1 4.php
```
```
________________________________________________________
Executed in  228.39 secs    fish           external
   usr time  211.87 secs    0.12 millis  211.87 secs
   sys time   11.70 secs    2.85 millis   11.70 secs

```
This actually gives a 30-seconds improvement. The explanation for this is the fact that the actual calculation of min and max is delegated to the C level and is (blazingly) faster than in PHP.

So far, so good, but at this point, I've seen a message in the internal chat that a college of mine could do it in 17 seconds for a single thread and ~7 seconds for a multi-threaded version(10 threads).
Ok, 7 seconds, even for multi-thread, looks fast for PHP, but it was tested on a file with 100M lines. So relax; PHP is fast nowadays but blazingly fast.
For 100M rows, my version would take 22.8 seconds, meaning it depends on the number of lines linearly(more or less)
```fish
time php -dmemory_limit=-1 4.php measurements-100M.txt 
________________________________________________________
Executed in   21.53 secs    fish           external
   usr time   20.67 secs  126.00 micros   20.67 secs
   sys time    0.59 secs  679.00 micros    0.59 secs
```
Long story short, I couldn't achieve a single-threaded version of my implementation to be faster than 22 seconds for 100M rows.

### Let's introduce multi-threading
As I metioned before, PHP does not have (had?) multi-threading capabilities.
But it does have a way to create a new process with `proc_open` and communicate with it.
Also, there is a `stream_set_blocking` function which
```
Sets blocking or non-blocking mode on a stream.
This function works for any stream that supports non-blocking mode (currently, regular files and socket streams).
```
This is what exactly I need.
To use it, I need to modify the script a bit.
My plan was the following:
1. Read the file in chunks and pass each chunk to a thread
2. set non-blocking mode for a thread
3. process chunk in the new thread and output the result in CSV format as `name;min;max;sum;count`
4. read the output of the processed chunk and merge it with the main result
5. sort the result by station name and output it
6. profit

The code is provided in the [5.php](5.php) file.
The result is...32 seconds
```fish
________________________________________________________
Executed in   30.00 secs    fish           external
   usr time  224.64 secs    0.15 millis  224.64 secs
   sys time   25.61 secs    1.22 millis   25.61 secs
```
That is almost 10 times faster than it was before!. I couldn't believe it. Initially, I tested on a 100M rows file, which gave me less than 4 seconds.
My first thoughts were:
- it is too good to be true
- most probably somewhere, I have a bug, so the subprocess doesn't run till the end and exits earlier
- how can I prove that I have an issue or not?

The following steps were to write a naive single-thread script and compare the output with the multi-threaded one.
After all, I did it, and the results were identical. That means that the multi-threaded version is correct. I was as happy as every programmer when I completed the task and could not believe it worked.

But I noticed one thing: the multi-threaded version uses all CPU cores and produces a log of processes.
That's not good. I want to control the number of processes and CPU cores used.

### Use Fiber class to limit CPU core usage
In the article [PHP Fibers: A practical example](https://aoeex.com/phile/php-fibers-a-practical-example/) provided in the internal chat, the author describes how to use the Fiber class to speed up the video file conversion process.
The main idea is to create a pool of fibers and then execute them in parallel.
The file [6.php](6.php) contains the idea's implementation and mostly adapts the article to my script.
Now I can run the script and specify the number of fibers/threads I want to use
```fish
> time php -dmemory_limit=-1 6.php measurements-1000M.txt 10
________________________________________________________
Executed in   31.41 secs    fish           external
   usr time  236.93 secs  160.00 micros  236.93 secs
   sys time   26.93 secs  685.00 micros   26.93 secs
```

### At the end
I am proud that I could implement the 1BRC in PHP and make it faster than expected.
I've learned a lot about PHP and its capabilities.
I think there are places to improve the script. I want to check other guys' implementations to see why my single-threaded version is slower.
Maybe if I improve it, I can make the multi-threaded version even faster.
I noticed that increasing the number of threads does not give big improvements, so this is also a place to look for improvements.

Sincerely yours,
Eksandral


PS:
I used my local PC for testing purposes.
Yeah-yeah, I know, it's a benchmark on the local machine, again, but deal with it.
The article's purpose is to show that even though PHP is born to die; it dies blazingly fast.
```fish
Software:
    System Software Overview:
      System Version: macOS 14.4
      Kernel Version: Darwin 23.4.0
      Boot Volume: Macintosh HD
      Boot Mode: Normal
      Secure Virtual Memory: Enabled
      System Integrity Protection: Enabled
Hardware:
    Hardware Overview:
      Model Name: MacBook Pro
      Model Identifier: Mac14,10
      Chip: Apple M2 Pro
      Total Number of Cores: 12 (8 performance and 4 efficiency)
      Memory: 32 GB
```

```fish
> php -v
PHP 8.3.2 (cli) (built: Jan 26 2024 23:59:55) (NTS)
Copyright (c) The PHP Group
Zend Engine v4.3.2, Copyright (c) Zend Technologies
    with Zend OPcache v8.3.2, Copyright (c), by Zend Technologies

```
