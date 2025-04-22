create procedure update_medicineexpert(IN achievementcategoryid bigint, IN userids bigint[])
    language plpgsql
as
$$
BEGIN
    CALL update_subject_expert('Medicine', achievementCategoryId, userIds);
END;
$$;

alter procedure update_medicineexpert(bigint, bigint[]) owner to learner;
