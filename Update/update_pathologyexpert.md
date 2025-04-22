create procedure update_pathologyexpert(IN achievementcategoryid bigint, IN userids bigint[])
    language plpgsql
as
$$
BEGIN
    CALL update_subject_expert('Pathology', achievementCategoryId, userIds); -- Call the main procedure with 'Pathology'
END;
$$;

alter procedure update_pathologyexpert(bigint, bigint[]) owner to learner;
