create function process_complimentary_courses(p_course_id integer, p_course_offer_id integer, p_user_id integer) returns void
    language plpgsql
as
$$
DECLARE
    complimentary_courses RECORD;
BEGIN
    -- Process each complimentary course for the user based on p_course_id and p_course_offer_id
    FOR complimentary_courses IN
        SELECT "addonCourseId", "addonCourseOfferId"
        FROM "ComplimentaryAddon"
        WHERE "courseId" = p_course_id
        AND (
            ("courseOfferId" = p_course_offer_id AND p_course_offer_id IS NOT NULL)
            OR ("courseOfferId" IS NULL AND p_course_offer_id IS NULL)
        )
    LOOP
        -- Check if the user already has access to the specific complimentary course being processed
        IF NOT EXISTS (
            SELECT 1
            FROM "UserCourse"
            WHERE "userId" = p_user_id
            AND "courseId" = complimentary_courses."addonCourseId"
            AND "expiryAt" > NOW()
            AND "expiryAt" - "startedAt" > INTERVAL '10 days'
        ) THEN
            -- Insert the specific complimentary course for the user if it does not already exist
            INSERT INTO "UserCourse" ("userId", "courseId", "startedAt", "expiryAt")
            VALUES (
                p_user_id,
                complimentary_courses."addonCourseId",
                NOW(),
                COALESCE(
                    CASE
                        WHEN complimentary_courses."addonCourseOfferId" IS NOT NULL THEN
                            (SELECT "expiryAt" FROM "CourseOffer" WHERE "id" = complimentary_courses."addonCourseOfferId")
                        ELSE
                            (SELECT "expiryAt" FROM "Course" WHERE "id" = complimentary_courses."addonCourseId")
                    END,
                    NOW() + INTERVAL '365 days'
                )
            );
        END IF;
    END LOOP;
END;
$$;

alter function process_complimentary_courses(integer, integer, integer) owner to neetprep_rw;
