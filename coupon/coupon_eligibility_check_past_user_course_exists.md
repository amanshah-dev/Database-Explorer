# SQL Function: coupon_eligibility_check_past_user_course_exists

```sql
create function coupon_eligibility_check_past_user_course_exists(input_params jsonb) returns boolean
    language plpgsql
as
$$
BEGIN
    RETURN EXISTS (
        SELECT 1
        FROM "UserCourse"
        WHERE "userId" = (input_params->>'user_id')::INTEGER
        AND "courseId" = (input_params->>'course_id')::INTEGER
        AND "expiryAt" < now()
        AND "expiryAt" - "startedAt" > INTERVAL '10 days'
    );
END;
$$;

alter function coupon_eligibility_check_past_user_course_exists(jsonb) owner to learner;
