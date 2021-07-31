---
layout: post
title:  "Generating IBAN numbers for simulated financial transactions"
categories: blog perl testing financial realtime graph
---
Financial transactions present a whole lot of interesting technology use cases.

  - For instance, fraud detection would apply rule based algorithms or, increasingly,
    machine learning to identify possibly fraudulent transactions by patterns in the total transaction
    graph.
  - Or you might identify higher net worth account holders by looking not only at their account balance,
    but also their total transaction patterns, so as to offer premium services to these customers,
    increasing both their satisfaction and the revenue they generate.

This offers great use cases for modern data processing technologies, such as NoSQL/graph
databases, multi model databases, and real time event processing.

All these use cases need to be tested, and often, demoed. But for obvious reasons,
it is usually not possible to access real transaction data, so simulated data needs 
to be generated.

Here is a small script that I use to generate an algorithmically correct IBAN, which 
is the unique account identifier we use in Europe. This is another quick Perl 5 hack
but it does the job.

It uses the German numbering system of so called Bankleitzahl (bank ID), and account number, 
both numerical. In the Netherlands, the bank identifier would be alphabetical, but my 
algorithm could be used with a minor format change.

Here's the script:

``` perl
#!/usr/bin/env perl

use strict;
use warnings;

#---------------------------------------
# makeIBAN
#---------------------------------------
# see https://www.ibantest.com/en/how-is-the-iban-check-digit-calculated
sub makeIBAN {
    my ( $blz, $acct ) = @_;

    # BLZ is 8 digits
    $blz =~ /^\d{8}$/ or die "BLZ has to be 8 digits";
    $acct =~ /^\d{1,10}$/ or die "Account number has to be numeric";

    # 1. Concatenate BLZ and account number into BBAN
    my $bban = sprintf( "%08d%010d", $blz, $acct );

    # 2. Create IBAN without check digit
    my $prefix = "DE00";
    my @prefix = split( //, $prefix );

    # 3. Translate letters into decimal numbers, starting at 10 for 'A'.
    # Append this at the end of BBAN
    my $sum = $bban;
    for (@prefix) {
        if (/[A-Z]/) {
	    $sum .= 10 + ord( $_ ) - ord( 'A' );
        } else {
            $sum .= $_;
        }
    }
 
    # 4. Take result mod 97, subtract from 98
    my $chksum = sprintf( "%02d", 98 - $sum % 97 );
    $prefix =~ s/00/$chksum/;

    return $prefix . $bban;
    
} # makeIBAN


my $iban = makeIBAN( "32050000", "12345600" );
print $iban;
```
