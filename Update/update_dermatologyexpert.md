create procedure update_dermatologyexpert(IN achievementcategoryid bigint, IN userids bigint[])
    language plpgsql
as
$$
BEGIN
    CALL update_subject_expert('Dermatology', achievementCategoryId, userIds);
END;
$$;

alter procedure update_dermatologyexpert(bigint, bigint[]) owner to learner;
