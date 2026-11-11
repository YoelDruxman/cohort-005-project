# Approving a Refund Request and sending the money are separate steps

Cadence has no payment-provider integration, so an Admin pays out every Refund by hand outside the app. Because of that, approving a Refund Request creates a Pending Refund, and an Admin later marks it Issued or Failed once they know whether the money went through. We expect to drop the Pending step once payouts are automated, at which point approval will issue the Refund directly.

## Consequences

- The learner loses access to the course when the request is approved, not when the money is sent. The ruling is final, and the learner shouldn't keep watching while the Refund is Pending.
- A Failed Refund isn't a final state. An Admin can still mark it Issued once the payout succeeds.
- The learner gets an email on approval and another when the Refund is Issued. Whether both emails survive automation depends on how quickly payouts complete.
