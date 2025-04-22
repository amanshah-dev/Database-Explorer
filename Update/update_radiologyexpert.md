create procedure update_radiologyexpert(IN achievementcategoryid bigint, IN userids bigint[])
    language plpgsql
as
$$
BEGIN
    CALL update_subject_expert('Radiology', achievementCategoryId, userIds);
END;
$$;

alter procedure update_radiologyexpert(bigint, bigint[]) owner to learner;
