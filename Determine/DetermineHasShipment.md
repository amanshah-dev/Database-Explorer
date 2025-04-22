create function "DetermineHasShipment"(paymentid integer) returns boolean
    language plpgsql
as
$$
DECLARE
    payment_record RECORD;
    has_shipment BOOLEAN := FALSE;
BEGIN
    -- Fetch the necessary fields from Payment table
    SELECT * INTO payment_record FROM "Payment" WHERE "id" = paymentId;

    IF payment_record."pincode" IS NOT NULL AND payment_record."address1" IS NOT NULL THEN

        -- Check for shipment in CourseOffer or Course directly
        SELECT COALESCE("hasShipment", FALSE) INTO has_shipment
        FROM (
            SELECT "hasShipment"
            FROM "CourseOffer"
            WHERE "id" = payment_record."courseOfferId"
            
            UNION ALL
            
            SELECT "hasShipment"
            FROM "Course"
            WHERE "id" = payment_record."paymentForId"
              AND payment_record."courseOfferId" IS NULL
        ) AS shipment_check
        WHERE "hasShipment" = TRUE
        LIMIT 1;

        IF has_shipment THEN
            RETURN TRUE; -- Step 1: Shipment found in CourseOffer or Course
        END IF;

        -- Check for shipment in CourseAddon
        SELECT EXISTS (
            SELECT 1
            FROM "CourseAddon"
            WHERE "courseId" = payment_record."paymentForId"
              AND ("courseOfferId" = payment_record."courseOfferId" OR (payment_record."courseOfferId" IS NULL AND "courseOfferId" IS NULL))
              AND (
                  ("addonCourseOfferId" IS NOT NULL AND 
                   EXISTS (SELECT 1 FROM "CourseOffer" WHERE "id" = "CourseAddon"."addonCourseOfferId" AND "hasShipment" = TRUE))
                  OR
                  ("addonCourseOfferId" IS NULL AND 
                   EXISTS (SELECT 1 FROM "Course" WHERE "id" = "CourseAddon"."addonCourseId" AND "hasShipment" = TRUE))
              )
        ) INTO has_shipment;

        IF has_shipment THEN
            RETURN TRUE; -- Step 2: Shipment found in CourseAddon
        END IF;

        -- Check for shipment in ComplimentaryAddon
        SELECT EXISTS (
            SELECT 1
            FROM "ComplimentaryAddon"
            WHERE "courseId" = payment_record."paymentForId"
              AND (
                  ("addonCourseOfferId" IS NOT NULL AND 
                   EXISTS (SELECT 1 FROM "CourseOffer" WHERE "id" = "ComplimentaryAddon"."addonCourseOfferId" AND "hasShipment" = TRUE))
                  OR
                  ("addonCourseOfferId" IS NULL AND 
                   EXISTS (SELECT 1 FROM "Course" WHERE "id" = "ComplimentaryAddon"."addonCourseId" AND "hasShipment" = TRUE))
              )
        ) INTO has_shipment;

        IF has_shipment THEN
            RETURN TRUE; -- Step 3: Shipment found in ComplimentaryAddon
        END IF;
    END IF;

    RETURN FALSE; -- Default return if no conditions are met
END;
$$;

alter function "DetermineHasShipment"(integer) owner to neetprep_rw;

