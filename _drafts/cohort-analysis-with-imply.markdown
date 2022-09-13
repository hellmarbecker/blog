---
layout: post
title:  "Cohort Analysis with Imply"
categories: blog druid imply pivot analytics tutorial gaming
---

## Simulating Data

```perl
#!/usr/bin/perl
# create data set for cohort analysis

use strict;
use warnings;

use Getopt::Std;
use DateTime;

my $start = DateTime->new(
    day   => 1,
    month => 1,
    year  => 2022,
);

my $stop = DateTime->new(
    day   => 1,
    month => 9,
    year  => 2022,
);

my $numPlayers = 100000;
my $playerMaxAge = 730;
my $minEventsPerDay = 300;
my $maxEventsPerDay = 1000;
my $maxDeposit = 50;

my @players;
my @playerSignup;

my $today = DateTime->today;

# initialize players and signup dates

for my $i (0 .. $numPlayers-1 ) {
    my $playerID = "p" . sprintf("%05d", $i);
    $players[$i] = $playerID;
    my $playerAgeNow = int(rand($playerMaxAge));
    my $signupDate = $today - DateTime::Duration->new(days => $playerAgeNow);
    $playerSignup[$i] = $signupDate;
}

print "curdate,playerid,signupdate,deposit\n";

for ( my $d = $start->clone; $d < $stop; $d->add(days => 1) ) {
    my $n = 0;
    my $numEventsMax = $minEventsPerDay + int(rand($maxEventsPerDay - $minEventsPerDay));
    while ($n < $numEventsMax) {
        my $ind = int(rand($numPlayers));
        my $playerID = $players[$ind];
        next if ( $playerSignup[$ind] > $d ); # player did not exist yet

        my $playerAge = $d->delta_days($playerSignup[$ind])->in_units('days');
        my $cooldown = exp(-3.0 * $playerAge / $playerMaxAge);
        next if ( rand(1.0) > $cooldown ); # simulate that players churn off slowly

        my $deposit = sprintf("%.2f", rand($maxDeposit));
        print "$d,$playerID,$playerSignup[$ind],$deposit\n";
        ++$n;
    }
}
```

## Ingesting the Data

```sql
REPLACE INTO cohort OVERWRITE ALL
WITH source AS (SELECT * FROM TABLE(
  EXTERN(
    '{"type":"s3","uris":["s3://imply-cloud-sales-data/valsea/cohort.csv"]}',
    '{"type":"csv","findColumnsFromHeader":true}',
    '[{"name":"curdate","type":"string"},{"name":"playerid","type":"string"},{"name":"signupdate","type":"string"},{"name":"deposit","type":"double"},{"name":"signupdate_long","type":"long"}]'
  )
))
SELECT
  TIME_PARSE(curdate) AS __time,
  playerid,
  signupdate,
  deposit
FROM source
PARTITIONED BY MONTH
CLUSTERED BY playerid
```
