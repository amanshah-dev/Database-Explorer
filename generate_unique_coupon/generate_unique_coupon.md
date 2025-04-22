CREATE FUNCTION generate_unique_coupon() RETURNS TEXT
    LANGUAGE plpgsql
AS
$$
DECLARE
    chars TEXT := 'ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789'; -- Alphanumeric characters
    coupon TEXT := '';
    i INT;
BEGIN
    LOOP
        -- Generate a new 6-character coupon
        coupon := '';
        FOR i IN 1..6 LOOP
            coupon := coupon || substr(chars, floor(random() * length(chars) + 1)::int, 1);
        END LOOP;

        -- Check if the coupon already exists in the "Coupon" table
        IF NOT EXISTS (
            SELECT 1 
            FROM "Coupon" 
            WHERE "code" = coupon
        ) THEN
            RETURN coupon; -- If the coupon is unique in the Coupon table, return it
        END IF;
    END LOOP;
END;
$$;

ALTER FUNCTION generate_unique_coupon() OWNER TO neetprep_rw;
