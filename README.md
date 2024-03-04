# Backoff mode

HSL functions to enable/disable backoff mode on sub-queues using dynamic queue policies.

## Installation

Follow the [instructions](https://docs.halon.io/manual/comp_install.html#installation) in our manual to add our package repository and then run the below command.

### Ubuntu

```
apt-get install halon-extras-backoff
```

### RHEL

```
yum install halon-extras-backoff
```

## Configuration

### smtpd.yaml

```
plugins:
  - id: bounce-list
    config:
      lists:
        - id: backoff
          path: /etc/halon/backoff.csv
```

### smtpd-app.yaml

```
queues:
  grouping:
    default: recipientdomain
    groupings:
      - id: google
        remotemx:
          - "*.google.com"
          - "*.googlemail.com"
          - "*.smtp.goog"
      - id: microsoft
        remotemx:
          - "*.protection.outlook.com"
      - id: yahoo
        remotemx:
          - "*.yahoodns.net"
```

### smtpd-policy.yaml

```
policies:
  - fields:
      - localip
      - grouping
    conditions:
      - if:
          grouping: "&google"
        then:
          concurrency: 4
          rate: 120/60
          properties:
            backoff-concurrency: 2
            backoff-rate: 10/3600
            backoff-ttl: 3600
            backoff-disableable: true
            backoff-suspendable: true
            # backoff-retry-intervals: 60,900,3600,7200,10800
            # backoff-retry-count: 30
    default:
      concurrency: 2
      rate: 30/60
      properties:
        backoff-concurrency: 1
        backoff-rate: 5/3600
        backoff-ttl: 3600
        backoff-disableable: false
        backoff-suspendable: false
        # backoff-retry-intervals: 60,900,3600,7200,10800
        # backoff-retry-count: 30
```

**Properties**

- backoff-concurrency `number` - The concurrency that should be applied when entering backoff mode for the queue
- backoff-rate `string` - The rate that should be applied when entering backoff mode for the queue
- backoff-ttl `number` - How long the backoff should be enabled for the queue
- backoff-disableable `boolean` - If the backoff should be disabled upon a successful delivery for the queue
- backoff-suspendable `boolean` - If the queue can be completely suspended by a specific backoff pattern
- backoff-retry-intervals `string` - The retry intervals that should be used when entering backoff mode for the queue
- backoff-retry-count `number` - The retry count that should be used when entering backoff mode for the queue
- backoff-requeue `boolean` - If the messages in the queue should be requeued instead of entering backoff mode
- backoff-requeue-* `string` - Custom properties to return when requeueing the messages which can be accessed using `$arguments["queue"]["plugin"]["return"]` in the [Pre-delivery](https://docs.halon.io/hsl/predelivery.html) script

### backoff.csv

```
"/^450/","tag=tag1",&google,EOD
"/^452/","tag=tag2,events=10/60"
"/^451/","tag=tag3,suspend=3600"
"/^5\d\d/","tag=tag4"
```

The format is described in [`halon-extras`](https://github.com/halon-extras/bounce-list?tab=readme-ov-file#bounce-list-format), this describes how to use the Grouping and SMTP state columns to limit the scope of a rule.

**Options, specified in the second column**

- tag `string` - The tag that should be applied for the dynamic policy / suspend. The max length is `8` in version `5.12` and below of the Halon MTA and `24` in version `6.0` and above
- events `string` - The rate that is needed for the backoff to trigger. If added you need to use a value above 1 event per interval. The default is to trigger immediately
- suspend `number` - Will suspend the queue completely for that many seconds. The default is to not suspend the queue

## Exported functions

These functions needs to be [imported](https://docs.halon.io/hsl/structures.html#import) from the `extras://backoff` module path.

### enable_backoff(arguments, message, list [, fields, [, rate]])

**Params**

- arguments `array` - The [$arguments](https://docs.halon.io/hsl/postdelivery.html#v-z1) variable
- message `array` - The [$message](https://docs.halon.io/hsl/postdelivery.html#v-m1) variable
- list `string` - The [bounce list](https://github.com/halon-extras/bounce-list) ID
- fields `array` - The fields (optional)
- rate `function` - The rate function

**Returns**

Returns an `array` with an optional index of `pattern` (`string`) if there was a matching pattern, an optional index of `delay` (`number`) if the message should be queued with a delay instead of bounced and an optional index of `bounce` (`boolean`) if the message should be bounced instead of queued.

### disable_backoff(arguments, message, [, fields])

**Params**

- arguments `array` - The [$arguments](https://docs.halon.io/hsl/postdelivery.html#v-z1) variable
- message `array` - The [$message](https://docs.halon.io/hsl/postdelivery.html#v-m1) variable
- fields `array` - The fields (optional)

**Example (Post-delivery)**

```
import { enable_backoff, disable_backoff } from "extras://backoff";

if ($arguments["action"]) {
  // Failed deliveries
  $backoff = enable_backoff($arguments, $message, "backoff"); // Enable all matching backoff policies
  // $backoff = enable_backoff($arguments, $message, "backoff", ["localip", "grouping"]); // Enable only one backoff policy based on it's fields
  if ($backoff["delay"]) {
    Queue(["delay" => $backoff["delay"]]);
  }
  if ($backoff["bounce"]) {
    Bounce();
  }
} else {
  // Successful deliveries
  disable_backoff($arguments, $message); // Disable all matching backoff policies
  // disable_backoff($arguments, $message, ["localip", "grouping"]); // Disable only one backoff policy based on it's fields
}
```