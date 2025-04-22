create procedure update_dppdynamo(IN achievementcategoryid bigint, IN userids bigint[] DEFAULT ARRAY[]::bigint[])
    language plpgsql
as
$$
DECLARE
    achievement_record RECORD;
    eligibleUsers      BIGINT[] := ARRAY []::BIGINT[];
BEGIN
    FOR achievement_record IN
        /* Query to get user answers and match them with correct options in the Question table */
        SELECT "id", "currentLevelUnit" as current_level, "achievementCategoryId"
        FROM "Achievement"
        WHERE "achievementCategoryId" = achievementCategoryId
        order by current_level
        LOOP
            eligibleUsers := dppStreakAchievers(userIds, achievement_record.current_level, 'practice'::dpp_type_enum);
            call insert_user_achievements(eligibleUsers, achievement_record);
        END LOOP;
END;
$$;

alter procedure update_dppdynamo(bigint, bigint[]) owner to neetprep_rw;
