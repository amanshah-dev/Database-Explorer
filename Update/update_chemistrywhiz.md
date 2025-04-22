
create procedure update_chemistrywhiz(IN achievementcategoryid bigint, IN userids bigint[])
    language plpgsql
as
$$
BEGIN
    CALL update_subject_achievements(userids, 'Chemistry', achievementcategoryid); -- Call the main procedure with 'Chemistry'
END;
$$;

alter procedure update_chemistrywhiz(bigint, bigint[]) owner to neetprep_rw;
