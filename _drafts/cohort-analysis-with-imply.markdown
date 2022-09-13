---
layout: post
title:  "Tutorial: Cohort Analysis with Imply"
categories: blog druid imply pivot analytics tutorial gaming
---

Imagine you are running an online gaming platform where players deposit funds into their accounts and then use these funds for games. In order to analyze  player retention and lifetime value, you want to group players according to when they signed up (which calendar month, or week), and then for each group chart their spending behaviour over time, starting at the time when they signed up.

This is the use case for [_cohort analysis_](https://en.wikipedia.org/wiki/Cohort_analysis).

Cohort analysis is also used to analyze customer loyalty in ecommerce, and there are numerous other application cases in various industries. With Imply, it is easy to do even on large data sets.

Let's look at an example!

In this tutorial, you will

- Generate a random data set that is suitable for cohort analysis
- Ingest this data set into Imply Druid using the new SQL based ingestion
- Create a suitable logical data model in Imply Pivot, on top of the Druid table
- Create a cohort based visualisation.

## Simulating Data

Let's create a simple data set. The following script creates a random set of players with signup dates in the last 2 years, and then generates events, trying to mimic typical player behaviour by creating less events for players that have been with the platform for longer time.

Each event row describes a deposit made, and includes the event date, the player ID, the player's signup date, and the amount of the deposit.

Run the Perl script below and pipe the output into a file named `cohort.csv`:

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

Upload this file into a S3 bucket.

## Ingesting the Data

```sql
REPLACE INTO cohort OVERWRITE ALL
WITH source AS (SELECT * FROM TABLE(
  EXTERN(
    '{"type":"s3","uris":["s3://<your S3 file path>"]}',
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
