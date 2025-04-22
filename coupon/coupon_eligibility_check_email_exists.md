# SQL Function: coupon_eligibility_check_email_exists

```sql
create function coupon_eligibility_check_email_exists(input_params jsonb) returns boolean
    language plpgsql
as
$$
BEGIN
    -- Check if the email exists for the user
    RETURN EXISTS (
        SELECT 1
        FROM "User"
        WHERE "id" = (input_params->>'user_id')::INT
        AND "email" LIKE '%goodeducator.com'
    );
END;
$$;

alter function coupon_eligibility_check_email_exists(jsonb) owner to learner;
