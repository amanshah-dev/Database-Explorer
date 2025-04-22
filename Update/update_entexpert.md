create procedure update_entexpert(IN achievementcategoryid bigint, IN userids bigint[])
    language plpgsql
as
$$
BEGIN
    CALL update_subject_expert('ENT', achievementCategoryId, userIds);
END;
$$;

alter procedure update_entexpert(bigint, bigint[]) owner to learner;
