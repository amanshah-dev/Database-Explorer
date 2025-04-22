create function process_expiry_status(expiry_period integer) returns void
    language plpgsql
as
$$
BEGIN
    -- Step 1: Expire referrals for users with less than 3 referrals
    UPDATE "Referral"
    SET "status" = 'Expired',
        "updatedAt" = NOW()
    WHERE "userCouponId" IN (
        SELECT r."userCouponId"
        FROM "Referral" r
        WHERE r."status" IN ('Cooldown', 'Completed')
          AND r."createdAt" <= NOW() - INTERVAL '1 day' * expiry_period
        GROUP BY r."userCouponId"
        HAVING COUNT(r."id") < 3
    );

    -- Step 2: Do nothing for users with exactly 3 referrals
    -- No query required as this condition requires no updates.

    -- Step 3: Expire the 4th referral for users with exactly 4 referrals
    UPDATE "Referral"
    SET "status" = 'Expired',
        "updatedAt" = NOW()
    WHERE "id" IN (
        SELECT r."id"
        FROM "Referral" r
        WHERE r."status" IN ('Cooldown', 'Completed')
          AND r."createdAt" <= NOW() - INTERVAL '1 day' * expiry_period
          AND r."userCouponId" IN (
              SELECT r2."userCouponId"
              FROM "Referral" r2
              WHERE r2."status" IN ('Cooldown', 'Completed')
                AND r2."createdAt" <= NOW() - INTERVAL '1 day' * expiry_period
              GROUP BY r2."userCouponId"
              HAVING COUNT(r2."id") = 4
          )
        ORDER BY r."createdAt" DESC
        LIMIT 1
    );  

    -- Step 4: Do nothing for users with exactly 5 referrals
    -- No query required as this condition requires no updates.

    -- Step 5: Expire all referrals beyond the first 5 for users with more than 5 referrals
    UPDATE "Referral"
    SET "status" = 'Expired',
        "updatedAt" = NOW()
    WHERE "id" IN (
        SELECT subquery."id"
        FROM (
            SELECT r."id",
                   r."userCouponId", -- Include userCouponId for use in outer query
                   ROW_NUMBER() OVER (PARTITION BY r."userCouponId" ORDER BY r."createdAt") AS referral_rank
            FROM "Referral" r
            WHERE r."status" IN ('Cooldown', 'Completed')
              AND r."createdAt" <= NOW() - INTERVAL '1 day' * expiry_period
        ) subquery
        WHERE subquery.referral_rank > 5 -- Target referrals beyond the first 5
          AND subquery."userCouponId" IN (
              SELECT r2."userCouponId"
              FROM "Referral" r2
              WHERE r2."status" IN ('Cooldown', 'Completed')
              AND r2."createdAt" <= NOW() - INTERVAL '1 day' * expiry_period
              GROUP BY r2."userCouponId"
              HAVING COUNT(r2."id") > 5 -- Target users with more than 5 referrals
          )
    );

END;
$$;

alter function process_expiry_status(integer) owner to neetprep_rw;
