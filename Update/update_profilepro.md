create procedure update_profilepro(IN achievementcategoryid bigint, IN userids bigint[] DEFAULT ARRAY[]::bigint[])
    language plpgsql
as
$$
DECLARE
    -- Define the achievement levels for Profile Pro
    achievement_record RECORD;
    eligibleUsers      INT[];
BEGIN
    -- Loop through each achievement level to check and update
    FOR achievement_record IN
        /* Query to get user answers and match them with correct options in the Question table */
        SELECT "id", "currentLevelUnit" as current_level, "achievementCategoryId"
        FROM "Achievement"
        WHERE "achievementCategoryId" = achievementCategoryId
        order by current_level
        LOOP

            eligibleUsers := calculate_profile_pros(userIds, achievement_record.current_level);
            -- Check if the user has achieved the current profile completion level
            call insert_user_achievements(eligibleUsers, achievement_record);
        END LOOP;
END;
$$;

alter procedure update_profilepro(bigint, bigint[]) owner to neetprep_rw;
