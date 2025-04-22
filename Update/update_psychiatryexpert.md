create procedure update_psychiatryexpert(IN achievementcategoryid bigint, IN userids bigint[])
    language plpgsql
as
$$
BEGIN
    CALL update_subject_expert('Psychiatry', achievementCategoryId, userIds);
END;
$$;

alter procedure update_psychiatryexpert(bigint, bigint[]) owner to learner;
