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

### smtpd-app.yaml

```
lists:
  remotemx:
    gsuite:
      - '*.google.com'
      - '*.googlemail.com'
      - '*.smtp.goog'
```

### smtpd-policy.yaml

```
policies:
  - fields:
      - remotemx:
          gsuite:
            - "@gsuite"
    conditions:
      - if:
          remotemx: "#gsuite"
        then:
          concurrency: 10
          rate: 50
          properties:
            backoff-concurrency: 2
            backoff-rate: 10/3600
            backoff-ttl: 3600
            backoff-disableable: true
    default:
      concurrency: 5
      rate: 10
      properties:
        backoff-concurrency: 1
        backoff-rate: 5/3600
        backoff-ttl: 3600
        backoff-disableable: false
```

## Exported functions

These functions needs to be [imported](https://docs.halon.io/hsl/structures.html#import) from the `extras://backoff` module path.

### enable_backoff(arguments, message, patterns [, fields])

**Params**

- arguments `array` - The [$arguments](https://docs.halon.io/hsl/postdelivery.html#v-z1) variable
- message `array` - The [$message](https://docs.halon.io/hsl/postdelivery.html#v-m1) variable
- patterns `array` - The patterns
- fields `array` - The fields (optional)

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
  $patterns = [
    ["pattern" => #/421 4\.3\.2 No system resources/, "tag" => "system"], // Use SMTP patterns to determine if backoff should be enabled
    // #/421 4\.3\.2 No system resources/ // The pattern can also be a plain regex or string without the tag property
  ];
  enable_backoff($arguments, $message, $patterns); // Enable all matching backoff policies
  enable_backoff($arguments, $message, $patterns, ["remotemx"]); // Enable only one backoff policy based on it's fields
} else {
  // Successful deliveries
  disable_backoff($arguments, $message); // Disable all matching backoff policies
  disable_backoff($arguments, $message, ["remotemx"]); // Disable only one backoff policy based on it's fields
}
```