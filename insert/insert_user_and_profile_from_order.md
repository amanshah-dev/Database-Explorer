CREATE FUNCTION insert_user_and_profile_from_order() RETURNS TRIGGER
LANGUAGE plpgsql
AS
$$
DECLARE
    v_user_id INTEGER;
    v_centre_id INTEGER;
    v_course_id INTEGER;
    v_course_offer_id INTEGER;
    course_rec RECORD; -- Record to iterate over multiple courses
BEGIN
    -- Check if the product_id exists in "ShopifyProductCourse"
    IF EXISTS (
        SELECT 1
        FROM "ShopifyProductCourse"
        WHERE "productId" = NEW."product_id"
    ) THEN

        -- Insert or update the User table
        INSERT INTO public."User" (
            "email", "phone", "source", "createdAt", "updatedAt"
        )
        VALUES (
            NEW."email", NEW."phone", 'shopify', NOW(), NOW()
        )
        ON CONFLICT ("email") DO UPDATE
            SET "updatedAt" = NOW()
        RETURNING "id" INTO v_user_id;

        -- If not returned in this insert, fetch existing user ID
        IF v_user_id IS NULL THEN
            SELECT "id" INTO v_user_id
            FROM public."User"
            WHERE "email" = NEW."email";
        END IF;

        IF v_user_id IS NOT NULL THEN
            -- Insert or update the UserProfile
            INSERT INTO "UserProfile" (
                "userId", "email", "phone", "createdAt", "updatedAt"
            )
            VALUES (
                v_user_id, NEW."email", NEW."phone", NOW(), NOW()
            )
            ON CONFLICT ("userId") DO UPDATE
                SET "email" = EXCLUDED."email",
                    "phone" = EXCLUDED."phone",
                    "updatedAt" = NOW();

            -- Loop through all courses mapped to the productId
            FOR course_rec IN (
                SELECT "centreId", "courseId", "courseOfferId"
                FROM "ShopifyProductCourse"
                WHERE "productId" = NEW."product_id"
            )
            LOOP
                v_centre_id := course_rec."centreId";
                v_course_id := course_rec."courseId";
                v_course_offer_id := course_rec."courseOfferId";

                -- Insert or update UserCourse for each course
                INSERT INTO "UserCourse" (
                    "userId", "courseId", "courseOfferId", "role", "startedAt", "expiryAt", "createdAt", "updatedAt"
                )
                VALUES (
                    v_user_id,
                    v_course_id,
                    v_course_offer_id,
                    'courseStudent',
                    NOW(),
                    CASE
                        WHEN v_course_offer_id IS NOT NULL THEN
                            (SELECT "expiryAt" FROM "CourseOffer" WHERE "CourseOffer"."id" = v_course_offer_id)
                        ELSE
                            (SELECT "expiryAt" FROM "Course" WHERE "Course"."id" = v_course_id)
                    END,
                    NOW(),
                    NOW()
                )
                ON CONFLICT ("userId", "courseId", "expiryAt") DO UPDATE
                    SET "updatedAt" = NOW(),
                        "courseOfferId" = EXCLUDED."courseOfferId";

                -- Insert into UserCentre if centreId is found
                IF v_centre_id IS NOT NULL THEN
                    INSERT INTO "UserCentre" (
                        "userId", "centreId", "createdAt", "updatedAt"
                    )
                    VALUES (
                        v_user_id, v_centre_id, NOW(), NOW()
                    )
                    ON CONFLICT DO NOTHING; -- Avoid duplicate UserCentre entries
                END IF;
            END LOOP;
        END IF;
    END IF;

    RETURN NEW;
END;
$$;

ALTER FUNCTION insert_user_and_profile_from_order() OWNER TO learner;
