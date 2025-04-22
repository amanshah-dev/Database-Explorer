CREATE PROCEDURE insert_user_achievements(
    IN eligibleusers BIGINT[], 
    IN achievement_record RECORD
) 
LANGUAGE plpgsql
AS
$$
BEGIN
    -- Exit the loop if the achievement days exceed the user's streak
    IF array_length(eligibleUsers, 1) = 0 THEN
        RETURN;
    END IF;

    -- Upgrade all previous achievements if necessary
    UPDATE "UserAchievement"
    SET "upgraded" = TRUE
    WHERE "userId" = ANY (eligibleUsers)
      AND "achievementCategoryId" = achievement_record."achievementCategoryId"
      AND "achievementId" < achievement_record.id
      AND NOT "upgraded";

    WITH new_achievements AS (
        -- List of userIds and corresponding achievement information
        SELECT unnest(eligibleUsers) AS "userId"
    )
    INSERT INTO "UserAchievement" (
        "userId", 
        "achievementId", 
        "createdAt", 
        "updatedAt", 
        "upgraded", 
        "achievedAt", 
        "achievementCategoryId"
    )
    SELECT 
        na."userId",
        achievement_record.id,
        CURRENT_TIMESTAMP,
        CURRENT_TIMESTAMP,
        FALSE,
        CURRENT_TIMESTAMP,
        achievement_record."achievementCategoryId"
    FROM new_achievements na
    ON CONFLICT ("userId", "achievementId") DO NOTHING;
END;
$$;

ALTER PROCEDURE insert_user_achievements(BIGINT[], RECORD) OWNER TO neetprep_rw;
