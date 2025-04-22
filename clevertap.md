# CleverTap Integration

Updated on 2025-02-13

## Events
1. **Page View:** Associated data can be the URL, search params.
2. **Button Clicked:** Associated data can be the name of the btn, state of the app/user/page/btn. Related, if available, 
   course ID, course offer ID.
3. **Paywall Encountered:** Associated data can be course info.
4. **Payment Result:** From backend, when the PayTM payment result has been updated. Associated data can be payment ID, 
   paymentStatus (`success` or `failure`), course info, course offer, coupon.
5. **Payment Intent:** On `generate_txn_token` API call. Associated data can be course info, course offer, coupon etc.
6. **User Registered:** From cron, that sends the data from `User` table, and the `ts` of the event must match the 
   `createdAt` field of the User object. If `createdAt` of the user is older than 2 years old NEET, the date will be 
   set to one-day before the 2 years old NEET.
7. **User Not Registered:** From cron, that sends the data from `User` table, and the `ts` of the event will be
   set to one-day before the 2 years old NEET. This event is required to make the "User Registered" event work.
8. **Payment Created:** From cron that sends all payments with status "created" in an hour.
9. **Payment Success:** From cron that sends all payments with status "responseReceivedSuccess" in an hour.
10. **Payment Failure:** From cron that sends all payments with status "responseReceivedFailure" in an hour.

## User Properties
### Identity
1. Email
2. ID (as in the User table)

### Associated data
1. isPaid: Boolean. `true`, if user has paid even once for any course.
2. User courses
3. Phone number
4. Address (Location, IP)
5. Number of questions practiced till now
6. Other fields from User Profile

## Changelog

* 2025-02-13: Added "Payment Created", "Payment Success", "Payment Failure" event
* 2025-02-07: Added "User Not Registered" event
* 2025-01-29: Added "User Registered" event
* 2025-01-20: Initial version
