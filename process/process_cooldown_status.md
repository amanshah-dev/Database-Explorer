create function process_cooldown_status(course_id integer) returns void
    language plpgsql
as
$$
BEGIN
    INSERT INTO "Referral" ("userId", "referredUserId", "status", "createdAt", "updatedAt", "paymentId")
    SELECT 
        "UserCoupon"."userId" AS referrer_id,
        "Payment"."userId" AS referred_id,
        'Cooldown' AS status,
        NOW() AS created_at,
        NOW() AS updated_at,
        "Payment"."id" AS payment_id  -- Store the paymentId in the Referral table
    FROM "Payment"
    JOIN "UserCoupon" 
        ON "Payment"."couponId" = "UserCoupon"."couponId"
    WHERE "Payment"."status" = 'responseReceivedSuccess'
      AND "Payment"."createdAt" >= NOW() - INTERVAL '10 days'
      AND "Payment"."courseOfferId" = course_id
      AND "UserCoupon"."userId" <> "Payment"."userId"
    ON CONFLICT ("userId", "referredUserId") DO NOTHING;
END;
$$;

alter function process_cooldown_status(integer) owner to neetprep_rw;
