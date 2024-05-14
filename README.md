# sts-twilio

Requires a TwiML Bin like the following:

```
<?xml version="1.0" encoding="UTF-8"?>
<Response>
  <Say voice="woman" language="en">"This call may be recorded."</Say>
  <Connect>
    <Stream url="wss://a127-75-172-116-97.ngrok-free.app/twilio" />
  </Connect>
</Response>
```
