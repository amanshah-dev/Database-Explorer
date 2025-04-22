create procedure update_obsgynaeexpert(IN achievementcategoryid bigint, IN userids bigint[])
    language plpgsql
as
$$
BEGIN
    CALL update_subject_expert('Obs - Gynae', achievementCategoryId, userIds);
END;
$$;

alter procedure update_obsgynaeexpert(bigint, bigint[]) owner to learner;
