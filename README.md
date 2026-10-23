## ClassCard Security Research
On 2026-02-24, while analyzing ClassCard's account recovery functionality, I discovered the following two vulnerabilities.

1. Insufficient email ownership verification in the username recovery feature
2. Predictable password reset tokens

By chaining these two vulnerabilities, it was possible to obtain another user's username and predict the password reset token, resulting in account takeover.

## Vulnerability Details
### 1. Insufficient Email Ownership Verification in Username Recovery
The username recovery feature did not sufficiently verify the ownership relationship between `user_idx` and the provided email address.

**Request**:
```
POST https://www.classcard.net/LoginProc/sendIDToContact HTTP/1.1
Content-Type: application/x-www-form-urlencoded; charset=UTF-8
sel_user_idx=[index_number]&str_contact=[email]&find_by=email
```

**Response**:
```
HTTP/1.1 200 OK
Content-Type: text/html; charset=UTF-8

{"result":"ok","msg":"","sendID":4}
```

When the target user's value was provided in `sel_user_idx` and an email address accessible to the attacker was provided in `str_contact`, the target user's username was sent to that email address.

The provided email address was used as the delivery destination without verifying whether it was registered to the target account.

### 2. Predictable Password Reset Tokens
Analysis of the password reset tokens showed that sequentially increasing numbers were encoded using Base64 instead of using cryptographically secure random values.

For example, the following tokens were observed.
```
OTQ0OTQ0 > 944944
OTQ0OTQ1 > 944945
```
The underlying values increased sequentially.
```
944944
944945
944946
...
```
The sequence was not maintained independently for each account. Instead, a single global sequence was shared across password reset requests for the entire service.
An attacker could determine the current sequence value simply by requesting a password reset token for their own account.
After requesting a password reset for the target account, the attacker could predict the generated reset token based on the next sequence value.

## Account Takeover
The two vulnerabilities could be chained as follows.

1. Use the username recovery issue to receive the target user's username at an email address accessible to the attacker.
2. Request a password reset for the attacker's own account.
3. Determine the current global sequence value from the issued token.
4. Request a password reset for the target user's username.
5. Calculate the next sequence value and predict the reset token generated for the target account.
6. Use the predicted token during the password reset process.

I confirmed that the predicted token could be used to change the target user's password without knowing the existing password.

This ultimately resulted in unauthorized access to the target account.

## Disclosure
The vulnerabilities were reported to the service operator on 2026-02-24 and were fixed on 2026-03-11.
The vulnerabilities are currently patched.
This document is published for portfolio purposes with permission from the service operator.
