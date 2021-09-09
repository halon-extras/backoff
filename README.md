# Enable backoff mode

A HSL function for enabling backoff mode on sub-queues using dynamic queue policies.

## smtpd-app.yaml

```
lists:
  remotemx:
    gsuite:
      - '*.google.com'
      - '*.googlemail.com'
      - '*.smtp.goog'
```

## smtpd-policy.yaml

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
    default:
      concurrency: 5
      rate: 10
      properties:
        backoff-concurrency: 1
        backoff-rate: 5/3600
        backoff-ttl: 3600
```

## Post-delivery script

```
enable_backoff($arguments, $message); // Enable all matching backoff policies
enable_backoff($arguments, $message, ["remotemx"]); // Enable only one backoff policy based on it's fields
```