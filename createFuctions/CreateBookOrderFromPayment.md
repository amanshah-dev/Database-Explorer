CREATE FUNCTION "CreateBookOrderFromPayment"() RETURNS VOID
    LANGUAGE plpgsql
AS
$$
BEGIN
    INSERT INTO "BookOrder" ("orderId", "courseId")
    SELECT "id" || '-' || floor(extract(epoch from "createdAt"))::integer, 1904
    FROM "Payment"
    WHERE "status" = 'responseReceivedSuccess' 
    AND "paymentForId" = 1970
    AND "createdAt" > (current_timestamp - interval '1 day')
    ON CONFLICT("orderId", "courseId") DO NOTHING;

    INSERT INTO "BookOrder" ("orderId", "courseId")
    SELECT "id" || '-' || floor(extract(epoch from "createdAt"))::integer, 1906
    FROM "Payment"
    WHERE "status" = 'responseReceivedSuccess' 
    AND "paymentForId" = 1970
    AND "createdAt" > (current_timestamp - interval '1 day')
    ON CONFLICT("orderId", "courseId") DO NOTHING;

    INSERT INTO "BookOrder" ("orderId", "courseId")
    SELECT "id" || '-' || floor(extract(epoch from "createdAt"))::integer, 1905
    FROM "Payment"
    WHERE "status" = 'responseReceivedSuccess' 
    AND "paymentForId" = 2333
    AND "createdAt" > (current_timestamp - interval '1 day')
    ON CONFLICT("orderId", "courseId") DO NOTHING;

    INSERT INTO "BookOrder" ("orderId", "courseId")
    SELECT "id" || '-' || floor(extract(epoch from "createdAt"))::integer, 1907
    FROM "Payment"
    WHERE "status" = 'responseReceivedSuccess' 
    AND "paymentForId" = 2333
    AND "createdAt" > (current_timestamp - interval '1 day')
    ON CONFLICT("orderId", "courseId") DO NOTHING;

    -- Additional inserts for other conditions, following the same pattern...
    -- (similar insertions for different paymentForIds and courseIds)

    -- Added only the line below to handle new masterclass companion book for class 11th
    INSERT INTO "BookOrder" ("orderId", "courseId")
    SELECT "id" || '-' || floor(extract(epoch from "createdAt"))::integer, 4313
    FROM "Payment"
    WHERE "status" = 'responseReceivedSuccess' 
    AND "paymentForId" = 4313
    AND "createdAt" > (current_timestamp - interval '1 day')
    ON CONFLICT("orderId", "courseId") DO NOTHING;

    INSERT INTO "BookOrder" ("orderId", "courseId")
    SELECT "id" || '-' || floor(extract(epoch from "createdAt"))::integer, 4314
    FROM "Payment"
    WHERE "status" = 'responseReceivedSuccess' 
    AND "paymentForId" = 4314
    AND "createdAt" > (current_timestamp - interval '1 day')
    ON CONFLICT("orderId", "courseId") DO NOTHING;
END;
$$;

ALTER FUNCTION "CreateBookOrderFromPayment"() OWNER TO learner;
