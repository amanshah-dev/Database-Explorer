create function updateuserachievement(userids bigint[] DEFAULT ARRAY[]::bigint[]) returns void
    language plpgsql
as
$$
DECLARE
    achievementCategory RECORD;
    procedureName       TEXT;
    varSQLQuery         TEXT;
BEGIN
    IF array_length(userIds, 1) = 0 THEN
--         Return early if no userIds
        return;
    end if;

    for achievementCategory in select * from "AchievementCategory"
        LOOP
            procedureName := 'update_' || replace(replace(lower(achievementCategory.name), ' ', ''), '-', '');
            varSQLQuery := format('CALL %s($1, $2);', procedureName);
            RAISE NOTICE 'varSQLQuery %', varSQLQuery;
            EXECUTE varSQLQuery USING achievementCategory.id, userIds;
        end loop;

end
$$;

alter function updateuserachievement(bigint[]) owner to neetprep_rw;
