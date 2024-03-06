---
title: 'How to load 100,000,000 rows into MySQL (fast)'
description: 'Mysql load queries'
pubDate: 'Mar 4 2024'
heroImage: '/blog-placeholder-1.jpg'
---
## Introduction
I need to run some queries on a 100 million rows from a 200 GB data set there's just one problem this data is setting in a text file, I need to get this loaded from that text file into MySQL and I want to do this fast I do not want to be waiting around all day so if you want to see my process for how i go from text file pre-processing and loading into MySQL stay right here sneak preview it's going to be a little tricky.

I was browsing the internet for some data i could use to analyze and i came across this database [lichess.org](https://database.lichess.org/) this website has over 2 billion games worth of historic data, pretty cool. But I'm not going to download all of that. In fact I tried to download all of that in th format that they give it would more that fill my whole hard drive. 

### Step 1
Im going to download one month worth of the data which still has over a 100 million games in it and then I can get this loaded into MySQL. But before we have to decompress the zst file, this process is going to take several time.

```console
sirlordluis@mipc:~$ wget https://database.lichess.org/standard/lichess_db_standard_rated_2024-02.pgn.zst
sirlordluis@mipc:~$ mv lichess_db_standard_rated_2024-02.pgn.zst lichess.pgn.zst
sirlordluis@mipc:~$ zstd -dc -r lichess.pgn.zst -o linchess.pgn

```

If you look at the file and i can take a look and see that it exist and i can also check out the size and it's 28G but if you take a look there's another file with extencion pgn, this file contains all the data we want to insert into MySQL .

```console
sirlordluis@mipc:~$ du -h *
28G     lichess.pgn.zst
199G    linchess.pgn

```

### Step 2

If I shows you how this looks, the format consists of several lines for each game, containing information such as the date, players, the type of game, and even a string representing the sequence of moves

```console
sirlordluis@mipc:~$ bat linchess.pgn
   1   │ [Event "Rated Blitz game"]
   2   │ [Site "https://lichess.org/1ZVhN0qE"]
   3   │ [Date "2024.02.01"]
   4   │ [Round "-"]
   5   │ [White "FabianGandur"]
   6   │ [Black "faustilini"]
   7   │ [Result "0-1"]
   8   │ [UTCDate "2024.02.01"]
   9   │ [UTCTime "00:00:12"]
  10   │ [WhiteElo "1329"]
  11   │ [BlackElo "1353"]
  12   │ [WhiteRatingDiff "-5"]
  13   │ [BlackRatingDiff "+7"]
  14   │ [ECO "C50"]
  15   │ [Opening "Italian Game: Giuoco Pianissimo"]
  16   │ [TimeControl "300+0"]
  17   │ [Termination "Normal"]
```
Now we have to convert the PGN file into a CSV file. In Python, there is a module called pgn2data, but Python is really slow. We definitely need to find something that is more performant to get this done efficiently. 

I wrote a program in C to make this process faster. You can find the program in my GitHub repository [link](https://github.com/sirlordluis/100milliontorow/blob/main/convert_pgn_to_csv.c).

```console
sirlordluis@mipc:~$ gcc -O3 convert_pgn_to_csv.c -o convert_pgn_to_csv
sirlordluis@mipc:~$ ./convert_pgn_to_csv linchess.pgn
sirlordluis@mipc:~$ du -h *
28G     lichess.pgn.zst
199G    linchess.pgn
11G     partidas.csv
20K     convert_pgn_to_csv
4.0K    convert_pgn_to_csv.c
```
If you look at the file partidas.csv, it now contains the information from linchess.pgn. I can check whit this command, almost we have 100 million of rows.

```console
sirlordluis@mipc:~$ time wc -l partidas.csv
91628936 partidas.csv
wc -l partidas.csv  0.48s user 5.88s system 4% cpu 2:08.12 total
```
### Step 3

Now we have to create a database, and the next table. 

``` sql
CREATE TABLE chessgame(
`id` bigint(20) NOT NULL AUTO_INCREMENT,
`game_type` varchar(256),
`url` varchar(256),
`white` varchar(32),
`black` varchar(32),
`white_elo` int(11),
`black_elo` int(11),
`result` varchar(10),
 PRIMARY KEY (`id`))
```
Normally, for bulk data, we need to generate the inserts. I've created a Python script to transform CSV rows into inserts. You can find the script in my GitHub repository [link](https://github.com/sirlordluis/100milliontorow/blob/main/convert_csv_to_inserts.py).

```console
sirlordluis@mipc:~$ python3 convert_csv_to_inserts.py partidas.csv > salida.sql
```
To execute all these queries,normally we use the following command, but this process is too slow.

```console
sirlordluis@mipc:~$ time mysql -u root database_name < salida.sql
mysql -u root database_name < salida.sql  0.00s user 0.01s system 0% cpu 7:04.70 total
```
Some databases, such as MySQL, provide the **'LOAD DATA'** statement, which is much more efficient for bulk data loading than traditional insert statements.

``` sql
LOAD DATA LOCAL INFILE 'partidas.csv' INTO TABLE chessgame
    FIELDS TERMINATED BY ','
    OPTIONALLY ENCLOSED BY '"'
    LINES TERMINATED BY '\n'
    (@col1, @col2, @none, @col4, @col5, @col6, @col7, @col8, @none)
    SET
    game_type=@col1,
    url=@col2,
    white=@col4,
    black=@col5,
    white_elo=@col7,
    black_elo=@col8,
    result=@col6;
```
Running the script.
``` console
sirlordluis@mipc:~$ time mysql --local-infile -u root database_name < increase_speed.sql
mysql --local-infile -u root database_name < increase_speed.sql  0.00s user 0.01s system 0% cpu 7:04.70 total
```
Look at the time, this is a big improvement over the previous one. The previous one took over 7 hours, whereas this one only took 7 minutes."