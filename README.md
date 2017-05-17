# Axibase Time Series Database Client for Perl

## Overview

The project contains sample code to parse tabular files and insert data into [Axibase Time Series Database](https://axibase.com/products/axibase-time-series-database/). The examples will be used in the future to implement a full-featured API client for Perl.

## Examples

### Sending Data using Network API

This examples illustrates how to read a CSV file, split its contents line by line, build a [series](https://github.com/axibase/atsd/blob/master/api/network/series.md#series-command) command and send it into ATSD over TCP.

The data file contains the latest values for a set of tests which are timestamped at the time the scipt is invoked.

* Data File

```ls
node,test_name,test_status,test_duration
axi-01,api-q-1,0,32
axi-01,api-q-2,0,2050
axi-01,api-q-4,0,120
```

* Perl Script

```perl
#!/usr/bin/perl
use strict;
use warnings FATAL => 'all';
use IO::Socket::INET;

my $num_args = $#ARGV + 1;
if ($num_args != 3) {
    print "\nUsage: series-uploader.pl host port filename\n";
    exit;
}

my $filename = $ARGV[2];
open(my $fh, '<:encoding(UTF-8)', $filename)
    or die "Could not open file '$filename' $!";

# converting file into series commands
my $separator = ',';
my $unix_seconds = time();
my $lineindex = 0;
my $commands = "";

while (my $line = <$fh>) {
    #skip header
    if ($lineindex == 0) {
        $lineindex++;
        next;
    }
    chomp $line;
    my @row = split($separator, $line);

    $commands = $commands . sprintf("series e:%s m:test_duration=%s t:test_name=%s t:test_status=%s s:%s\n",
        $row[0], $row[3], $row[1], $row[2], $unix_seconds);
    $lineindex++;
}

# data to send to a server
# create a connecting socket
my $socket = new IO::Socket::INET (
    PeerHost => $ARGV[0],
    PeerPort => $ARGV[1],
    Proto => 'tcp',
);
die "cannot connect to the server $ARGV[0]:$ARGV[1] $!\n" unless $socket;

$socket->send($commands);

# notify server that request has been sent
shutdown($socket, 1);

$socket->close();
```

* Usage

```bash
perl series-uploader.pl localhost 8081 data.csv
```

* Commands Sent

```ls
series e:axi-01 m:test_duration=32 t:test_name=api-q-1 t:test_status=0 s:1495033767
series e:axi-01 m:test_duration=2050 t:test_name=api-q-2 t:test_status=0 s:1495033767
series e:axi-01 m:test_duration=120 t:test_name=api-q-4 t:test_status=0 s:1495033767
```


### Sending Data using Network API (Data File with History)

This example is similar to the initial example except that each row in the data file contains a timestamp in the local time zone.

```ls
date,node,test_name,test_status,test_duration
2017-05-15 22:00:00,axi-01,api-q-1,0,32
2017-05-15 22:00:00,axi-01,api-q-2,0,2050
2017-05-15 22:00:00,axi-01,api-q-4,0,120
2017-05-15 22:00:00,axi-01,api-q-1,0,32
2017-05-15 22:00:00,axi-01,api-q-2,0,2050
2017-05-15 22:00:00,axi-01,api-q-4,0,120
```

* Perl Script

```perl
#!/usr/bin/perl
use strict;
use warnings FATAL => 'all';
use Date::Parse;
use Time::Piece;
use IO::Socket::INET;

my $num_args = $#ARGV + 1;
if ($num_args != 3) {
    print "\nUsage: series-uploader.pl host port filename\n";
    exit;
}

my $filename = $ARGV[2];
open(my $fh, '<:encoding(UTF-8)', $filename)
    or die "Could not open file '$filename' $!";

# converting file into series commands
my $separator = ',';
my $lineindex = 0;
my $commands = "";

while (my $line = <$fh>) {
    #skip header
    if ($lineindex == 0) {
        $lineindex++;
        next;
    }
    chomp $line;
    my @row = split($separator, $line);

    my $seriesdate = gmtime(str2time($row[0]))->strftime('%Y-%m-%dT%H:%M:%SZ');

    $commands = $commands . sprintf("series e:%s m:test_duration=%s t:test_name=%s t:test_status=%s d:%s\n",
        $row[1], $row[4], $row[2], $row[3], $seriesdate);
    $lineindex++;
}

# data to send to a server
# create a connecting socket
my $socket = new IO::Socket::INET (
    PeerHost => $ARGV[0],
    PeerPort => $ARGV[1],
    Proto => 'tcp',
);
die "cannot connect to the server $ARGV[0]:$ARGV[1] $!\n" unless $socket;

$socket->send($commands);

# notify server that request has been sent
shutdown($socket, 1);

$socket->close();
```

* Usage

```bash
perl series-uploader.pl localhost 8081 data.csv
```

* Commands Sent

```ls
series e:axi-01 m:test_duration=32 t:test_name=api-q-1 t:test_status=0 d:2017-05-15T19:00:00Z
series e:axi-01 m:test_duration=2050 t:test_name=api-q-2 t:test_status=0 d:2017-05-15T19:00:00Z
series e:axi-01 m:test_duration=120 t:test_name=api-q-4 t:test_status=0 d:2017-05-15T19:00:00Z
series e:axi-01 m:test_duration=32 t:test_name=api-q-1 t:test_status=0 d:2017-05-15T19:00:00Z
series e:axi-01 m:test_duration=2050 t:test_name=api-q-2 t:test_status=0 d:2017-05-15T19:00:00Z
series e:axi-01 m:test_duration=120 t:test_name=api-q-4 t:test_status=0 d:2017-05-15T19:00:00Z
```

### Sending Data using Data API

This example is similar to the initial example and illustrates how to read a CSV file, split its contents line by line, build a batch of [series](https://github.com/axibase/atsd/blob/master/api/network/series.md#series-command) commands and upload them into ATSD over HTTP(s) using the Data API [command](https://github.com/axibase/atsd/blob/master/api/data/ext/command.md) method.

* Perl Script

```perl
#!/usr/bin/perl
use strict;
use warnings FATAL => 'all';
use Date::Parse;
use Time::Piece;
require LWP::UserAgent;
require IO::Socket::SSL;

my $num_args = $#ARGV + 1;
if ($num_args != 2) {
    print "\nUsage: https-series-uploader.pl url filename\n";
    exit;
}

my $filename = $ARGV[1];
open(my $fh, '<:encoding(UTF-8)', $filename)
    or die "Could not open file '$filename' $!";

# converting file into series commands
my $separator = ',';
my $lineindex = 0;
my $commands = "";

while (my $line = <$fh>) {
    #skip header
    if ($lineindex == 0) {
        $lineindex++;
        next;
    }
    chomp $line;
    my @row = split($separator, $line);

    my $seriesdate = gmtime(str2time($row[0]))->strftime('%Y-%m-%dT%H:%M:%SZ');

    $commands = $commands . sprintf("series e:%s m:test_duration=%s t:test_name=%s t:test_status=%s d:%s\n",
        $row[1], $row[4], $row[2], $row[3], $seriesdate);
    $lineindex++;
}

# disable certificate verification
$ENV{'PERL_LWP_SSL_VERIFY_HOSTNAME'} = 0;
IO::Socket::SSL::set_ctx_defaults(
    SSL_verifycn_scheme => 'www',
    SSL_verify_mode => 0,
);

# send commands
my $ua = LWP::UserAgent->new;
$ua->timeout(10); # 10 seconds
$ua->env_proxy;

my $request = HTTP::Request->new(POST=>$ARGV[0] . '/api/v1/command');
$request->content_type('text/plain');
$request->content($commands);

my $response = $ua->request($request);

if (!$response->is_success) {
    die $response->status_line;
}

if ($response->code != 200) {
    die "Server returned error: " . $response->code;
}

if ($response->header('Content-Type') !~ 'application/json') {
    die "Unexpected content type: " . $response->header('Content-Type');
}

print $response->decoded_content;
```

* Usage

```bash
perl https-series-uploader.pl https://user:password@localhost:8443 data.csv
```

