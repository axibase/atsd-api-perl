# ATSD Perl Client

![](./images/axibase-and-perl.png)

## Table of Contents

* [Overview](#overview)
* [Examples](#examples)

## Overview

**ATSD Perl Client** enables Perl developers to parse tabular files into [Axibase Time Series Database](https://axibase.com/docs/atsd/) series commands, which can be inserted into a target ATSD instance via `TCP` and `HTTPS` protocols.

## Examples

### Prerequisites

Some examples use Perl module [`DateTime::Format::Strptime`](https://metacpan.org/pod/DateTime::Format::Strptime) to parse local timestamps.

If a script contains the `use DateTime::Format::Strptime;`  declaration, install the module as follows:

```sh
sudo apt-get install cpanminus
# sudo yum install cpanminus
sudo cpanm DateTime::Format::Strptime
```

### Sending Data via [Network API](https://axibase.com/docs/atsd/api/network/)

This example illustrates how to read a CSV file, split the contents line by line, build a [series command](https://axibase.com/docs/atsd/api/network/series.html) and send it into the target ATSD server via `TCP`.

The data file contains the latest values for a set of tests which are timestamped with the time of script invocation.

* Data File

```ls
node,test_name,test_status,test_duration
axi-01,api-q-1,0,32
axi-01,api-q-2,0,2050
axi-01,api-q-4,0,120
```

* Model

| CSV Column | Command Field | Sample Value |
|---|---|---:|
| `node` | Entity | `axi-01` |
| `test_name` | Tag | `api-q-1` |
| `test_status` | Metric | `0` |
| `test_duration` | Metric | `32` |
| - | Time (seconds) | `1495198062` |

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
# all samples will be timestamped with the same time in UNIX seconds
my $current_time_unix_seconds = time();
my $line_index = 0;
# an array of commands, separated by line feed
my $commands = "";

while (my $line = <$fh>) {
    #skip header
    if ($line_index== 0) {
        $line_index++;
        next;
    }
    chomp $line;
    # split the row into multiple tokens using comma as the separator
    my @row = split($separator, $line);

    $commands = $commands . sprintf("series e:%s t:test_name=%s m:test_status=%s m:test_duration=%s s:%s\n",
        $row[0], $row[1], $row[2], $row[3], $current_time_unix_seconds);
    $line_index++;
}

print $commands;

# send commands to the ATSD server
# create a connecting socket
my $socket = new IO::Socket::INET (
    PeerHost => $ARGV[0],
    PeerPort => $ARGV[1],
    Proto => 'tcp',
);
die "cannot connect to the ATSD server $ARGV[0]:$ARGV[1] $!\n" unless $socket;

$socket->send($commands);

# flush the buffer and close the socket gracefully
shutdown($socket, 1);

$socket->close();
```

* Usage

```bash
perl series-uploader.pl localhost 8081 data.csv
```

* Commands Sent

```ls
series e:axi-01 t:test_name=api-q-1 m:test_status=0 m:test_duration=32 s:1495198062
series e:axi-01 t:test_name=api-q-2 m:test_status=0 m:test_duration=2050 s:1495198062
series e:axi-01 t:test_name=api-q-4 m:test_status=0 m:test_duration=120 s:1495198062
```

### Sending Data via [Network API](https://axibase.com/docs/atsd/api/network/) (Data File with History)

Consider the above example with the additional component of each row in the data file containing a timestamp in the local time zone.

```ls
date,node,test_name,test_status,test_duration
2017-05-15 22:00:00,axi-01,api-q-1,0,32
2017-05-15 22:00:00,axi-01,api-q-2,0,2050
2017-05-15 22:00:00,axi-01,api-q-4,0,120
2017-05-15 23:00:00,axi-01,api-q-1,0,28
2017-05-15 23:00:00,axi-01,api-q-2,0,4000
2017-05-15 23:00:00,axi-01,api-q-4,0,201
```

* Model

| CSV Column | Command Field | Sample Value |
|---|---|---:|
| `node` | Entity | `axi-01` |
| `test_name` | Tag | `api-q-1` |
| `test_status` | Metric | `0` |
| `test_duration` | Metric | `32` |
| `date` | Time ([ISO 8601](https://axibase.com/docs/atsd/shared/date-format.html)) | `2017-05-16T05:00:00Z` |

* Perl Script

```perl
#!/usr/bin/perl
use strict;
use warnings FATAL => 'all';
use IO::Socket::INET;
# make sure the module is installed
use DateTime::Format::Strptime;

my $num_args = $#ARGV + 1;
if ($num_args != 3) {
    print "\nUsage: series-date-uploader.pl host port filename\n";
    exit;
}

my $filename = $ARGV[2];
open(my $fh, '<:encoding(UTF-8)', $filename)
    or die "Could not open file '$filename' $!";

# converting file into series commands
my $separator = ',';
# the parser will parser local dates contained in the CSV file in US/Pacific timezone
my $date_parser = new DateTime::Format::Strptime(
        pattern => '%Y-%m-%d %H:%M:%S',
        time_zone => 'US/Pacific',
    );
my $line_index = 0;
my $commands = "";

while (my $line = <$fh>) {
    #skip header
    if ($line_index == 0) {
        $line_index++;
        next;
    }
    chomp $line;
    my @row = split($separator, $line);

    my $series_date =  $date_parser->parse_datetime($row[0]);
    $series_date->set_time_zone('GMT');

    $commands = $commands . sprintf("series e:%s t:test_name=%s m:test_status=%s m:test_duration=%s d:%s\n",
        $row[1], $row[2], $row[3], $row[4], $series_date->strftime("%Y-%m-%dT%H:%M:%SZ"));
    $line_index++;
}

print $commands;

# data to send to a server
# create a connecting socket
my $socket = new IO::Socket::INET (
    PeerHost => $ARGV[0],
    PeerPort => $ARGV[1],
    Proto => 'tcp',
);
die "cannot connect to the server $ARGV[0]:$ARGV[1] $!\n" unless $socket;

$socket->send($commands);

# flush the buffer and close the socket gracefully
shutdown($socket, 1);

$socket->close();
```

* Usage

```bash
perl series-date-uploader.pl localhost 8081 data.csv
```

* Commands Sent

```ls
series e:axi-01 t:test_name=api-q-1 m:test_status=0 m:test_duration=32 d:2017-05-16T05:00:00Z
series e:axi-01 t:test_name=api-q-2 m:test_status=0 m:test_duration=2050 d:2017-05-16T05:00:00Z
series e:axi-01 t:test_name=api-q-4 m:test_status=0 m:test_duration=120 d:2017-05-16T05:00:00Z
series e:axi-01 t:test_name=api-q-1 m:test_status=0 m:test_duration=28 d:2017-05-16T06:00:00Z
series e:axi-01 t:test_name=api-q-2 m:test_status=0 m:test_duration=4000 d:2017-05-16T06:00:00Z
series e:axi-01 t:test_name=api-q-4 m:test_status=0 m:test_duration=201 d:2017-05-16T06:00:00Z
```

### Sending Data via [REST API](https://axibase.com/docs/atsd/api/data/)

This example illustrates how to read a CSV file, split the contents line by line, build a [series command](https://axibase.com/docs/atsd/api/network/series.html) and send it into the target ATSD server via `HTTPS` using the Data API [`command` method](https://axibase.com/docs/atsd/api/data/ext/command.html).

* Perl Script

```perl
#!/usr/bin/perl
use strict;
use warnings FATAL => 'all';
# make sure the module is installed
use DateTime::Format::Strptime;
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
my $date_parser = new DateTime::Format::Strptime(
        pattern => '%Y-%m-%d %H:%M:%S',
        time_zone => 'US/Pacific',
    );
my $line_index = 0;
my $commands = "";

while (my $line = <$fh>) {
    #skip header
    if ($line_index == 0) {
        $line_index++;
        next;
    }
    chomp $line;
    my @row = split($separator, $line);

    my $series_date =  $date_parser->parse_datetime($row[0]);
    $series_date->set_time_zone('GMT');

    $commands = $commands . sprintf("series e:%s t:test_name=%s m:test_status=%s m:test_duration=%s d:%s\n",
        $row[1], $row[2], $row[3], $row[4], $series_date->strftime("%Y-%m-%dT%H:%M:%SZ"));
    $line_index++;
}

print $commands;

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