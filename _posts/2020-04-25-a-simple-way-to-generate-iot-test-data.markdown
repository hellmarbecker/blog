---
layout: post
title:  "A simple way to generate IoT test data"
categories: blog perl iot dataflow nifi
---
A while ago, I was called to do a presentation at a large manufacturer of wind turbines. You might have
a couple of those installed in the fields around your town if you live in the German countryside,
but the more lucrative way of deploying turbines is in offshore windparks.

My idea was to demo a solution approach based on Apache Nifi, which is a great tool for live data flow
management. My only problem was that I don't have a wind turbine in my garden, so I didn't have a test 
data set.

So I took a simplistic approach: I scoured a bunch of web sites and literature that is available online,
in order to learn what kind of sensor data are typically collected around a wind turbine. This was some 
interesting learning.

Engineers do, of course, measure the wind speed and direction, the rotation frequency of the blades, and the
total electrical power that is generated. But they also measure speed and acceleration of the blade, total
movement of the tower, and mechanical strain on various parts of the installation.

So, I picked a set of sensor values and wrote a script to populate these with random data. Since this 
is a one-shot project, I decided for a quick and dirty solution in Perl 5.

The plan was to generate a row of data roughly every second, each row representing one particular sensor 
value for one turbine in a wind park that has 100 turbines. So, the format was going to be:

    turbineID|timestamp|signalName|signalValue

and in order to generate plausible random values, I would need a map of all the sensor names and the 
possible ranges of values.

Here's my code:

``` perl
#!/usr/bin/env perl

use strict;
use warnings;
use Time::HiRes qw( time usleep );

# TurbineID, Timestamp, SignalName, SignalValue

my $NumTurbines = 100;
my %SignalRange = (
    wind_speed          => [ 0.0, 35.0 ],   # m/sec
    power               => [ 0.0, 9.0 ],    # MW
    rpm_rotor           => [ 0.0, 20.0 ],   # 1/min
    rpm_generator       => [ 1400.0, 1800.0 ],   # 1/min
    torque_generator    => [ 0.0, 50.0 ],   # kN.m
    angle_blade         => [ 0.0, 90.0 ],   # deg
    v_angle_blade       => [ 0.0, 10.0 ],   # deg/sec            
    wind_direction      => [ -180.0, 180.0 ], # deg
    a_blade             => [ 0.0, 10.0 ],   # m/(sec^2)
    a_tower             => [ 0.0, 10.0 ],   # m/(sec^2)
    strain_blade_root   => [ 0.0, 0.1 ],    # mm
    strain_shaft        => [ 0.0, 0.1 ],    # mm
    strain_tower        => [ 0.0, 0.1 ]     # mm
);
my @SignalName = sort keys %SignalRange;

sub rrand {
    my( $min, $max ) = @{$_[0]};
    rand( $max - $min ) + $min;
}

while ( 1 ) {
    my $turbineID = int( rand( $NumTurbines ) );
    my $timestamp = time;
    my $signalName = $SignalName[ rand( $#SignalName ) ];
    my $signalValue = rrand( $SignalRange{ $signalName } );

    my $outstr = sprintf( "%d,%.3f,%s,%.2f\n", $turbineID, $timestamp, $signalName, $signalValue );
    print $outstr;
    usleep 990;
}
```

This code generates the desired values and writes them to standard output. From there, I can pick them up using 
any downstream process I want. If my downstream process exposes a service port, I can just pipe my
output into `nc` in order to hand it over.

That's what I am going to do with these data, using Nifi. I will elaborate on that in another post.

