create function process_refund_status(cooldown_period integer) returns void
    language plpgsql
as
$$
BEGIN
    -- Update status from 'Cooldown' to 'Refund'
    UPDATE "Referral"
    SET "status" = 'Refund', 
        "updatedAt" = NOW()
    FROM "Payment"
    WHERE "Referral"."status" = 'Cooldown'
      AND "Referral"."createdAt" <= NOW() - INTERVAL '1 day' * cooldown_period
      AND "Referral"."referredUserPaymentId" = "Payment"."id"
      AND "Payment"."status" = 'responseReceivedRefund';
END;
$$;

alter function process_refund_status(integer) owner to neetprep_rw;
