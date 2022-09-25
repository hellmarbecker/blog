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
- Ingest this data set into Imply Polaris
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

## Ingesting the Data

[Here](https://docs.imply.io/polaris/quickstart/) you can find instructions how to sign up for a free trial of Imply Polaris. After following the signup process, you will have a fresh Polaris environment.

Let's create a table. Select `Tables` from the left navigation bar and start the table creation wizard using the `Create table` button. Name the new table `cohort` and make sure `Detail` is selected.

![Create table settings](/assets/2022-09-25-01-create-table.jpg)

(There are two kinds of tables in Polaris, `Detail` and `Aggregate`. Detail tables store the individual rows as they occur in the source data. Aggregate tables correspond to what we call _rollup_ in Druid, where data is aggregated to a predefined time granularity, greatly reducing storage needs. One of the cool things in Druid and Polaris is that you can [have that level of aggregation and still count unique things, like visitors](/2022/06/05/druid-data-cookbook-counting-unique-visitors-for-overlapping-segments/). But for now, we need the detail data.)

Select `Load data` to define your new table's schema based on the sample file. This will take you to the data sources dialog, where you select `Files` and `Upload files from your computer`. Select the `cohort.csv` file from the previous step, wait until the upload completes and proceed to the next step.

You will be presented with a sample from your data which should look similar to this:

![CSV parser settings](/assets/2022-09-25-02-parse-csv.jpg)

Verify that the input format is CSV and the `Data has header` switch is set to `Yes`. Continue to the next step:

![Schema definition](/assets/2022-09-25-03-schema.jpg)

Makes sure the primary timestamp (`__time`) is derived from the `curdate` column, and the `signupdate` field is modeled as a `string`. Then start the ingestion.






