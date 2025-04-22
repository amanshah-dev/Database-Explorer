create procedure update_biologyexpert(IN achievementcategoryid bigint, IN userids bigint[])
    language plpgsql
as
$$
BEGIN
    CALL update_subject_achievements(userids, 'Biology', achievementcategoryid); -- Call the main procedure with 'Biology'
END;
$$;

alter procedure update_biologyexpert(bigint, bigint[]) owner to neetprep_rw;
