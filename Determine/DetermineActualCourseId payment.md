CREATE FUNCTION "DetermineActualCourseId"(paymentforid INTEGER, courseofferid INTEGER)
    RETURNS INTEGER
    LANGUAGE plpgsql
AS
$$
DECLARE
    actual_course_id INTEGER;
BEGIN
    -- If courseOfferId is NULL, return paymentForId as fallback
    IF courseofferid IS NULL THEN
        actual_course_id := paymentforid;
    ELSE
        -- If courseOfferId is present, fetch actualCourseId from the CourseOffer table
        SELECT "actualCourseId"
        INTO actual_course_id
        FROM "CourseOffer"
        WHERE "id" = courseofferid;
        
        IF actual_course_id IS NULL THEN
            -- If no actualCourseId found, return paymentForId as fallback
            actual_course_id := paymentforid;
        END IF;
    END IF;

    -- Return the determined actualCourseId
    RETURN actual_course_id;
END;
$$;

ALTER FUNCTION "DetermineActualCourseId"(INTEGER, INTEGER) OWNER TO learner;
