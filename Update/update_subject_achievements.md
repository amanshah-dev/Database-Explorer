create procedure update_subject_achievements(IN userids bigint[], IN subject text, IN achievementcategoryid bigint)
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
            eligibleUsers := getStreakers(userIds, achievement_record.current_level, subject);
            --             select array_agg(userId)
--             into eligibleUsers
--             from totalTable
--             where correctAnswers >= achievement_record.current_level;

            call insert_user_achievements(eligibleUsers, achievement_record);
        end loop;
END;
$$;

alter procedure update_subject_achievements(bigint[], text, bigint) owner to neetprep_rw;
