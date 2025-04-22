create function process_completed_status(cooldown_period integer) returns void
    language plpgsql
as
$$
BEGIN
    -- Update the status from 'Cooldown' to 'Completed' for eligible referrals
    UPDATE "Referral"
    SET "status" = 'Completed',
        "updatedAt" = NOW()  -- Set the update timestamp
    FROM "Payment"
    WHERE "Referral"."status" = 'Cooldown'
      AND "Referral"."createdAt" <= NOW() - INTERVAL '1 day' * cooldown_period  -- Check if the referral is past the cooldown period
      AND "Referral"."referredUserPaymentId" = "Payment"."id"  -- Ensure the paymentId matches
      AND "Payment"."status" = 'responseReceivedSuccess';  -- Payment status must be 'responseReceivedSuccess'
END;
$$;

alter function process_completed_status(integer) owner to neetprep_rw;
