create function process_cooldown_status(course_id integer, payment_window integer) returns void
    language plpgsql
as
$$
BEGIN
    INSERT INTO "Referral" ("userCouponId", "referredUserId", "status", "createdAt", "updatedAt", "referredUserPaymentId")
    SELECT 
        "UserCoupon"."id" AS user_coupon_id,       -- Use the primary key from UserCoupon as userCouponId
        "Payment"."userId" AS referred_id,        -- The referred user's ID
        'Cooldown' AS status,                     -- Status set to 'Cooldown'
        NOW() AS created_at,                      -- Current timestamp for createdAt
        NOW() AS updated_at,                      -- Current timestamp for updatedAt
        "Payment"."id" AS referred_payment_id     -- Store the payment ID as referredUserPaymentId
    FROM "Payment"
    JOIN "UserCoupon" 
        ON "Payment"."couponId" = "UserCoupon"."couponId"
    WHERE "Payment"."status" = 'responseReceivedSuccess' -- Filter for successful payments
      AND "Payment"."createdAt" >= NOW() - (payment_window || ' days')::INTERVAL -- Use the payment_window parameter
      AND "Payment"."paymentForId" = course_id              -- Match the course ID
      AND "UserCoupon"."userId" <> "Payment"."userId"       -- Exclude self-referrals
    ON CONFLICT ("userCouponId", "referredUserId") WHERE "status" != 'Refund' DO NOTHING;

END;
$$;

alter function process_cooldown_status(integer, integer) owner to neetprep_rw;
