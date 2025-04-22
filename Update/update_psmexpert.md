create procedure update_psmexpert(IN achievementcategoryid bigint, IN userids bigint[])
    language plpgsql
as
$$
BEGIN
    CALL update_subject_expert('PSM', achievementCategoryId, userIds);
END;
$$;

alter procedure update_psmexpert(bigint, bigint[]) owner to learner;
