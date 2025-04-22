CREATE FUNCTION "DetermineActualCourseId"(paymentid INTEGER) 
    RETURNS INTEGER
    LANGUAGE plpgsql
AS
$$
DECLARE
    actual_course_id INTEGER;
BEGIN
    -- Directly use the NEW record's courseOfferId to determine actualCourseId
    IF NEW."courseOfferId" IS NULL THEN
        -- If courseOfferId is NULL, return paymentForId as fallback
        actual_course_id := NEW."paymentForId";
    ELSE
        -- If courseOfferId is present, fetch actualCourseId from the CourseOffer table
        SELECT "actualCourseId"
        INTO actual_course_id
        FROM "CourseOffer"
        WHERE "id" = NEW."courseOfferId";

        IF actual_course_id IS NULL THEN
            -- If no actualCourseId found, return paymentForId as fallback
            actual_course_id := NEW."paymentForId";
        END IF;
    END IF;

    -- Return the determined actualCourseId
    RETURN actual_course_id;
END;
$$;

ALTER FUNCTION "DetermineActualCourseId"(INTEGER) OWNER TO learner;
