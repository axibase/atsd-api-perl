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

```

* Usage

```bash

```

* Commands Sent

```ls

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

```

* Usage

```bash

```

* Commands Sent

```ls

```

### Sending Data using Data API

This example is similar to the initial example and illustrates how to read a CSV file, split its contents line by line, build a batch of [series](https://github.com/axibase/atsd/blob/master/api/network/series.md#series-command) commands and upload them into ATSD over HTTP(s) using the Data API [command](https://github.com/axibase/atsd/blob/master/api/data/ext/command.md) method.

* Perl Script

```perl

```

* Usage

```bash

```

